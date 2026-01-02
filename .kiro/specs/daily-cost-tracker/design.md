# Design Document: Daily Cost Tracker

## Overview

The Daily Cost Tracker is a serverless AWS solution that automatically monitors daily AWS spending and sends email notifications. The system uses EventBridge for scheduling, Lambda for processing, Cost Explorer API for cost data retrieval, and SES for email delivery. The architecture follows AWS best practices for serverless applications with proper error handling, logging, and configuration management.

## Architecture

```mermaid
graph TB
    EB[EventBridge Rule] --> LF[Lambda Function]
    LF --> CE[Cost Explorer API]
    LF --> SES[Simple Email Service]
    LF --> CW[CloudWatch Logs]
    
    subgraph "Lambda Function"
        CM[Cost Manager]
        EM[Email Manager]
        CFG[Config Manager]
    end
    
    EB -.->|Daily Trigger<br/>cron(0 8 * * ? *)| LF
    CE -.->|Cost Data| CM
    CM -.->|Formatted Report| EM
    EM -.->|Email| SES
    LF -.->|Logs| CW
```

The system operates on a daily schedule, triggered by EventBridge at 8:00 AM UTC. The Lambda function retrieves cost data from the previous day, formats it into a readable report, and sends it via email to configured recipients.

## Components and Interfaces

### EventBridge Scheduler
- **Purpose**: Triggers the Lambda function daily at a specified time
- **Configuration**: Uses cron expression `cron(0 8 * * ? *)` for 8:00 AM UTC daily execution
- **Target**: Lambda function with appropriate IAM permissions
- **Retry Policy**: Built-in EventBridge retry with exponential backoff

### Lambda Function (cost_tracker.py)
The main orchestrator containing three core managers:

#### Cost Manager
```python
class CostManager:
    def get_daily_costs(self, date: str) -> Dict[str, Any]
    def format_cost_data(self, cost_data: Dict) -> Dict[str, float]
    def calculate_total_cost(self, services: Dict[str, float]) -> float
```

- **Responsibilities**: Interact with Cost Explorer API, process cost data, handle API errors
- **Key Methods**:
  - `get_daily_costs()`: Retrieves cost data for specified date using `get_cost_and_usage` API
  - `format_cost_data()`: Processes raw API response into service-grouped costs
  - `calculate_total_cost()`: Sums all service costs for daily total

#### Email Manager
```python
class EmailManager:
    def create_email_content(self, cost_data: Dict, date: str) -> str
    def send_email(self, recipients: List[str], subject: str, body: str) -> bool
    def validate_email_config(self) -> bool
```

- **Responsibilities**: Format cost data into email content, send emails via SES
- **Key Methods**:
  - `create_email_content()`: Generates HTML email with cost breakdown
  - `send_email()`: Sends email using SES `send_email` API
  - `validate_email_config()`: Validates sender email and SES configuration

#### Configuration Manager
```python
class ConfigManager:
    def get_email_recipients(self) -> List[str]
    def get_sender_email(self) -> str
    def validate_config(self) -> bool
```

- **Responsibilities**: Manage environment variables and configuration validation
- **Environment Variables**:
  - `EMAIL_RECIPIENTS`: Comma-separated list of recipient emails
  - `SENDER_EMAIL`: Verified SES sender email address
  - `AWS_REGION`: AWS region for SES and Cost Explorer

## Data Models

### Cost Data Structure
```python
@dataclass
class DailyCostReport:
    date: str
    total_cost: float
    currency: str
    services: Dict[str, float]
    estimated: bool
    
@dataclass
class ServiceCost:
    service_name: str
    amount: float
    unit: str
```

### API Response Models
The Cost Explorer API returns data in this structure:
```python
{
    "ResultsByTime": [
        {
            "TimePeriod": {"Start": "2025-01-01", "End": "2025-01-02"},
            "Total": {"BlendedCost": {"Amount": "12.34", "Unit": "USD"}},
            "Groups": [
                {
                    "Keys": ["Amazon Elastic Compute Cloud - Compute"],
                    "Metrics": {"BlendedCost": {"Amount": "8.50", "Unit": "USD"}}
                }
            ],
            "Estimated": false
        }
    ]
}
```

### Email Template Structure
```html
<!DOCTYPE html>
<html>
<head><title>Daily AWS Cost Report</title></head>
<body>
    <h2>AWS Cost Report for {date}</h2>
    <p><strong>Total Daily Cost: ${total_cost} {currency}</strong></p>
    <h3>Cost Breakdown by Service:</h3>
    <ul>
        {service_list}
    </ul>
    <p><em>Report generated at {timestamp}</em></p>
</body>
</html>
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Based on the prework analysis and property reflection, the following consolidated properties ensure system correctness:

**Property 1: Cost data processing accuracy**
*For any* valid cost data retrieved from Cost Explorer, the total calculated cost should equal the sum of all individual service costs, costs should be grouped by service name, currency amounts should be formatted with appropriate precision, and zero-cost days should be handled without errors.
**Validates: Requirements 1.3, 1.4, 5.2, 5.3, 5.4, 5.5**

**Property 2: Email content completeness**
*For any* valid cost data, the generated email should contain the date, total cost in readable format, currency information, and a complete breakdown of costs by AWS service.
**Validates: Requirements 2.2, 2.3, 2.4**

**Property 3: Configuration validation correctness**
*For any* set of environment variables, the configuration validation should correctly accept valid email recipients and sender configurations while rejecting invalid email addresses, missing required variables, and improperly configured SES settings.
**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

**Property 4: API interaction reliability**
*For any* valid date, when the Lambda function retrieves cost data from Cost Explorer, it should make API calls with correct parameters for the current calendar day in UTC and handle both successful responses and empty data gracefully.
**Validates: Requirements 1.2, 5.1, 4.2**

**Property 5: Error handling and retry behavior**
*For any* failure scenario (AWS API failures, SES failures, timeouts), the system should implement appropriate retry logic with exponential backoff, log descriptive error messages, and never send partial or incorrect data in notifications.
**Validates: Requirements 1.5, 2.5, 4.1, 4.3, 4.4, 4.5**

**Property 6: Email delivery consistency**
*For any* calculated daily costs, when the system attempts to send notifications, it should successfully deliver emails to all configured recipients or handle failures with appropriate error logging and retry attempts.
**Validates: Requirements 2.1, 2.5**

## Error Handling

### Cost Explorer API Errors
- **Timeout/Rate Limiting**: Implement exponential backoff with maximum 3 retries
- **Invalid Date Range**: Validate date parameters before API calls
- **No Data Available**: Handle empty responses gracefully, send "No costs incurred" message
- **Access Denied**: Log error and notify via CloudWatch, do not send email

### SES Email Errors
- **Invalid Recipients**: Validate email addresses before sending
- **SES Not Configured**: Check sender verification status
- **Send Failures**: Retry once, then log failure without crashing
- **Rate Limiting**: Implement basic retry with delay

### Lambda Function Errors
- **Timeout Prevention**: Set appropriate timeout (5 minutes) and monitor execution time
- **Memory Issues**: Configure sufficient memory (256MB) for JSON processing
- **Environment Variable Missing**: Fail fast with descriptive error messages
- **Partial Data**: Never send incomplete cost reports, prefer no email over wrong data

## Testing Strategy

### Unit Testing Approach
The system will use Python's `unittest` framework for specific examples and edge cases:

- **Cost Manager Tests**: Test API response parsing, error handling, zero-cost scenarios
- **Email Manager Tests**: Test email formatting, SES integration, recipient validation
- **Configuration Tests**: Test environment variable parsing, validation logic
- **Integration Tests**: Test end-to-end flow with mocked AWS services

### Property-Based Testing Approach
The system will use the `hypothesis` library for comprehensive input validation:

- **Minimum 100 iterations** per property test to ensure thorough coverage
- **Property tests** will validate universal behaviors across randomized inputs
- **Test tagging format**: `# Feature: daily-cost-tracker, Property {number}: {property_text}`

**Property Test Configuration**:
- Use `hypothesis.strategies` to generate valid cost data, email addresses, and dates
- Test edge cases through randomized input generation
- Validate that properties hold across all generated test cases
- Each property test references its corresponding design document property

**Dual Testing Benefits**:
- Unit tests catch specific integration bugs and validate concrete examples
- Property tests ensure correctness across the full input space
- Together they provide comprehensive coverage of both specific scenarios and general behavior
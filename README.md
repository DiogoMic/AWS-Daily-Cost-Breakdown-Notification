# AWS Daily Cost Breakdown Automation

This project creates a Lambda function that generates daily AWS cost breakdowns and sends them via email using Amazon SES. The function is triggered automatically every morning at 07:00 UTC using Amazon EventBridge Scheduler to deliver the previous day's cost analysis.

## Architecture

- **Lambda Function**: Fetches cost data from AWS Cost Explorer API and formats it into an email
- **EventBridge Scheduler**: Triggers the Lambda function daily at 07:00 UTC
- **Amazon SES**: Sends the formatted cost breakdown email
- **CloudWatch Logs**: Stores Lambda execution logs with configurable retention
- **IAM Roles**: Provides necessary permissions for Cost Explorer and SES access
- **Multi-Region Support**: Monitor costs across multiple AWS regions

## Features

- **Multi-Region Cost Analysis**: Monitor costs across multiple AWS regions
- **Daily Cost Breakdown**: Shows costs by AWS service for the previous day
- **Regional Breakdown**: Displays cost distribution across your target regions
- **Rich Email Format**: Both HTML and text versions with formatted tables
- **Cost Analysis**: Includes blended costs, unblended costs, usage quantities, and percentages
- **Morning Delivery**: Receive yesterday's costs at 7 AM UTC for actionable insights
- **Error Handling**: Comprehensive error handling and logging
- **Security**: Least-privilege IAM policies with optional VPC and KMS encryption
- **Flexible Configuration**: Customizable regions, schedule, and retention settings

## Prerequisites

### Required
1. **AWS Account** with appropriate permissions
2. **AWS CLI** installed and configured
3. **SES Email Verification**: Your email address must be verified in Amazon SES
4. **Cost Explorer Enabled**: Must be enabled in your AWS account

### Optional
- **AWS CDK** (if using CDK deployment method)
- **Python 3.11+** (for local development/testing)

## Deployment Methods

This solution can be deployed using either **CloudFormation** (recommended) or **AWS CDK**:

### Method 1: CloudFormation (Recommended)

#### Prerequisites for CloudFormation
- AWS CLI installed and configured
- Appropriate IAM permissions for CloudFormation, Lambda, SES, EventBridge, and Cost Explorer

#### Quick Deployment

1. **Verify Your Email in SES**:
```bash
aws ses verify-email-identity --email-address your-email@example.com
```
#### Manual CloudFormation Deployment

**Via AWS Console:**
1. Go to [CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
2. Click "Create stack" → "With new resources"
3. Upload the `cloudformation-template.yaml` file
4. Fill in parameters:
   - **EmailAddress**: Your verified email address
   - **TargetRegions**: `us-east-1,us-west-2,eu-west-1` (or your preferred regions)
   - **ScheduleExpression**: `cron(0 7 * * ? *)` (7 AM UTC daily)
5. Acknowledge IAM resource creation
6. Click "Create stack"

**Via AWS CLI:**
```bash
aws cloudformation create-stack \
  --stack-name aws-cost-breakdown \
  --template-body file://cloudformation-template.yaml \
  --parameters \
    ParameterKey=EmailAddress,ParameterValue=your-email@example.com \
    ParameterKey=TargetRegions,ParameterValue="us-east-1,us-west-2,eu-west-1" \
  --capabilities CAPABILITY_NAMED_IAM
```

#### Advanced CloudFormation Options

**With KMS Encryption:**
```bash
./deploy-cloudformation.sh \
  -e your-email@example.com \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
```

**Custom Schedule (e.g., 8 AM UTC):**
```bash
./deploy-cloudformation.sh \
  -e your-email@example.com \
  -s "cron(0 8 * * ? *)"
```

### Required Parameters
- **EmailAddress**: Your verified email address for receiving cost reports
- **TargetRegions**: Comma-separated list of AWS regions to monitor

### Optional Parameters
- **ScheduleExpression**: Cron expression for timing (default: `cron(0 7 * * ? *)`)
- **LogRetentionDays**: CloudWatch log retention period (default: 7 days)
- **VpcSecurityGroupIds**: Security Group IDs for VPC deployment (optional)
- **VpcSubnetIds**: Subnet IDs for VPC deployment (optional)
- **KmsKeyId**: KMS Key ARN for log encryption (optional)

### Environment Variables (Set Automatically)
- `EMAIL_ADDRESS`: Your verified email address
- `TARGET_REGIONS`: Comma-separated list of regions
- `LOG_LEVEL`: Logging level (INFO)

## Schedule Configuration

The default schedule runs daily at **07:00 UTC** to deliver the previous day's costs in the morning. Common schedule options:

```bash
# 6 AM UTC
cron(0 6 * * ? *)

# 7 AM UTC (default)
cron(0 7 * * ? *)

# 8 AM UTC
cron(0 8 * * ? *)

# 9 AM UTC
cron(0 9 * * ? *)
```

**Time Zone Examples for 7 AM UTC:**
- **EST/EDT**: 2 AM / 3 AM (East Coast US)
- **PST/PDT**: 11 PM / 12 AM (West Coast US)
- **CET/CEST**: 8 AM / 9 AM (Central Europe)
- **GMT**: 7 AM (UK)

## Email Format

The cost breakdown email includes:

### Summary Section
- Total daily cost across all monitored regions
- Number of services with charges
- List of monitored regions
- Number of regions with actual costs
- Report date

### Regional Breakdown Table
- Region name
- Cost per region
- Percentage of total cost

### Service Breakdown Table
- Service name
- Blended cost (with discounts applied)
- Unblended cost (without discounts)
- Usage quantity
- Percentage of total cost

### Additional Information
- Generation timestamp
- Explanatory notes about data timing

## Testing

### Manual Testing via Console
1. Go to [Lambda Console](https://console.aws.amazon.com/lambda/)
2. Find `cost-breakdown-function`
3. Click "Test" tab
4. Create test event: `{"source": "manual-test"}`
5. Click "Test"

### Manual Testing via CLI
```bash
aws lambda invoke \
  --function-name cost-breakdown-function \
  --payload '{"source": "manual-test"}' \
  response.json

cat response.json
```

### Check Logs
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/cost-breakdown-function \
  --start-time $(date -d '10 minutes ago' +%s)000
```

## Monitoring

### CloudWatch Logs
Logs are stored in `/aws/lambda/cost-breakdown-function` with configurable retention.

### CloudWatch Metrics (Available)
- Function duration, errors, invocations
- Custom application metrics (if using enhanced version)

### Alarms (Optional)
You can create CloudWatch alarms for:
- Function errors
- High daily costs
- Email delivery failures

## Cost Considerations

### AWS Services Used
- **Lambda**: ~$0.20 per million requests + compute time
- **EventBridge Scheduler**: $1.00 per million invocations
- **SES**: $0.10 per 1,000 emails (first 62,000 free per month)
- **Cost Explorer API**: $0.01 per request
- **CloudWatch Logs**: $0.50 per GB ingested

### Estimated Monthly Cost
For daily execution:
- Lambda: ~$0.01/month (30 invocations)
- EventBridge: ~$0.001/month
- SES: ~$0.003/month (30 emails)
- Cost Explorer: ~$0.30/month (30 API calls)
- CloudWatch: ~$0.01/month

**Total: ~$0.32/month**

## Security

### IAM Permissions
The Lambda function has minimal required permissions:
- Cost Explorer: Read-only access to cost and usage data
- SES: Send email permissions for verified identities only
- CloudWatch: Basic execution role for logging

### Security Features
- Least-privilege IAM policies
- No hardcoded credentials
- Encrypted environment variables
- Optional VPC deployment for enhanced security
- Optional KMS encryption for CloudWatch logs
- Cross-partition ARN support

## Troubleshooting

### Common Issues

1. **Email not received**
   - Verify email address in SES: `aws ses get-identity-verification-attributes --identities your-email@example.com`
   - Check spam folder
   - Verify SES sending limits

2. **Permission errors**
   - Ensure Cost Explorer is enabled in your account
   - Check IAM role permissions
   - Verify SES permissions for the email address

3. **No cost data**
   - Cost Explorer data has 24-48 hour delay
   - Ensure you have AWS usage in target regions
   - Check if Cost Explorer is enabled

4. **Function timeout**
   - Default timeout is 5 minutes (should be sufficient)
   - Check CloudWatch logs for specific errors

### Debugging Steps
1. Check CloudWatch logs for error details
2. Test Lambda function manually
3. Verify SES email sending separately
4. Check Cost Explorer API access
5. Review IAM permissions

## Customization

### Email Template
The email format can be customized by modifying the Lambda function code to:
- Change styling and layout
- Add additional cost metrics
- Include different data groupings
- Add charts or graphs

### Cost Analysis Enhancements
- Weekly/monthly comparisons
- Budget alerts and thresholds
- Resource-level details
- Cost anomaly detection
- Trend analysis

### Notification Channels
Extend beyond email by adding:
- Slack notifications
- SMS via SNS
- Microsoft Teams webhooks
- Discord notifications
- Dashboard updates

### Cleanup
**CloudFormation**:
```bash
aws cloudformation delete-stack --stack-name aws-cost-breakdown
```
This removes all resources except SES email verification.

## License

This project is licensed under the MIT License.
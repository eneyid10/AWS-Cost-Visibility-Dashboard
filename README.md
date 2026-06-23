# AWS Cost Visibility Dashboard

**Cloud:** AWS &nbsp;|&nbsp; **IaC:** Terraform

> Build a real-time cost tracking and alerting system that gives business owners full visibility into their AWS spend — with automatic email alerts before bills become a problem.

---

## Table of Contents

- [Business Problem](#business-problem)
- [What Gets Built](#what-gets-built)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Deployment](#step-by-step-deployment)
  - [Step 1 — variables.tf](#step-1--variablestf)
  - [Step 2 — terraform.tfvars](#step-2--terraformtfvars)
  - [Step 3 — main.tf](#step-3--maintf)
  - [Step 4 — lambda\_function.py](#step-4--lambda_functionpy)
  - [Step 5 — outputs.tf](#step-5--outputstf)
  - [Step 6 — Deploy with Terraform](#step-6--deploy-with-terraform)
  - [Step 7 — Verify SES Email Identity](#step-7--verify-ses-email-identity)
  - [Step 8 — Build the CloudWatch Dashboard](#step-8--build-the-cloudwatch-dashboard)
- [Verification Checklist](#verification-checklist)
- [Troubleshooting](#troubleshooting)
- [Skills Demonstrated](#skills-demonstrated)
- [Teardown](#teardown)

---

## Business Problem

Most small businesses move to the cloud expecting lower costs — then the AWS bills arrive full of line items like `Amazon EC2 — $340` that nobody in the business can interpret, predict, or explain to a stakeholder.

This project solves that by building a system that:

- **Tracks** spending across all AWS services against a monthly budget ceiling
- **Alerts** automatically when spending hits thresholds ($50, $100, $200)
- **Notifies** via email using Lambda + SES triggered by an SNS budget alert
- **Displays** a live dashboard in CloudWatch showing spend trends and account activity
- **Routes** all management API events from CloudTrail into a CloudWatch Log Group for querying

---

## What Gets Built

```
cost-dashboard-[yourname]  (Terraform-managed, tagged)
├── AWS Budgets                → watches $50 / $100 / $200 monthly thresholds
├── Amazon SNS Topic           → receives budget alerts and fans out to subscribers
├── AWS Lambda Function        → triggered by SNS; formats and sends alert email via SES
├── Amazon SES Identity        → verified email sender for Lambda notifications
├── IAM Role + Policies        → least-privilege execution role for Lambda
├── AWS CloudTrail Trail       → records all management API events in the account
├── CloudWatch Log Group       → stores CloudTrail events (30-day retention)
└── CloudWatch Dashboard       → live spend + activity visibility in the console
```

---

## Architecture

<img width="2720" height="2240" alt="aws_cost_dashboard_architecture" src="https://github.com/user-attachments/assets/f2abe0a3-151e-4f72-8356-b2b16a4b6598" />


**Data flow summary:**

1. AWS Budgets monitors monthly account-level spend against a $200 ceiling.
2. When spend crosses a threshold (25%, 50%, or 100%), Budgets publishes an alert to an SNS topic.
3. SNS fans out: one subscription triggers the Lambda function, another sends a direct email to the subscriber.
4. Lambda receives the SNS payload, formats it into a readable message, and sends an alert email via SES.
5. In parallel, CloudTrail captures every management API call in the account and streams events into a CloudWatch Log Group.
6. A CloudWatch Dashboard provides a live view of billing metrics and account activity pulled from CloudWatch and Cost Explorer.

---

## Prerequisites

Complete the following before starting. If you have done a prior AWS lab with Terraform installed, skip to step 4.

### macOS

```bash
# 1. Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. Install Terraform
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform --version

# 3. Install AWS CLI
brew install awscli
aws --version

# 4. Configure credentials
aws configure
# Enter your AWS Access Key ID, Secret Access Key, default region (us-east-1), and output format (json)

# 5. Verify authentication
aws sts get-caller-identity
```

### Windows (PowerShell)

```powershell
# 1. Download Terraform from https://developer.hashicorp.com/terraform/install
#    Extract to C:\terraform\ then add to PATH:
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\terraform", "User")
terraform --version

# 2. Install AWS CLI from https://aws.amazon.com/cli/
aws --version

# 3. Configure credentials
aws configure

# 4. Verify authentication
aws sts get-caller-identity
```

> **AWS credentials note:** For a lab environment, using an IAM user with programmatic access (Access Key + Secret Key) is the fastest path. For production, use IAM Identity Center (SSO) or instance profiles instead.

---

## Project Structure

```
cost-dashboard-001/
├── main.tf                # All AWS resources
├── variables.tf           # Input variable declarations
├── terraform.tfvars       # Your personal values (name, email, region)
├── outputs.tf             # Terminal output values after apply
└── lambda_function.py     # Lambda handler — formats and sends the alert email
```

### Create the directory

**macOS:**
```bash
mkdir ~/cost-dashboard-001
cd ~/cost-dashboard-001
touch main.tf variables.tf outputs.tf terraform.tfvars lambda_function.py
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path "$HOME\cost-dashboard-001"
cd "$HOME\cost-dashboard-001"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars, lambda_function.py
```

---

## Step-by-Step Deployment

### Step 1 — variables.tf

Declares the input variables Terraform expects. You supply the actual values in `terraform.tfvars`.

```hcl
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
}

variable "aws_region" {
  description = "AWS region to deploy into."
  type        = string
  default     = "us-east-1"
}

variable "alert_email" {
  description = "Email address to receive cost alert notifications. Must be SES-verified."
  type        = string
}

variable "budget_limit" {
  description = "Monthly budget ceiling in USD."
  type        = number
  default     = 200
}

variable "tags" {
  type = map(string)
  default = {
    project     = "cost-dashboard"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

---

### Step 2 — terraform.tfvars

Supply your personal values here. Replace the email with the address you want to receive alerts.

```hcl
yourname     = "eneyi"
aws_region   = "us-east-1"
alert_email  = "your.email@example.com"
budget_limit = 200
```

---

### Step 3 — main.tf

Each resource block is explained before the code so you understand what it does and why it is written the way it is.

#### Provider and data sources

The `aws` provider is the Terraform plugin that communicates with AWS. The `aws_caller_identity` data source reads your active `aws configure` session and exposes your AWS account ID, which is needed to scope IAM policies and SNS subscriptions correctly.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = var.tags
  }
}

data "aws_caller_identity" "current" {}
```

---

#### SNS Topic

Amazon SNS (Simple Notification Service) is the fan-out layer between AWS Budgets and your notification targets. When a budget threshold fires, Budgets publishes a message to this topic. The topic then delivers it to all subscribers simultaneously — in this case, the Lambda function and a direct email subscription.

```hcl
resource "aws_sns_topic" "cost_alerts" {
  name = "cost-alerts-${var.yourname}"
}
```

SNS requires an explicit resource policy to allow AWS Budgets to publish to it. Without this policy, Budgets will fail silently when attempting to send alerts.

```hcl
resource "aws_sns_topic_policy" "cost_alerts" {
  arn = aws_sns_topic.cost_alerts.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowBudgetsPublish"
        Effect = "Allow"
        Principal = {
          Service = "budgets.amazonaws.com"
        }
        Action   = "SNS:Publish"
        Resource = aws_sns_topic.cost_alerts.arn
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })
}
```

---

#### SNS Email Subscription

This subscribes your alert email address directly to the SNS topic. AWS will send a confirmation email to this address immediately after `terraform apply`. **You must click the confirmation link in that email before alerts will be delivered.**

```hcl
resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.cost_alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

---

#### IAM Role for Lambda

Lambda needs an execution role that grants it permission to write logs to CloudWatch and send emails via SES. Following least-privilege, the role is scoped to only what this specific function needs — nothing more.

```hcl
resource "aws_iam_role" "lambda_exec" {
  name = "lambda-cost-alert-role-${var.yourname}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_ses" {
  name = "lambda-ses-send-${var.yourname}"
  role = aws_iam_role.lambda_exec.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ses:SendEmail", "ses:SendRawEmail"]
      Resource = "*"
    }]
  })
}
```

---

#### Lambda Function

The Lambda function receives the SNS event payload (a JSON object describing which budget threshold fired), formats it into a human-readable email, and sends it via SES. Terraform packages `lambda_function.py` into a zip file and uploads it automatically.

```hcl
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/lambda_function.py"
  output_path = "${path.module}/lambda_function.zip"
}

resource "aws_lambda_function" "cost_alert" {
  function_name    = "cost-alert-${var.yourname}"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  handler          = "lambda_function.lambda_handler"
  runtime          = "python3.12"
  role             = aws_iam_role.lambda_exec.arn

  environment {
    variables = {
      ALERT_EMAIL = var.alert_email
      AWS_REGION  = var.aws_region
    }
  }
}
```

Grant SNS permission to invoke the Lambda function. Without this, SNS will receive a 403 when trying to trigger it.

```hcl
resource "aws_lambda_permission" "sns_invoke" {
  statement_id  = "AllowSNSInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cost_alert.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.cost_alerts.arn
}

resource "aws_sns_topic_subscription" "lambda" {
  topic_arn = aws_sns_topic.cost_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.cost_alert.arn
}
```

---

#### SES Email Identity

Amazon SES (Simple Email Service) is what Lambda uses to send formatted alert emails. In sandbox mode (the default for new AWS accounts), both the sender and recipient addresses must be verified. This resource registers the sender identity — you will confirm it in Step 7.

```hcl
resource "aws_ses_email_identity" "alert_sender" {
  email = var.alert_email
}
```

---

#### AWS Budgets

`aws_budgets_budget` creates a monthly budget at the account level. This is what watches your overall AWS spend and publishes alerts to SNS when you cross thresholds.

- `budget_type = "COST"` tracks actual dollar spend
- `time_unit = "MONTHLY"` resets at the start of each calendar month, matching how AWS bills
- `limit_amount = var.budget_limit` sets the $200 monthly ceiling
- Each `notification` block defines one threshold: 25% ($50), 50% ($100), and 100% ($200)
- `comparison_operator = "GREATER_THAN"` fires when spending crosses upward through the threshold
- `notification_type = "ACTUAL"` alerts on real spend (not forecasted spend)

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "budget-cost-${var.yourname}"
  budget_type  = "COST"
  limit_amount = tostring(var.budget_limit)
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 25
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 50
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }
}
```

---

#### CloudWatch Log Group

This is where CloudTrail ships management event logs. Setting a 30-day retention policy keeps lab costs near zero — CloudWatch charges for data stored beyond the free tier.

```hcl
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/cost-dashboard-${var.yourname}"
  retention_in_days = 30
}
```

---

#### IAM Role for CloudTrail

CloudTrail needs permission to write log events into the CloudWatch Log Group. This role is separate from the Lambda role — each AWS service gets its own scoped execution role.

```hcl
resource "aws_iam_role" "cloudtrail" {
  name = "cloudtrail-cost-role-${var.yourname}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "cloudtrail.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "cloudtrail_logs" {
  name = "cloudtrail-logs-${var.yourname}"
  role = aws_iam_role.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Resource = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
    }]
  })
}
```

---

#### CloudTrail Trail

CloudTrail records every management API call in your AWS account — who created or deleted what, when, and from where. Without it, CloudWatch has no activity data to query.

- `is_multi_region_trail = false` scopes the trail to the deployed region only (sufficient for a lab)
- `include_global_service_events = true` captures IAM and STS events, which are global but relevant
- `enable_log_file_validation = true` adds SHA-256 digest files so you can verify log integrity

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "trail-cost-${var.yourname}"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.bucket
  cloud_watch_logs_group_arn    = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn     = aws_iam_role.cloudtrail.arn
  is_multi_region_trail         = false
  include_global_service_events = true
  enable_log_file_validation    = true
}
```

CloudTrail requires an S3 bucket to store raw log archives. Even though you will primarily query logs through CloudWatch, CloudTrail mandates an S3 destination.

```hcl
resource "aws_s3_bucket" "cloudtrail" {
  bucket        = "cloudtrail-cost-${var.yourname}-${data.aws_caller_identity.current.account_id}"
  force_destroy = true
}

resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck"
        Effect = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
      },
      {
        Sid    = "AWSCloudTrailWrite"
        Effect = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
        Condition = {
          StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" }
        }
      }
    ]
  })
}
```

---

#### CloudWatch Dashboard

The dashboard provides a live view of billing metrics and account activity directly in the AWS Console. The two built-in widgets query the `AWS/Billing` namespace for estimated charges and your CloudTrail log group for recent management events.

> **Note:** Billing metrics must be enabled in your account before they appear. Go to **Billing → Billing Preferences → Receive Billing Alerts** and enable it. This is a one-time manual step — it cannot be done via Terraform.

```hcl
resource "aws_cloudwatch_dashboard" "cost" {
  dashboard_name = "CostVisibility-${var.yourname}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Estimated AWS Charges (USD)"
          view   = "timeSeries"
          period = 86400
          metrics = [
            ["AWS/Billing", "EstimatedCharges", "Currency", "USD"]
          ]
          region = "us-east-1"
          stat   = "Maximum"
        }
      },
      {
        type   = "log"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Recent Management Events (CloudTrail)"
          region  = var.aws_region
          logGroupNames = [aws_cloudwatch_log_group.cloudtrail.name]
          query   = "SOURCE '${aws_cloudwatch_log_group.cloudtrail.name}' | fields @timestamp, eventName, userIdentity.arn | sort @timestamp desc | limit 20"
          view    = "table"
        }
      }
    ]
  })
}
```

---

### Step 4 — lambda\_function.py

This is the Lambda handler. It parses the SNS alert payload and sends a formatted email via SES. Terraform packages this file automatically into a zip during `apply`.

```python
import json
import os
import boto3


def lambda_handler(event, context):
    ses = boto3.client("ses", region_name=os.environ["AWS_REGION"])
    alert_email = os.environ["ALERT_EMAIL"]

    for record in event.get("Records", []):
        message_raw = record["Sns"]["Message"]
        try:
            message = json.loads(message_raw)
        except json.JSONDecodeError:
            message = {"detail": message_raw}

        budget_name = message.get("budgetName", "Unknown budget")
        threshold = message.get("thresholdExceeded", "Unknown threshold")
        actual_spend = message.get("actualSpend", {}).get("amount", "Unknown")
        currency = message.get("actualSpend", {}).get("unit", "USD")

        subject = f"AWS Cost Alert — {budget_name} threshold reached"
        body = (
            f"Your AWS budget alert has fired.\n\n"
            f"Budget:           {budget_name}\n"
            f"Threshold:        {threshold}%\n"
            f"Current spend:    {actual_spend} {currency}\n\n"
            f"Log in to the AWS Console to review your Cost Explorer breakdown:\n"
            f"https://console.aws.amazon.com/cost-management/home#/cost-explorer\n\n"
            f"This alert was sent by the cost-dashboard Terraform project."
        )

        ses.send_email(
            Source=alert_email,
            Destination={"ToAddresses": [alert_email]},
            Message={
                "Subject": {"Data": subject},
                "Body": {"Text": {"Data": body}},
            },
        )
        print(f"Alert sent: {subject}")

    return {"statusCode": 200}
```

---

### Step 5 — outputs.tf

Outputs print useful values to your terminal after `terraform apply` completes.

```hcl
output "sns_topic_arn" {
  description = "SNS topic ARN — attach additional subscribers here if needed."
  value       = aws_sns_topic.cost_alerts.arn
}

output "lambda_function_name" {
  description = "Lambda function name — use this to invoke a test event from the CLI."
  value       = aws_lambda_function.cost_alert.function_name
}

output "cloudtrail_log_group" {
  description = "CloudWatch Log Group name for CloudTrail events."
  value       = aws_cloudwatch_log_group.cloudtrail.name
}

output "cloudwatch_dashboard_url" {
  description = "Direct link to the CloudWatch cost dashboard."
  value       = "https://${var.aws_region}.console.aws.amazon.com/cloudwatch/home?region=${var.aws_region}#dashboards:name=CostVisibility-${var.yourname}"
}

output "cloudtrail_s3_bucket" {
  description = "S3 bucket storing raw CloudTrail log archives."
  value       = aws_s3_bucket.cloudtrail.bucket
}
```

---

### Step 6 — Deploy with Terraform

These commands are identical on macOS and Windows PowerShell.

```bash
terraform init
```

Expected output: `Terraform has been successfully initialized.`

```bash
terraform plan
```

Review the plan. You should see **13 resources to add**: SNS topic, SNS topic policy, SNS subscriptions (×2), IAM roles (×2), IAM policies (×3), Lambda function, Lambda permission, SES identity, Budgets budget, CloudWatch log group, CloudTrail trail, S3 bucket, S3 bucket policy, and CloudWatch dashboard.

```bash
terraform apply
```

Type `yes` when prompted. Deployment takes approximately 2–3 minutes.

---

### Step 7 — Verify SES Email Identity

AWS SES sandbox mode requires email verification before any messages can be sent. Two confirmation emails will arrive immediately after `terraform apply`:

1. **SNS subscription confirmation** — subject: `AWS Notification - Subscription Confirmation`. Click **Confirm subscription**.
2. **SES sender verification** — subject: `Amazon Web Services – Email Address Verification Request`. Click the verification link.

Both must be confirmed before alert emails will be delivered.

> **SES sandbox limitation:** In sandbox mode, SES can only send to verified addresses. For a lab this is fine since sender and recipient are the same address. To send to unverified recipients in production, request SES production access from the AWS Console under **SES → Account dashboard → Request production access**.

**Test the Lambda function manually** (optional but recommended):

```bash
aws lambda invoke \
  --function-name cost-alert-[yourname] \
  --payload '{"Records":[{"Sns":{"Message":"{\"budgetName\":\"budget-cost-[yourname]\",\"thresholdExceeded\":\"25\",\"actualSpend\":{\"amount\":\"52.34\",\"unit\":\"USD\"}}"}}]}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
```

A `{"statusCode": 200}` response and a test alert email in your inbox confirms the pipeline is working end to end.

---

### Step 8 — Build the CloudWatch Dashboard

Terraform already created the CloudWatch Dashboard with two starter widgets. To view and extend it:

1. In the AWS Console, navigate to **CloudWatch → Dashboards**
2. Click **CostVisibility-[yourname]**
3. You will see:
   - **Estimated AWS Charges** — a time-series graph of daily estimated charges pulled from the `AWS/Billing` metric namespace
   - **Recent Management Events** — a table of the 20 most recent CloudTrail events showing timestamp, event name, and the IAM principal that triggered each action
4. To add more widgets, click **+ Add widget** in the top-right corner
5. To add a spend breakdown by service, add a new **Metrics** widget and browse **AWS/Billing → ServiceName** to select individual services

> **Billing metrics note:** If the Estimated Charges graph shows no data, confirm that billing alerts are enabled in your account: **Billing → Billing Preferences → Receive Billing Alerts → Save preferences**. Metrics appear within 24 hours of enabling this setting.

---

## Verification Checklist

| Resource | Location in Console | Expected Status |
|---|---|---|
| SNS topic `cost-alerts-[yourname]` | SNS → Topics | Active |
| SNS email subscription | SNS → Subscriptions | Status: **Confirmed** (not PendingConfirmation) |
| Lambda `cost-alert-[yourname]` | Lambda → Functions | State: **Active** |
| SES identity `your.email@example.com` | SES → Verified identities | Status: **Verified** |
| Budget `budget-cost-[yourname]` | Budgets → Budgets | Active |
| CloudTrail `trail-cost-[yourname]` | CloudTrail → Trails | Logging: **On** |
| CloudWatch Log Group `/aws/cloudtrail/cost-dashboard-[yourname]` | CloudWatch → Log groups | Exists, events appearing |
| CloudWatch Dashboard `CostVisibility-[yourname]` | CloudWatch → Dashboards | Visible, widgets rendering |

---

## Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `Error: creating Budgets Budget: AccessDeniedException` | IAM user lacks `budgets:CreateBudget` permission | Attach the `AWSBudgetsActionsWithAWSResourceControlAccess` managed policy to your IAM user |
| SNS subscription stuck at `PendingConfirmation` | Confirmation email not clicked | Check spam folder; re-send from SNS → Subscriptions → Request confirmation |
| Lambda test returns SES `MessageRejected` | SES email identity not yet verified | Verify the email in SES → Verified identities before invoking |
| CloudTrail shows `InsufficientS3BucketPolicyException` | S3 bucket policy not applied before trail creation | Run `terraform apply` again — Terraform will detect the dependency and retry |
| Billing metrics widget shows no data | Billing alerts not enabled in account | Enable in Billing → Billing Preferences → Receive Billing Alerts |
| `Error: creating CloudWatch Dashboard: InvalidParameterInput` | Malformed dashboard JSON | Run `terraform plan` again and check the dashboard_body output for JSON syntax errors |

---

## Skills Demonstrated

| Category | What This Lab Covers |
|---|---|
| **Infrastructure as Code** | Terraform AWS provider, resource blocks, `archive_file` data source, `default_tags`, outputs |
| **Cloud Cost Management** | AWS Budgets, threshold alerting by percentage, SNS subscriber integration |
| **Event-Driven Architecture** | SNS fan-out pattern, Lambda triggers, SNS resource policies |
| **Serverless Compute** | Lambda function authoring (Python), IAM execution roles, environment variables |
| **Email Delivery** | SES identity verification, programmatic email send via boto3, sandbox vs production mode |
| **Observability** | CloudTrail trail configuration, CloudWatch Log Groups, log retention policies |
| **Dashboarding** | CloudWatch Dashboard JSON, Billing metric namespace, CloudWatch Logs Insights queries |
| **IAM & Least Privilege** | Separate roles per service, scoped inline policies, `sts:AssumeRole` trust policies |

---

## Teardown

Destroy all resources when you are done to avoid ongoing charges.

```bash
terraform destroy
```

Type `yes` when prompted. This deletes all provisioned resources including the SNS topic, Lambda function, SES identity, CloudTrail trail, S3 bucket, CloudWatch log group, and dashboard.

> **Note:** The S3 bucket is created with `force_destroy = true` so Terraform can empty and delete it automatically. Without this flag, Terraform would refuse to delete a non-empty bucket.

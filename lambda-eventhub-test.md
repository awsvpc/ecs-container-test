Below is a complete, modular Terraform example that creates an AWS Lambda function inside a VPC, provisions the required IAM roles and permissions, and uses Amazon EventBridge for both scheduled execution and EC2 reboot events.

The Lambda function will:

- Accept an EC2 instance ID as input for testing.
- Retrieve and print the instance ID, current instance state/status, and instance type.
- Run automatically every hour using an EventBridge schedule.
- Trigger when an EC2 instance reboot-related event occurs.
- Run inside the VPC using the two subnet IDs and VPC ID you provide.
- Use environment variables to configure the instance ID, VPC settings, and runtime behavior.

## 1. Architecture

EventBridge Scheduler

Every 1 hour

EventBridge EC2 event rule

Reboot-related events

AWS Lambda

Python 3.12 · VPC · Two subnets

Amazon EC2

Instance ID · State · Instance type

Important design details:

- EventBridge sends an event to Lambda when an EC2 reboot-related event matches the configured rule. This is separate from the hourly schedule.
- The hourly invocation checks the instance configured in an environment variable. The reboot event invocation checks the instance ID from the event payload.
- Lambda needs EC2 read permissions and permissions to create/manage its VPC network interfaces.
- A VPC-attached Lambda does not automatically have internet access. The example uses the EC2 API through the AWS SDK; configure a suitable VPC endpoint or NAT connectivity if needed.

## 2. Project structure

Create the following files in one Terraform project directory:

text

```
ec2-lambda-terraform/
├── provider.tf
├── variables.tf
├── data.tf
├── iam.tf
├── lambda.tf
├── main.tf
├── output.tf
└── lambda_function.py
```

## 3. `provider.tf`

Configures the AWS provider and the Archive provider used to package the Python Lambda function.

hcl

```

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

The provider uses your configured AWS credentials, such as an AWS profile, environment variables, or an IAM role. Avoid hardcoding access keys in Terraform.

## 4. `variables.tf`

All configurable values are defined here, including the VPC, two subnets, instance ID, Lambda settings, and schedule.

hcl

```

variable "aws_region" {
  description = "AWS region for the deployment"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name prefix for resources"
  type        = string
  default     = "ec2-monitor"
}

variable "vpc_id" {
  description = "VPC ID where Lambda will run"
  type        = string
}

variable "subnet_id_1" {
  description = "First subnet ID for Lambda"
  type        = string
}

variable "subnet_id_2" {
  description = "Second subnet ID for Lambda"
  type        = string
}

variable "target_instance_id" {
  description = "EC2 instance ID to monitor during hourly checks and manual tests"
  type        = string
}

variable "lambda_timeout" {
  description = "Lambda timeout in seconds"
  type        = number
  default     = 30
}

variable "lambda_memory_size" {
  description = "Lambda memory in MB"
  type        = number
  default     = 128
}

variable "schedule_expression" {
  description = "EventBridge schedule expression"
  type        = string
  default     = "rate(1 hour)"
}
```

## 5. `data.tf`

Looks up the specified VPC and subnets, and validates that the subnets belong to the supplied VPC.

hcl

```

data "aws_vpc" "selected" {
  id = var.vpc_id
}

data "aws_subnet" "subnet_1" {
  id = var.subnet_id_1
}

data "aws_subnet" "subnet_2" {
  id = var.subnet_id_2
}

data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

locals {
  lambda_subnet_ids = [
    var.subnet_id_1,
    var.subnet_id_2
  ]

  common_tags = {
    Project   = var.project_name
    ManagedBy = "Terraform"
  }
}

check "subnets_belong_to_vpc" {
  assert {
    condition = (
      data.aws_subnet.subnet_1.vpc_id == data.aws_vpc.selected.id &&
      data.aws_subnet.subnet_2.vpc_id == data.aws_vpc.selected.id
    )
    error_message = "Both subnet IDs must belong to the supplied VPC."
  }
}

check "subnets_are_distinct" {
  assert {
    condition     = var.subnet_id_1 != var.subnet_id_2
    error_message = "Provide two different subnet IDs."
  }
}
```

## 6. `iam.tf`

Creates the Lambda execution role, the trust policy, and the minimum AWS permissions required for this implementation.

hcl

```

resource "aws_iam_role" "lambda_execution_role" {
  name = "${var.project_name}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [{
      Effect = "Allow"

      Principal = {
        Service = "lambda.amazonaws.com"
      }

      Action = "sts:AssumeRole"
    }]
  })

  tags = local.common_tags
}

# CloudWatch Logs and VPC network interface permissions
resource "aws_iam_role_policy_attachment" "lambda_vpc_execution" {
  role = aws_iam_role.lambda_execution_role.name

  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Read EC2 instance details and instance status
resource "aws_iam_role_policy" "ec2_read_policy" {
  name = "${var.project_name}-ec2-read"
  role = aws_iam_role.lambda_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"

    Statement = [{
      Effect = "Allow"

      Action = [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus"
      ]

      Resource = "*"
    }]
  })
}

# Allow EventBridge to invoke the Lambda function
resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_monitor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.hourly_check.arn
}

resource "aws_lambda_permission" "allow_reboot_event" {
  statement_id  = "AllowEC2RebootEventInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_monitor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.reboot_event.arn
}
```

The Lambda execution role has read-only EC2 permissions. It does not have permission to reboot, start, stop, or modify instances.

## 7. `lambda.tf`

Packages the Python code, creates the Lambda function in the supplied VPC subnets, and configures environment variables.

hcl

```

data "archive_file" "lambda_package" {
  type        = "zip"
  source_file = "${path.module}/lambda_function.py"
  output_path = "${path.module}/lambda_function.zip"
}

resource "aws_lambda_function" "ec2_monitor" {
  function_name = "${var.project_name}-function"
  description   = "Reports EC2 instance state, status, and instance type"

  role    = aws_iam_role.lambda_execution_role.arn
  handler = "lambda_function.lambda_handler"
  runtime = "python3.12"

  filename         = data.archive_file.lambda_package.output_path
  source_code_hash = data.archive_file.lambda_package.output_base64sha256

  timeout     = var.lambda_timeout
  memory_size = var.lambda_memory_size

  # Place Lambda network interfaces in the specified VPC subnets
  vpc_config {
    subnet_ids         = local.lambda_subnet_ids
    security_group_ids = [aws_security_group.lambda_sg.id]
  }

  environment {
    variables = {
      TARGET_INSTANCE_ID = var.target_instance_id
      VPC_ID             = var.vpc_id
      SUBNET_IDS         = join(",", local.lambda_subnet_ids)
      AWS_REGION         = var.aws_region
      LOG_LEVEL          = "INFO"
    }
  }

  depends_on = [
    aws_iam_role_policy_attachment.lambda_vpc_execution,
    aws_iam_role_policy.ec2_read_policy
  ]

  tags = local.common_tags
}

# Lambda security group
resource "aws_security_group" "lambda_sg" {
  name        = "${var.project_name}-lambda-sg"
  description = "Security group for EC2 monitoring Lambda"
  vpc_id      = var.vpc_id

  # No inbound rules are required for this function.
  # Add outbound rules or VPC endpoints as required by your network.

  tags = local.common_tags
}

# Hourly EventBridge rule
resource "aws_cloudwatch_event_rule" "hourly_check" {
  name                = "${var.project_name}-hourly"
  description         = "Run EC2 monitoring Lambda every hour"
  schedule_expression = var.schedule_expression

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "hourly_target" {
  rule      = aws_cloudwatch_event_rule.hourly_check.name
  target_id = "EC2MonitorLambda"
  arn       = aws_lambda_function.ec2_monitor.arn
}

# Reboot API event rule.
# Requires EC2 management events to be delivered by CloudTrail.
resource "aws_cloudwatch_event_rule" "reboot_event" {
  name        = "${var.project_name}-reboot"
  description = "Invoke Lambda when EC2 RebootInstances API is called"

  event_pattern = jsonencode({
    source        = ["aws.ec2"]
    "detail-type" = ["AWS API Call via CloudTrail"]

    detail = {
      eventSource = ["ec2.amazonaws.com"]
      eventName   = ["RebootInstances"]
    }
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "reboot_target" {
  rule      = aws_cloudwatch_event_rule.reboot_event.name
  target_id = "EC2RebootLambda"
  arn       = aws_lambda_function.ec2_monitor.arn
}
```

## 8. `lambda_function.py`

This is the Python code executed by Lambda. It supports three types of invocation:

- Hourly schedule: uses `TARGET_INSTANCE_ID`.
- Manual test: accepts an `instance_id` in the test event.
- Reboot event: extracts the instance ID from the EventBridge CloudTrail event.

python

```

import json
import logging
import os

import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(os.environ.get("LOG_LEVEL", "INFO").upper())

AWS_REGION = os.environ.get("AWS_REGION")
TARGET_INSTANCE_ID = os.environ.get("TARGET_INSTANCE_ID")
VPC_ID = os.environ.get("VPC_ID")
SUBNET_IDS = [
    subnet.strip()
    for subnet in os.environ.get("SUBNET_IDS", "").split(",")
    if subnet.strip()
]

ec2 = boto3.client("ec2", region_name=AWS_REGION)

def get_instance_id(event):
    """
    Resolve the instance ID from:
    1. A manual test event.
    2. An EC2 RebootInstances CloudTrail event.
    3. The TARGET_INSTANCE_ID environment variable.
    """

    # Manual test input: {"instance_id": "i-0123456789abcdef0"}
    if event.get("instance_id"):
        return event["instance_id"]

    # EventBridge event generated from CloudTrail
    detail = event.get("detail", {})

    if detail.get("eventName") == "RebootInstances":
        request_params = detail.get("requestParameters") or {}
        instance_set = request_params.get("instancesSet") or {}
        items = instance_set.get("items") or []

        if items:
            return items[0].get("instanceId")

    # Scheduled invocation or fallback
    return TARGET_INSTANCE_ID

def get_instance_details(instance_id):
    response = ec2.describe_instances(
        InstanceIds=[instance_id]
    )

    reservations = response.get("Reservations", [])

    for reservation in reservations:
        for instance in reservation.get("Instances", []):
            if instance["InstanceId"] == instance_id:
                state = instance.get("State", {})
                return {
                    "instance_id": instance["InstanceId"],
                    "instance_state": state.get("Name", "unknown"),
                    "instance_type": instance.get("InstanceType", "unknown"),
                    "availability_zone": instance.get(
                        "Placement", {}
                    ).get("AvailabilityZone"),
                    "private_ip": instance.get("PrivateIpAddress"),
                }

    raise ValueError(f"Instance {instance_id} was not found")

def get_instance_status(instance_id):
    response = ec2.describe_instance_status(
        InstanceIds=[instance_id],
        IncludeAllInstances=True,
    )

    statuses = response.get("InstanceStatuses", [])

    if not statuses:
        return {
            "instance_status": "not_available",
            "system_status": "not_available",
        }

    status = statuses[0]

    return {
        "instance_status": status.get(
            "InstanceStatus", {}
        ).get("Status", "unknown"),
        "system_status": status.get(
            "SystemStatus", {}
        ).get("Status", "unknown"),
    }

def lambda_handler(event, context):
    logger.info("Received event: %s", json.dumps(event))

    instance_id = get_instance_id(event)

    if not instance_id:
        raise ValueError(
            "No instance ID found. Set TARGET_INSTANCE_ID "
            "or provide instance_id in the event."
        )

    try:
        details = get_instance_details(instance_id)
        details.update(get_instance_status(instance_id))

        result = {
            "message": "EC2 instance details retrieved",
            "trigger": (
                "reboot_event"
                if event.get("detail", {}).get("eventName")
                == "RebootInstances"
                else "scheduled_or_manual"
            ),
            "vpc_id": VPC_ID,
            "subnet_ids": SUBNET_IDS,
            **details,
        }

        # Appears in CloudWatch Logs
        logger.info("EC2 details: %s", json.dumps(result))

        print(json.dumps(result))

        return {
            "statusCode": 200,
            "body": json.dumps(result),
        }

    except (ClientError, ValueError):
        logger.exception(
            "Failed to retrieve EC2 details for %s",
            instance_id,
        )
        raise
```

What the output looks like

json

```
{
  "message": "EC2 instance details retrieved",
  "trigger": "scheduled_or_manual",
  "vpc_id": "vpc-0123456789abcdef0",
  "subnet_ids": [
    "subnet-0123456789abcdef0",
    "subnet-abcdef01234567890"
  ],
  "instance_id": "i-0123456789abcdef0",
  "instance_state": "running",
  "instance_type": "t3.micro",
  "availability_zone": "us-east-1a",
  "private_ip": "10.0.1.25",
  "instance_status": "ok",
  "system_status": "ok"
}
```

`instance_state` is the EC2 state, while `instance_status` and `system_status` represent AWS health checks. Health checks may temporarily be unavailable for a newly started or stopped instance.

## 9. `main.tf`

Creates the CloudWatch log group with a retention policy so the Lambda logs do not grow indefinitely.

hcl

```

resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${var.project_name}-function"
  retention_in_days = 14

  tags = local.common_tags
}
```

For the log group to be created before the Lambda starts, add `aws_cloudwatch_log_group.lambda_logs` to the `depends_on` list in `aws_lambda_function.ec2_monitor` in `lambda.tf`.

## 10. `output.tf`

Displays useful deployment information after Terraform finishes.

hcl

```

output "lambda_function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.ec2_monitor.function_name
}

output "lambda_function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.ec2_monitor.arn
}

output "lambda_execution_role_arn" {
  description = "Lambda IAM execution role ARN"
  value       = aws_iam_role.lambda_execution_role.arn
}

output "vpc_id" {
  description = "VPC used by Lambda"
  value       = data.aws_vpc.selected.id
}

output "lambda_subnet_ids" {
  description = "Subnets configured for Lambda"
  value       = local.lambda_subnet_ids
}

output "hourly_event_rule_name" {
  description = "EventBridge rule for hourly execution"
  value       = aws_cloudwatch_event_rule.hourly_check.name
}

output "reboot_event_rule_name" {
  description = "EventBridge rule for EC2 reboot events"
  value       = aws_cloudwatch_event_rule.reboot_event.name
}

output "cloudwatch_log_group" {
  description = "Lambda CloudWatch log group"
  value       = aws_cloudwatch_log_group.lambda_logs.name
}
```

## 11. `terraform.tfvars`

Create this additional file to supply your actual AWS configuration. Replace the sample values with your own resources.

hcl

```

aws_region         = "us-east-1"
project_name       = "ec2-monitor"

vpc_id             = "vpc-0123456789abcdef0"
subnet_id_1        = "subnet-0123456789abcdef0"
subnet_id_2        = "subnet-abcdef01234567890"

target_instance_id = "i-0123456789abcdef0"

lambda_timeout     = 30
lambda_memory_size = 128

schedule_expression = "rate(1 hour)"
```

Keep `terraform.tfvars` out of source control if it contains environment-specific or sensitive configuration.

## 12. Deploy the Terraform configuration

Run these commands from the project directory.

bash

```

# 1. Initialize Terraform providers
terraform init

# 2. Validate Terraform syntax
terraform validate

# 3. Review the resources to be created
terraform plan -out=tfplan

# 4. Deploy the resources
terraform apply tfplan

# 5. Display the outputs
terraform output
```

## 13. Test the Lambda function

After deployment, test it manually with an EC2 instance ID.

Use the AWS CLI:

bash

```

aws lambda invoke \
  --function-name ec2-monitor-function \
  --cli-binary-format raw-in-base64-out \
  --payload '{"instance_id":"i-0123456789abcdef0"}' \
  response.json

cat response.json
```

Replace the example instance ID with an existing instance ID in the configured AWS region. The instance must be accessible to the Lambda execution role through EC2 API permissions.

You can also test from the AWS Console:

1. Open AWS Lambda and select `ec2-monitor-function`.
2. Open the Test tab.
3. Create a test event with this JSON:

json

```
{
  "instance_id": "i-0123456789abcdef0"
}
```

4. Click Test and review the response.
5. Open the function's Monitor → View CloudWatch logs to see the printed instance details.

## 14. Important AWS configuration requirements

Before relying on the automatic triggers, verify these details.

| Requirement | What to configure |
| --- | --- |
| VPC connectivity | Ensure the Lambda subnets can reach the regional EC2 API using a NAT gateway or an EC2 interface VPC endpoint. |
| Security group | Permit outbound HTTPS (TCP 443) to the chosen endpoint or network path. Configure the endpoint security group to allow inbound HTTPS from the Lambda security group when using an interface endpoint. |
| IAM permissions | The Lambda role already includes EC2 describe permissions and VPC execution permissions. |
| Hourly trigger | EventBridge invokes the Lambda every hour using `rate(1 hour)`. |
| Reboot trigger | Ensure CloudTrail management events are available in the region and the EventBridge rule matches the `RebootInstances` API event. |
| Logging | Lambda logs are retained for 14 days. |

Two important distinctions:

- The reboot rule detects a `RebootInstances` API call. It does not guarantee that the reboot completed successfully. If you need to track completed reboots or actual state changes, you will need a different monitoring approach.
- The hourly schedule checks only the single instance configured in `TARGET_INSTANCE_ID`. To monitor multiple instances, the Python code can be extended to accept a list of instance IDs from an environment variable.

This implementation gives you a modular Terraform deployment with a VPC-attached Lambda, an IAM execution role, scheduled monitoring, reboot-event triggering, environment-variable configuration, and CloudWatch logging.

IF your ECS service is already deployed through AWS CDK and your application image already exists in ECR, you don't need to rebuild your application image. You need to update the ECS task definition so CrowdStrike Falcon Container Sensor can protect your application containers.

The key detail is that the CrowdStrike ECS Fargate guide you linked uses a task-definition patching utility, rather than simply adding a normal sidecar container. The utility modifies the task definition and injects the sensor into your application containers.

GitHub

## Step-by-step approach

ECS Fargate 성능 측정하기 - 구축부터 Benchmark까지 (상) | dealicious-inc.github.io

1. Keep your existing application image

Continue using the ECR image you already deploy. You don't need to change your Dockerfile just to integrate the sensor.

CrowdStrike Falcon Endpoint Protection - Features, Reviews & Pricing (February 2026)

2. Make the Falcon sensor image available in ECR

Obtain the sensor image using your CrowdStrike Falcon credentials, then push it to an ECR repository your deployment can access. If it's already there, reuse it.

What is the AWS CDK? - AWS Cloud Development Kit (AWS CDK) v2

3. Patch the task definition

Use CrowdStrike's patching utility to create a modified ECS task definition JSON containing the Falcon initialization container and the necessary application-container changes.

New look for Amazon ECS in the AWS Management Console | Containers

4. Deploy a new task-definition revision

Update your CDK deployment workflow to register and deploy the modified task definition. Verify the service reaches a stable state and the sensor reports to Falcon.

Reference: [CrowdStrike's ECS Fargate implementation guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>). Note that this repository was archived in November 2024, so verify the current sensor image and supported deployment process with CrowdStrike before using it in production.

GitHub

## Script: patch and deploy your existing ECS task

The following example assumes ECS Fargate, AWS CLI, Docker, `jq`, and an existing ECS service. Run it from your deployment environment or CI/CD runner.

### 1. Set your variables

bash

```
export AWS_REGION="us-east-1"
export ECS_CLUSTER="my-ecs-cluster"
export ECS_SERVICE="my-ecs-service"

# Existing application task definition family
export TASK_FAMILY="my-app-task"

# CrowdStrike sensor image already pushed to ECR
export FALCON_IMAGE_URI="123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon-sensor:latest"

# Your CrowdStrike CID, including checksum
export FALCON_CID="YOUR_CID-WITH_CHECKSUM"
```

Replace these example values with your actual values. The sensor image URI must refer to a compatible Falcon Container Sensor image.

### 2. Export your current task definition

bash

```
aws ecs describe-task-definition \
  --task-definition "$TASK_FAMILY" \
  --region "$AWS_REGION" \
  --query taskDefinition \
  --output json |
jq 'del(
  .taskDefinitionArn,
  .revision,
  .status,
  .requiresAttributes,
  .compatibilities,
  .registeredAt,
  .registeredBy,
  .deregisteredAt
)' > taskdefinition.json
```

This creates `taskdefinition.json` from your current ECS task definition, including your existing application image and configuration.

### 3. Generate the ECR pull token

The patcher may need to inspect your private ECR application image to determine its entry point and command.

bash

```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account --output text)

export PULL_TOKEN=$(
  printf '{"auths":{"%s.dkr.ecr.%s.amazonaws.com":{"auth":"%s"}}}' \
    "$AWS_ACCOUNT_ID" \
    "$AWS_REGION" \
    "$(printf 'AWS:%s' "$(aws ecr get-login-password \
      --region "$AWS_REGION")" | base64 | tr -d '\n')" |
  base64 | tr -d '\n'
)
```

This example assumes the application images are in the same AWS account and region. For cross-account or cross-region ECR, adapt the authentication configuration accordingly.

### 4. Run the CrowdStrike patching utility

bash

```
docker run --rm \
  -v "$PWD:/var/run/spec" \
  "$FALCON_IMAGE_URI" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE_URI" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > taskdefinitionwithfalcon.json
```

The patcher produces `taskdefinitionwithfalcon.json`, adding the Falcon initialization container and the changes required to launch the sensor within your application containers.

GitHub

Important: If your task definition uses a custom entry point, command, private registry, or special container configuration, review the patched JSON before deployment.

### 5. Register the patched task definition

bash

```
aws ecs register-task-definition \
  --cli-input-json file://taskdefinitionwithfalcon.json \
  --region "$AWS_REGION"
```

Capture the new task-definition revision or ARN from the output.

### 6. Update your existing ECS service

After registration, deploy the new revision:

bash

```
export NEW_TASK_REVISION="my-app-task:12"

aws ecs update-service \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --task-definition "$NEW_TASK_REVISION" \
  --region "$AWS_REGION"

aws ecs wait services-stable \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION"
```

Replace `my-app-task:12` with the actual new revision returned by step 5.

### 7. Verify the deployment

bash

```
aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].{status:status,running:runningCount,desired:desiredCount,events:events[0:5]}'
```

Then verify the Falcon sensor's health and telemetry in your CrowdStrike Falcon console. A stable ECS service alone does not prove the sensor is reporting successfully.

## Important: integrate this with CDK

Because CDK manages your infrastructure, don't make the patched task definition a manual, permanent change. Your next CDK deployment may replace it with the unpatched definition.

A practical CI/CD flow is:

1. Build or select the existing application image.
2. Generate the ECS task-definition JSON from CDK.
3. Run the CrowdStrike patcher against that JSON.
4. Register the patched task definition.
5. Update the ECS service and verify deployment.

Keep the Falcon CID and any registry credentials in your secret-management system rather than hardcoding them.
--------------------------------------
For ECS Fargate + AWS CDK in TypeScript, I recommend integrating CrowdStrike into your deployment pipeline rather than adding a regular sidecar manually.

The important detail is that CrowdStrike's ECS Fargate guide injects the Falcon sensor into your existing application container using a patching utility. It also adds `SYS_PTRACE` and modifies the container entry point, so simply adding a second container in CDK is not equivalent.

GitHub

## Recommended deployment flow

Existing CDK task definition

Application image already in ECR

CrowdStrike patching utility

Inject Falcon sensor and update task JSON

Register patched task revision

Deploy to your existing ECS Fargate service

Verify ECS and Falcon

Confirm tasks are healthy and sensor telemetry arrives

One caveat: the linked CrowdStrike repository is archived and its guide is explicitly described as an unsupported open-source project. Validate the sensor version and deployment approach against your current CrowdStrike subscription before rolling it out.

GitHub

+1

## 1. Prepare the CrowdStrike sensor image

If the Falcon sensor image is already in your ECR, skip the image-publishing step.

Set these variables:

bash

```
export AWS_REGION="us-east-1"
export ECS_CLUSTER="my-ecs-cluster"
export ECS_SERVICE="my-ecs-service"

# ECR image containing the Falcon patching utility
export FALCON_IMAGE_URI="123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon-sensor/falcon-container:YOUR_TAG"

# CID including checksum from CrowdStrike Falcon
export FALCON_CID="YOUR_CID-WITH_CHECKSUM"
```

Use an actual versioned sensor tag instead of `YOUR_TAG`; avoid relying on `latest` for production deployments.

## 2. Create a deployment script

This script uses your existing ECS service and task definition, patches it, registers a new revision, and updates the service.

Save it as `deploy-with-falcon.sh`.

bash

```

#!/usr/bin/env bash
set -euo pipefail

: "${AWS_REGION:?Set AWS_REGION}"
: "${ECS_CLUSTER:?Set ECS_CLUSTER}"
: "${ECS_SERVICE:?Set ECS_SERVICE}"
: "${FALCON_IMAGE_URI:?Set FALCON_IMAGE_URI}"
: "${FALCON_CID:?Set FALCON_CID}"

WORKDIR="$(mktemp -d)"
trap 'rm -rf "$WORKDIR"' EXIT

echo "1. Getting current task definition..."

TASK_ARN=$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' \
  --output text)

aws ecs describe-task-definition \
  --task-definition "$TASK_ARN" \
  --region "$AWS_REGION" \
  --query taskDefinition \
  --output json |
jq 'del(
  .taskDefinitionArn,
  .revision,
  .status,
  .requiresAttributes,
  .compatibilities,
  .registeredAt,
  .registeredBy,
  .deregisteredAt
)' > "$WORKDIR/taskdefinition.json"

echo "2. Creating ECR pull token..."

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Base64-encode the ECR Docker auth JSON for the patcher.
AUTH=$(printf 'AWS:%s' \
  "$(aws ecr get-login-password --region "$AWS_REGION")" |
  base64 | tr -d '\n')

PULL_TOKEN=$(printf '{"auths":{"%s":{"auth":"%s"}}}' \
  "$REGISTRY" "$AUTH" | base64 | tr -d '\n')

echo "3. Patching the ECS task definition..."

docker run --rm \
  -v "$WORKDIR:/var/run/spec" \
  "$FALCON_IMAGE_URI" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE_URI" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > "$WORKDIR/taskdefinitionwithfalcon.json"

echo "4. Registering the patched task definition..."

NEW_TASK_ARN=$(aws ecs register-task-definition \
  --cli-input-json "file://$WORKDIR/taskdefinitionwithfalcon.json" \
  --region "$AWS_REGION" \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Registered: $NEW_TASK_ARN"

echo "5. Updating the ECS service..."

aws ecs update-service \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --task-definition "$NEW_TASK_ARN" \
  --region "$AWS_REGION" >/dev/null

echo "6. Waiting for ECS service stability..."

aws ecs wait services-stable \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION"

echo "Deployment completed. Task definition: $NEW_TASK_ARN"
```

The script follows the patcher's documented workflow. It assumes your application image and Falcon image are accessible through the configured ECR registry and that the patcher can inspect the application image. If your task uses multiple private registries or cross-account images, adjust the pull-token configuration.

GitHub

Run it with:

bash

```
chmod +x deploy-with-falcon.sh
./deploy-with-falcon.sh
```

Before running it, ensure your AWS identity has the required ECS permissions and ECR pull permissions.

## 3. Make the deployment persistent in CDK

There is one important CDK issue: the script registers a task definition outside CloudFormation. Your next `cdk deploy` can restore the task definition declared in your TypeScript code.

For a lasting implementation, choose one of these approaches:

- Quick test: Run the script against a non-production ECS service and validate Falcon injection.
- Production CI/CD: Add the patching step after CDK synthesis and ensure the patched task definition is the one deployed by your pipeline. Avoid letting a later CDK deployment silently replace it.
- Fully CDK-managed infrastructure: Implement a deployment integration that preserves the patched task definition and service revision, rather than just adding a normal sidecar.

## One thing to check before deploying

Your CDK code likely contains something similar to:

typescript

```
const taskDefinition = new ecs.FargateTaskDefinition(
  this,
  'AppTaskDefinition',
  {
    cpu: 512,
    memoryLimitMiB: 1024,
  },
);

taskDefinition.addContainer('AppContainer', {
  image: ecs.ContainerImage.fromRegistry(
    '123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest',
  ),
});
```

You generally do not need to change your application image. The key is to ensure the generated task definition is patched before the new revision is deployed.

Below is an example of how I would modify your existing AWS CDK TypeScript stack for ECS Fargate.

I'll assume your stack already creates a `FargateTaskDefinition`, adds your application container from ECR, and creates a `FargateService`.

Important: CrowdStrike's linked implementation uses a task-definition patching utility to inject Falcon into the application container. Adding Falcon as a normal sidecar in CDK is not equivalent. The CDK changes below preserve your existing service and image, while the deployment step applies the CrowdStrike patch before ECS registers the final task definition.

GitHub

## 1. Existing CDK stack — updated TypeScript

This example uses a separate deployment script to patch the task definition. Your CDK stack continues to own the task definition and service.

### `lib/ecs-stack.ts`

Use your existing application image, execution role, task role, networking, and load balancer configuration. The key change is to add `SYS_PTRACE` to the application container and export the task-definition ARN for the patching deployment step.

typescript

```

import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecr from 'aws-cdk-lib/aws-ecr';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class EcsStack extends cdk.Stack {
  constructor(
    scope: Construct,
    id: string,
    props?: cdk.StackProps,
  ) {
    super(scope, id, props);

    // Replace with your existing VPC, cluster, and repository.
    const vpc = ec2.Vpc.fromLookup(this, 'ExistingVpc', {
      vpcId: 'vpc-xxxxxxxx',
    });

    const cluster = new ecs.Cluster(this, 'ExistingCluster', {
      vpc,
      clusterName: 'my-ecs-cluster',
    });

    const repository = ecr.Repository.fromRepositoryName(
      this,
      'AppRepository',
      'my-app',
    );

    // Existing application task definition.
    const taskDefinition = new ecs.FargateTaskDefinition(
      this,
      'AppTaskDefinition',
      {
        family: 'my-app-task',
        cpu: 512,
        memoryLimitMiB: 1024,
      },
    );

    // Existing application image; no rebuild required.
    const appContainer = taskDefinition.addContainer(
      'AppContainer',
      {
        image: ecs.ContainerImage.fromEcrRepository(
          repository,
          'latest',
        ),
        logging: ecs.LogDrivers.awsLogs({
          streamPrefix: 'my-app',
        }),
      },
    );

    // Required by the CrowdStrike ECS Fargate patching approach.
    appContainer.addLinuxParameters(
      new ecs.LinuxParameters(this, 'AppLinuxParameters', {
        initProcessEnabled: true,
      }),
    );

    // Grant SYS_PTRACE to the application container.
    // The patcher also modifies the task definition.
    const cfnTaskDefinition =
      taskDefinition.node.defaultChild as ecs.CfnTaskDefinition;

    const containerDefinitions =
      cfnTaskDefinition.containerDefinitions;

    for (const container of containerDefinitions) {
      container.linuxParameters = {
        ...(container.linuxParameters ?? {}),
        capabilities: {
          ...(container.linuxParameters?.capabilities ?? {}),
          add: [
            ...(container.linuxParameters?.capabilities?.add ?? []),
            'SYS_PTRACE',
          ],
        },
      };
    }

    // Existing Fargate service.
    const service = new ecs.FargateService(this, 'AppService', {
      cluster,
      serviceName: 'my-ecs-service',
      taskDefinition,
      desiredCount: 2,
      assignPublicIp: false,
      platformVersion: ecs.FargatePlatformVersion.VERSION1_4,
    });

    new cdk.CfnOutput(this, 'TaskDefinitionArn', {
      value: taskDefinition.taskDefinitionArn,
    });

    new cdk.CfnOutput(this, 'ClusterName', {
      value: cluster.clusterName,
    });

    new cdk.CfnOutput(this, 'ServiceName', {
      value: service.serviceName,
    });
  }
}
```

Important: This CDK example prepares the task definition, but it does not inject the Falcon sensor by itself. Also, `initProcessEnabled` is optional and is not a substitute for the patcher. Keep your existing load balancer and service settings if you already have them.

## 2. Deploy using the CrowdStrike patcher

The safest way to test the integration is to synthesize the task definition, patch it, and register the patched revision.

The deployment pipeline should:

1. Run `cdk synth`.
2. Extract the `AWS::ECS::TaskDefinition` properties from the synthesized CloudFormation template.
3. Remove CloudFormation-only fields and save the properties as a task-definition JSON file.
4. Run the CrowdStrike patching utility against that JSON.
5. Register the patched task definition and update the ECS service to its new revision.

Refer to the [CrowdStrike ECS Fargate guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md) for the patcher's required arguments and image preparation.


One deployment caveat: If you register and deploy a patched task definition outside CDK, a future `cdk deploy` can restore the unpatched revision. For production, integrate the patching step into your release pipeline and ensure subsequent deployments preserve the sensor injection.

## With Terraform

For this use case, let's build a Terraform example in two stages:

1. Baseline: Deploy an ECS Fargate cluster, task definition, and service using your existing ECR application image, with no CrowdStrike changes.
2. Lower-environment test: Apply the CrowdStrike Falcon integration and deploy a new task-definition revision to a development or test service.

We'll keep the application image unchanged. The test will focus on the ECS task definition and deployment process.

One important distinction: the CrowdStrike repository you linked describes a patching approach that injects the sensor into the application container. We should test that documented workflow rather than assume a standard sidecar alone provides equivalent protection. The repository is archived, so validate compatibility with your current Falcon sensor version before production use. [CrowdStrike ECS Fargate guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>).

## Stage 1 — Terraform without CrowdStrike

Assumptions:

- AWS ECS Fargate.
- An existing ECR application image.
- An existing VPC with private subnets.
- A development environment with one running task.
- Terraform AWS provider.

### File structure

text

```
ecs-falcon-test/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

### `main.tf`

This creates an ECS cluster, IAM execution role, task definition, CloudWatch log group, and Fargate service. It deliberately does not include CrowdStrike.

hcl

```

# main.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_ecs_cluster" "main" {
  name = var.cluster_name

  tags = {
    Environment = var.environment
  }
}

resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.service_name}"
  retention_in_days = 7
}

data "aws_iam_policy_document" "ecs_tasks_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "execution" {
  name               = "${var.service_name}-execution-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_assume_role.json
}

resource "aws_iam_role_policy_attachment" "execution" {
  role       = aws_iam_role.execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "task" {
  name               = "${var.service_name}-task-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_assume_role.json
}

resource "aws_security_group" "app" {
  name   = "${var.service_name}-sg"
  vpc_id = var.vpc_id

  ingress {
    description = "Application traffic from the VPC"
    from_port   = var.app_port
    to_port     = var.app_port
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = var.service_name
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"
  memory                   = "1024"

  execution_role_arn = aws_iam_role.execution.arn
  task_role_arn      = aws_iam_role.task.arn

  container_definitions = jsonencode([
    {
      name      = "app"
      image     = var.app_image
      essential = true

      portMappings = [
        {
          containerPort = var.app_port
          protocol      = "tcp"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.app.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "app"
        }
      }
    }
  ])

  tags = {
    Environment = var.environment
  }

  depends_on = [aws_iam_role_policy_attachment.execution]
}

resource "aws_ecs_service" "app" {
  name            = var.service_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  platform_version = "1.4.0"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  depends_on = [aws_iam_role_policy_attachment.execution]
}
```

### `variables.tf`

hcl

```

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "cluster_name" {
  type    = string
  default = "demo-ecs-cluster"
}

variable "service_name" {
  type    = string
  default = "demo-app"
}

variable "app_image" {
  description = "Existing application image URI in ECR"
  type        = string
}

variable "app_port" {
  type    = number
  default = 8080
}

variable "vpc_id" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "private_subnet_ids" {
  type = list(string)
}
```

### `terraform.tfvars`

hcl

```
aws_region   = "us-east-1"
environment  = "dev"
cluster_name = "demo-ecs-cluster"
service_name = "demo-app"

app_image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1"

vpc_id             = "vpc-0123456789abcdef0"
vpc_cidr           = "10.0.0.0/16"
private_subnet_ids = ["subnet-aaa111", "subnet-bbb222"]

app_port = 8080
```

The VPC and private subnets must already exist. Because this service has no public IP, ensure the subnets have NAT access or the required VPC endpoints for ECR image pulls, CloudWatch Logs, and other required services.

### `outputs.tf`

hcl

```
output "ecs_cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "ecs_service_name" {
  value = aws_ecs_service.app.name
}

output "task_definition_arn" {
  value = aws_ecs_task_definition.app.arn
}
```

Deploy the baseline:

bash

```
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

This is the starting point against which you can compare task health, startup time, memory usage, and application behavior after enabling Falcon.

## Stage 2 — Add CrowdStrike for lower-environment testing

For this stage, use CrowdStrike's Terraform module, which creates the Falcon-enabled task definition, including the init container, sensor injection, and required Linux capability. This is a different integration from adding a standalone sidecar.

GitHub

+1

### Step 1: Prepare the sensor image and CID

Before applying Terraform:

- Obtain the Falcon Container sensor image through your CrowdStrike subscription.
- Push the compatible image to an ECR repository accessible to ECS.
- Obtain your Falcon CID, including checksum.
- Store the CID in a protected variable or secret-management system.

For example:

hcl

```
# dev.auto.tfvars — keep out of source control
falcon_image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon-sensor:PINNED_VERSION"
falcon_cid   = "YOUR_CID_WITH_CHECKSUM"
```

Add these variables to `variables.tf`:

hcl

```
variable "falcon_image" {
  type        = string
  description = "Version-pinned Falcon Container sensor image"
}

variable "falcon_cid" {
  type      = string
  sensitive = true
}
```

### Step 2: Replace the task-definition resource

In `main.tf`, remove `aws_ecs_task_definition.app` and replace it with this module:

hcl

```

module "falcon_ecs_task" {
  source = "github.com/crowdstrike/terraform-aws-ecs-fargate"

  app_name  = var.service_name
  app_image = var.app_image

  falcon_image = var.falcon_image
  falcon_cid   = var.falcon_cid

  task_cpu    = "512"
  task_memory = "1024"

  app_port_mappings = [
    {
      containerPort = var.app_port
      protocol      = "tcp"
    }
  ]

  # Reuse the baseline IAM roles.
  create_execution_role = false
  execution_role_arn    = aws_iam_role.execution.arn
  task_role_arn         = aws_iam_role.task.arn

  # Reuse the baseline CloudWatch log group.
  create_log_group = false
  log_group_name   = aws_cloudwatch_log_group.app.name

  platform_version = "1.4.0"

  tags = {
    Environment = var.environment
    Security    = "CrowdStrike-Falcon-Test"
  }
}
```

This uses the existing execution role, task role, log group, and application image. The module automatically adds the Falcon init container and `SYS_PTRACE` capability to the application container.

GitHub

+1

Entrypoint compatibility: Before testing, check how your application image starts. If it has a custom entrypoint or command, pass the appropriate `app_entrypoint` and `app_command` values to the module. Incorrect entrypoint handling can prevent your application from starting.

### Step 3: Change the ECS service to use the Falcon task definition

Replace the service's task-definition reference:

hcl

```
resource "aws_ecs_service" "app" {
  name            = var.service_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = module.falcon_ecs_task.task_definition_arn
  desired_count   = 1
  launch_type     = "FARGATE"

  platform_version = module.falcon_ecs_task.platform_version

  enable_execute_command = module.falcon_ecs_task.enable_execute_command

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  depends_on = [aws_iam_role_policy_attachment.execution]
}
```

If you later add a load balancer, use `module.falcon_ecs_task.container_name` as the application container name in its target-group configuration.

### Step 4: Apply in development

bash

```
terraform init
terraform fmt
terraform validate
terraform plan -out=falcon-dev.tfplan
terraform apply falcon-dev.tfplan
```

Review the plan carefully. Confirm it changes the intended development service and task definition, not your production resources.

### Step 5: Verify the test

bash

```
aws ecs describe-services \
  --cluster demo-ecs-cluster \
  --services demo-app \
  --region us-east-1 \
  --query 'services[0].{running:runningCount,desired:desiredCount,taskDefinition:taskDefinition,events:events[0:5]}'
```

Then inspect the running task's container list:

bash

```
TASK_ARN=$(aws ecs list-tasks \
  --cluster demo-ecs-cluster \
  --service-name demo-app \
  --region us-east-1 \
  --query 'taskArns[0]' \
  --output text)

aws ecs describe-tasks \
  --cluster demo-ecs-cluster \
  --tasks "$TASK_ARN" \
  --region us-east-1 \
  --query 'tasks[0].containers[*].{name:name,image:image,lastStatus:lastStatus,reason:reason}'
```

Check all three outcomes:

- The ECS service reaches steady state and the application passes its health checks.
- The task definition includes the Falcon initialization container and the application container starts successfully.
- The sensor appears in the CrowdStrike Falcon console and telemetry is received.

## Important testing notes

- The module's init container uses additional CPU and memory. The example's 512 CPU / 1024 MiB task allocation may need to be increased depending on your application load and the module's sensor resource requirements.
- Keep the Falcon image version pinned and validate the module's current version and compatibility before using it in production.
- If your existing task definition has additional containers, secrets, volumes, environment variables, or custom logging, migrate those settings into the module configuration as well. Otherwise, replacing the task definition could remove them.


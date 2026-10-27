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

## Using manual custom script

Below is a single interactive Bash script for testing CrowdStrike's archived ECS Fargate patching workflow against an existing NGINX service. It runs locally using Docker, backs up the current task definition, shows the proposed changes, and requires two separate deployment approvals.

Safety note: The script does not modify your ECS service until you approve the registration and service update. Because the CrowdStrike repository is archived and unsupported, use a disposable development service first. The local Falcon image must contain the repository's expected patcher and support its documented CLI arguments.

## Save as `install-crowdstrike-ecs.sh`

This version assumes the Falcon image is already present locally under the same ECR URI you enter, and that the application images are accessible through ECR or public registries. It builds the ECR pull token for private ECR images referenced by the task definition.

bash

```

#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

# CrowdStrike ECS Fargate task-definition patcher test.
# Use a non-production ECS service first.
# Requires: AWS CLI v2, Docker, jq, diff, base64.

die() {
  echo "ERROR: $*" >&2
  exit 1
}

confirm() {
  local answer
  echo
  read -r -p "$1 Type YES to continue: " answer
  [[ "$answer" == "YES" ]]
}

echo "=============================================="
echo " CrowdStrike ECS Fargate patcher - test only"
echo "=============================================="
echo "This modifies an ECS service only after approval."
echo "A timestamped backup will be retained locally."
echo

# 1. Prerequisites
for cmd in aws docker jq diff base64; do
  command -v "$cmd" >/dev/null 2>&1 ||
    die "Missing prerequisite: $cmd"
done

aws --version | grep -q 'aws-cli/2' ||
  die "AWS CLI v2 is required."

docker info >/dev/null 2>&1 ||
  die "Docker daemon is unavailable. Start Docker and retry."

# 2. Prompt for inputs
read -r -p "AWS region (e.g. us-east-1): " AWS_REGION
read -r -p "ECS cluster name or ARN: " ECS_CLUSTER
read -r -p "ECS service name or ARN: " ECS_SERVICE
read -r -p "Falcon CID (including checksum): " FALCON_CID
read -r -p "Falcon image ECR URI (with pinned tag): " FALCON_IMAGE_URI

[[ -n "$AWS_REGION" ]] || die "Region is required."
[[ -n "$ECS_CLUSTER" ]] || die "Cluster is required."
[[ -n "$ECS_SERVICE" ]] || die "Service is required."
[[ -n "$FALCON_CID" ]] || die "Falcon CID is required."
[[ -n "$FALCON_IMAGE_URI" ]] || die "Falcon image URI is required."

[[ "$FALCON_CID" != *" "* ]] ||
  die "CID must not contain spaces."

# Require a standard private ECR URI with a tag.
if [[ ! "$FALCON_IMAGE_URI" =~ ^[0-9]{12}\.dkr\.ecr\.[a-z0-9-]+\.amazonaws\.com/.+:[^/:]+$ ]]; then
  die "Falcon image must be a tagged private ECR URI."
fi

FALCON_REGISTRY="${FALCON_IMAGE_URI%%/*}"

echo
echo "Checking AWS identity..."
aws sts get-caller-identity --region "$AWS_REGION" ||
  die "Unable to authenticate to AWS."

echo
echo "Checking local Falcon image..."
docker image inspect "$FALCON_IMAGE_URI" >/dev/null 2>&1 ||
  die "Falcon image is not available locally under that exact URI. Pull or tag it first."

echo "Local Falcon image is available."

# Create a persistent, private backup directory.
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="./ecs-falcon-backup-${STAMP}"
mkdir -m 700 "$BACKUP_DIR"

echo "Backup directory: $BACKUP_DIR"

# 3. Retrieve existing service and task definition
echo
echo "Retrieving ECS service..."

SERVICE_JSON=$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --output json) ||
  die "Could not describe ECS service."

SERVICE_COUNT=$(jq '.services | length' <<<"$SERVICE_JSON")
[[ "$SERVICE_COUNT" == "1" ]] ||
  die "Service not found or multiple service matches returned."

SERVICE_STATUS=$(jq -r '.services[0].status' <<<"$SERVICE_JSON")
[[ "$SERVICE_STATUS" == "ACTIVE" ]] ||
  die "Service is not ACTIVE: $SERVICE_STATUS"

CURRENT_TASK_ARN=$(jq -r '.services[0].taskDefinition' <<<"$SERVICE_JSON")
[[ -n "$CURRENT_TASK_ARN" && "$CURRENT_TASK_ARN" != "null" ]] ||
  die "Service has no task definition."

echo "Current task definition: $CURRENT_TASK_ARN"

aws ecs describe-task-definition \
  --task-definition "$CURRENT_TASK_ARN" \
  --region "$AWS_REGION" \
  --output json |
jq '.taskDefinition' > "$BACKUP_DIR/original-full.json"

# 4. Remove AWS-generated fields that cannot be registered again.
jq 'del(
  .taskDefinitionArn,
  .requiresAttributes,
  .status,
  .revision,
  .compatibilities,
  .registeredAt,
  .registeredBy,
  .deregisteredAt,
  .tags
)' "$BACKUP_DIR/original-full.json" \
  > "$BACKUP_DIR/taskdefinition.json"

jq -e '
  (.family | type == "string") and
  (.containerDefinitions | type == "array" and length > 0) and
  (.requiresCompatibilities | index("FARGATE") != null)
' "$BACKUP_DIR/taskdefinition.json" >/dev/null ||
  die "Task definition is not a valid Fargate task definition."

# Verify that this is an NGINX service.
if ! jq -e '
  any(.containerDefinitions[];
    ((.name // "") + " " + (.image // "") | ascii_downcase)
    | contains("nginx"))
' "$BACKUP_DIR/taskdefinition.json" >/dev/null; then
  die "No NGINX container name or image found. Refusing to modify this service."
fi

# Refuse to patch a definition already containing the Falcon init container.
if jq -e '
  any(.containerDefinitions[];
    .name == "crowdstrike-falcon-init-container")
' "$BACKUP_DIR/taskdefinition.json" >/dev/null; then
  die "Falcon init container already exists. Refusing to inject it twice."
fi

echo
echo "Original task definition saved."
echo "Checking ECR image authentication..."

# 5. Build Docker config auth for ECR registries referenced by
# the task definition and Falcon image. The resulting token is
# passed only to the patcher and is not written to disk.
AUTH_CONFIG='{"auths":{}}'

mapfile -t ECR_REGISTRIES < <(
  {
    printf '%s\n' "$FALCON_IMAGE_URI"
    jq -r '.containerDefinitions[].image' \
      "$BACKUP_DIR/taskdefinition.json"
  } |
  while IFS= read -r image; do
    registry="${image%%/*}"
    if [[ "$registry" =~ ^[0-9]{12}\.dkr\.ecr\.([a-z0-9-]+)\.amazonaws\.com$ ]]; then
      printf '%s\n' "$registry"
    fi
  done |
  sort -u
)

((${#ECR_REGISTRIES[@]} > 0)) ||
  die "No private ECR registries found."

for registry in "${ECR_REGISTRIES[@]}"; do
  registry_region="${registry#*.dkr.ecr.}"
  registry_region="${registry_region%.amazonaws.com}"

  echo "Obtaining ECR authorization for $registry"

  PASSWORD=$(aws ecr get-login-password \
    --region "$registry_region") ||
    die "Cannot obtain ECR credentials for $registry"

  AUTH=$(printf 'AWS:%s' "$PASSWORD" |
    base64 | tr -d '\r\n')
  unset PASSWORD

  AUTH_CONFIG=$(jq -c \
    --arg registry "$registry" \
    --arg auth "$AUTH" \
    '.auths[$registry] = {"auth": $auth}' \
    <<<"$AUTH_CONFIG")

  unset AUTH
done

PULL_TOKEN=$(printf '%s' "$AUTH_CONFIG" |
  base64 | tr -d '\r\n')
unset AUTH_CONFIG

# 6. Run patcher using the locally available Falcon image.
echo
echo "Running CrowdStrike patcher..."

if ! docker run --rm --pull=never \
  -v "$PWD/$BACKUP_DIR:/var/run/spec" \
  "$FALCON_IMAGE_URI" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE_URI" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > "$BACKUP_DIR/taskdefinitionwithfalcon.json"; then
  unset PULL_TOKEN
  die "CrowdStrike patcher failed. Original definition is preserved."
fi

unset PULL_TOKEN

PATCHED="$BACKUP_DIR/taskdefinitionwithfalcon.json"

# 7. Validate patched JSON
jq -e '
  (.family | type == "string") and
  (.containerDefinitions | type == "array" and length > 0) and
  (.requiresCompatibilities | index("FARGATE") != null) and
  any(.containerDefinitions[];
    .name == "crowdstrike-falcon-init-container")
' "$PATCHED" >/dev/null ||
  die "Patched JSON is invalid or Falcon init container is missing."

jq -e '
  all(.containerDefinitions[];
    ((.linuxParameters.capabilities.add // []) | index("SYS_PTRACE")) != null
  )
' "$PATCHED" >/dev/null ||
  die "SYS_PTRACE is not present on every patched container."

if jq -e '
  any(.containerDefinitions[];
    ((.name // "") + " " + (.image // "") | ascii_downcase)
    | contains("nginx"))
' "$PATCHED" >/dev/null; then
  echo "NGINX container found in patched definition."
else
  die "NGINX container missing from patched definition."
fi

echo
echo "Patched task definition validated."
echo "Review the full diff below:"
echo

diff -u \
  "$BACKUP_DIR/taskdefinition.json" \
  "$PATCHED" || true

echo
echo "Backup files:"
echo "  Original full: $BACKUP_DIR/original-full.json"
echo "  Clean input:   $BACKUP_DIR/taskdefinition.json"
echo "  Patched JSON:  $PATCHED"

# 8. Explicit approval before registering the new revision.
echo
echo "Cluster: $ECS_CLUSTER"
echo "Service: $ECS_SERVICE"
echo "Current task definition: $CURRENT_TASK_ARN"
echo "Falcon image: $FALCON_IMAGE_URI"

confirm "Register the patched task definition?" ||
  die "Registration cancelled. ECS was not changed."

NEW_TASK_ARN=$(aws ecs register-task-definition \
  --cli-input-json "file://$PATCHED" \
  --region "$AWS_REGION" \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text) ||
  die "Task definition registration failed."

echo
echo "Registered new task definition:"
echo "$NEW_TASK_ARN"

# 9. Explicit approval before changing the service.
if ! confirm "Update the ECS service to use this new revision?"; then
  echo "Service was not updated."
  echo "Registered revision remains available: $NEW_TASK_ARN"
  exit 0
fi

# Re-check that the service still points to the original revision.
LATEST_TASK_ARN=$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' \
  --output text)

[[ "$LATEST_TASK_ARN" == "$CURRENT_TASK_ARN" ]] ||
  die "Service task definition changed since inspection. Refusing to overwrite it."

aws ecs update-service \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --task-definition "$NEW_TASK_ARN" \
  --region "$AWS_REGION" >/dev/null ||
  die "ECS service update failed."

# 10. Wait for stability and validate running NGINX task.
echo
echo "Waiting for ECS service stability..."

aws ecs wait services-stable \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" ||
  die "Service did not stabilize. Check ECS events and stopped task reasons."

FINAL_SERVICE=$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --output json)

FINAL_TASK_ARN=$(jq -r '.services[0].taskDefinition' <<<"$FINAL_SERVICE")
RUNNING=$(jq -r '.services[0].runningCount' <<<"$FINAL_SERVICE")
DESIRED=$(jq -r '.services[0].desiredCount' <<<"$FINAL_SERVICE")

[[ "$FINAL_TASK_ARN" == "$NEW_TASK_ARN" ]] ||
  die "Service is not using the registered Falcon task definition."

[[ "$RUNNING" -gt 0 && "$RUNNING" == "$DESIRED" ]] ||
  die "Service running count does not match desired count."

mapfile -t TASK_ARNS < <(
  aws ecs list-tasks \
    --cluster "$ECS_CLUSTER" \
    --service-name "$ECS_SERVICE" \
    --desired-status RUNNING \
    --region "$AWS_REGION" \
    --query 'taskArns[]' \
    --output text |
  tr '\t' '\n' |
  grep '^arn:' || true
)

((${#TASK_ARNS[@]} > 0)) ||
  die "No running service tasks found."

TASK_JSON=$(aws ecs describe-tasks \
  --cluster "$ECS_CLUSTER" \
  --tasks "${TASK_ARNS[@]}" \
  --region "$AWS_REGION" \
  --output json)

jq -e '
  (.failures | length) == 0 and
  any(.tasks[];
    .lastStatus == "RUNNING" and
    any(.containers[];
      ((.name // "") + " " + (.image // "") | ascii_downcase)
      | contains("nginx")
    ) and
    any(.containers[];
      .name == "crowdstrike-falcon-init-container"
    )
  )
' <<<"$TASK_JSON" >/dev/null ||
  die "Running task validation failed. Inspect task containers and ECS events."

echo
echo "=============================================="
echo " SUCCESS: ECS Fargate test deployment"
echo "=============================================="
echo "Cluster:          $ECS_CLUSTER"
echo "Service:          $ECS_SERVICE"
echo "Task definition:   $NEW_TASK_ARN"
echo "Running tasks:     $RUNNING / $DESIRED"
echo "Backup directory:  $BACKUP_DIR"
echo
echo "Confirm sensor health and telemetry in Falcon."
echo "ECS stability alone does not prove sensor reporting."
```

## Run it

Save the script, then execute:

bash

```
chmod 700 install-crowdstrike-ecs.sh
./install-crowdstrike-ecs.sh
```

Before approving deployment, review the generated diff. The patcher should add the `crowdstrike-falcon-init-container`, adjust application container entry points, and add `SYS_PTRACE`. 


### Important limitations

- This script handles private ECR registries in the standard AWS commercial partition. It assumes the current AWS identity has permission to pull every private ECR image referenced in the task definition.
- The Falcon image must be tagged locally with exactly the ECR URI you enter. `--pull=never` prevents Docker from silently downloading a different image.
- If NGINX uses a custom command or entrypoint, inspect the patched JSON carefully before approval.
- If deployment fails, inspect ECS service events and stopped-task reasons. A successful registration does not guarantee that Fargate can start the patched task.
- The script intentionally leaves the registered revision in place if you decline the second approval. It does not automatically roll back a deployment after an application or sensor health failure.
- The final validation confirms ECS container state, not successful Falcon telemetry. Check the Falcon console to verify the sensor is actually reporting.

## Key concepts

Let's break this down from the beginning, assuming you're new to ECS. The goal is to understand what the script does, why each step is necessary, and how CrowdStrike gets added to an existing NGINX application without rebuilding the NGINX image.

# 1. First, understand what we're changing

Imagine you already have an NGINX web server running in AWS. Your application is packaged as a Docker image and stored in ECR. ECS Fargate runs that image for you.

## Before CrowdStrike

ECR — Container image registry

Stores your NGINX Docker image

ECS task definition

Instructions for running NGINX

ECS Fargate task

NGINX container

Your application runs here

After the CrowdStrike integration, the original NGINX image remains in ECR. The task definition is modified so that the Falcon sensor can be injected when the task starts.

## After CrowdStrike patching

ECR

Existing NGINX image + Falcon sensor image

Patched ECS task definition

NGINX configuration + Falcon initialization container

ECS Fargate task

Falcon initialization container

Prepares the sensor injection

NGINX container

Runs with the injected sensor setup

The Falcon initialization container is not necessarily a permanently running monitoring sidecar. In the documented patching approach, it prepares the application container for sensor injection.

# 2. Key ECS terms in simple English

| Term | Simple meaning |
| --- | --- |
| Docker image | A packaged application, like NGINX, ready to run. |
| Container | A running instance of an image. |
| ECR | AWS storage for Docker images. Think of it as a container image library. |
| ECS | AWS service that manages your containers. |
| Fargate | AWS runs the underlying compute infrastructure, so you don't manage EC2 servers. |
| ECS cluster | A logical group where your ECS services and tasks run. |
| Task definition | A recipe describing the images, CPU, memory, ports, roles, and settings needed to run a task. |
| Task | One running instance of a task definition. |
| Service | Keeps the desired number of tasks running and replaces unhealthy tasks. |
| Task revision | A numbered version of a task definition, such as `nginx-app:4` or `nginx-app:5`. |
| Falcon CID | Your CrowdStrike customer identifier, generally including the checksum supplied by Falcon. |
| Patcher | A utility that modifies the task-definition JSON to prepare it for Falcon sensor injection. |
| IAM role | Permissions that allow AWS services or containers to access other AWS resources. |
| Deployment approval | Your confirmation before a change is registered or deployed. |

The most important distinction to remember is:

ECR stores the image. The task definition describes how to run it. The service keeps the desired number of tasks running.

# 3. The complete workflow

Here is the process your script follows, from your laptop to AWS.

1. Check your laptop
   The script checks that AWS CLI, Docker, and `jq` are installed. Docker must be running.
2. Ask for configuration
   You enter the AWS region, ECS cluster, ECS service, Falcon CID, and Falcon image URI.
3. Verify AWS access
   The script calls AWS STS to check which AWS identity you are using and confirms that you can authenticate.
4. Download the current task definition
   It asks ECS which task definition the existing service uses, then retrieves that definition as JSON.
5. Create a backup
   It saves the original definition before making any changes.
6. Run the CrowdStrike patcher locally
   Docker runs the Falcon patcher image against the task-definition JSON. The patcher produces a modified JSON file.
7. Validate and review
   The script checks the JSON and displays a diff showing the differences between the original and patched definitions.
8. Approve task registration
   If you type `YES`, AWS registers the patched task definition as a new revision.
9. Approve service deployment
   If you type `YES` again, the ECS service is updated to use the new revision.
10. Wait and verify
    The script waits for ECS to stabilize, checks the running task, and verifies that the expected containers appear.

The two approvals serve different purposes: the first creates a new task-definition revision in AWS; the second actually changes the task definition used by your service.

# 4. Explaining the script section by section

## Step 1 — Check prerequisites

bash

```
for cmd in aws docker jq diff base64; do
  command -v "$cmd" >/dev/null 2>&1 ||
    die "Missing prerequisite: $cmd"
done
```

This checks whether the required command-line tools exist.

- `aws`: communicates with AWS.
- `docker`: runs the Falcon patcher image.
- `jq`: reads and modifies JSON.
- `diff`: compares the original and patched files.
- `base64`: encodes ECR authentication data.

If a tool is missing, the script stops instead of failing halfway through the deployment.

## Step 2 — Ask for input

bash

```
read -r -p "AWS region: " AWS_REGION
read -r -p "ECS cluster: " ECS_CLUSTER
read -r -p "ECS service: " ECS_SERVICE
```

These values tell the script which AWS resources to work with.

For example:

text

```
AWS region: us-east-1
ECS cluster: dev-cluster
ECS service: nginx-service
Falcon CID: your-CID-with-checksum
Falcon image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon:version
```

The script doesn't create a new cluster or service. It targets the existing service you specify.

## Step 3 — Check AWS identity

bash

```
aws sts get-caller-identity --region "$AWS_REGION"
```

AWS STS tells you which account and IAM identity your CLI is using.

This is important because you might have access to several AWS accounts, such as development, staging, and production.

Successful authentication doesn't automatically mean you have all the permissions needed. You also need the appropriate ECS and ECR permissions.

## Step 4 — Retrieve the task definition

bash

```
CURRENT_TASK_ARN=$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' \
  --output text)
```

Think of the service as saying:

> "I am currently using task definition revision 7."

The script retrieves that task definition and saves its configuration.

It then removes AWS-generated fields:

bash

```
jq 'del(
  .taskDefinitionArn,
  .requiresAttributes,
  .status,
  .revision,
  .compatibilities,
  .registeredAt,
  .registeredBy,
  .deregisteredAt,
  .tags
)' original-full.json
```

Why?

When you retrieve a task definition from AWS, the response includes metadata AWS generated when it registered the definition. Some of these fields must not be sent back as input when registering a new revision.

The script removes that metadata while retaining the application configuration.

## Step 5 — Create the backup

The script creates a timestamped directory such as:

text

```
ecs-falcon-backup-20261027-153000/
├── original-full.json
├── taskdefinition.json
└── taskdefinitionwithfalcon.json
```

Each file has a purpose:

- `original-full.json`: the complete response retrieved from AWS.
- `taskdefinition.json`: the cleaned definition that will be passed to the patcher.
- `taskdefinitionwithfalcon.json`: the patched output.

This lets you inspect the original configuration and understand exactly what changed.

## Step 6 — Authenticate to ECR

ECR repositories can be private, meaning Docker needs authentication to pull images.

bash

```
aws ecr get-login-password --region "$registry_region"
```

This requests an ECR authorization token using your AWS credentials.

The script encodes the authentication data and passes it to the patcher.

Important: The script's pull-token handling assumes the patcher's documented `-pulltoken` option accepts the encoded Docker authentication configuration. Verify this against the exact patcher version you use. The token is temporary, but it should still be treated as a secret.

## Step 7 — Run the patcher

The central command is:

bash

```
docker run --rm --pull=never \
  -v "$PWD/$BACKUP_DIR:/var/run/spec" \
  "$FALCON_IMAGE_URI" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE_URI" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > "$BACKUP_DIR/taskdefinitionwithfalcon.json"
```

Let's break it down.

| Command or option | Meaning |
| --- | --- |
| `docker run` | Starts a container to execute the patcher. |
| `--rm` | Removes the patcher container after it exits. |
| `--pull=never` | Uses the image already present locally instead of pulling it. |
| `-v` | Makes your local backup directory accessible inside the patcher container. |
| `-cid` | Supplies your Falcon customer identifier. |
| `-image` | Identifies the Falcon sensor image used by the patcher. |
| `-pulltoken` | Supplies authentication data so the patcher can inspect private images. |
| `-ecs-spec-file` | Points the patcher at the original task-definition JSON. |
| `>` | Saves the patcher's output to a new JSON file. |

Notice that the script runs the patcher on your laptop. It does not immediately change the ECS service.

The patcher is preparing the task definition, not directly installing software on your laptop or rebuilding NGINX.

## Step 8 — Validate the patched definition

The script checks whether the output is valid JSON, whether it still describes a Fargate task definition, and whether the expected Falcon initialization container exists.

It also checks for `SYS_PTRACE`.

In simple terms, `SYS_PTRACE` is a Linux capability related to process inspection and debugging. The documented sensor-injection method requires it for the relevant application container configuration.

These checks are useful, but they don't prove the sensor will start successfully. That's why the script also performs deployment validation.

## Step 9 — Review the diff

bash

```
diff -u original.json taskdefinitionwithfalcon.json
```

A diff shows what changed between two files.

For example, the output might show:

diff

```
+ "name": "crowdstrike-falcon-init-container"
```

or a change to the NGINX container's entry point.

The exact output depends on your task definition and the patcher version.

Review this output carefully. If the patcher unexpectedly removes environment variables, secrets, port mappings, health checks, or other settings, do not proceed until you understand why.

## Step 10 — Register the new revision

bash

```
aws ecs register-task-definition \
  --cli-input-json file://taskdefinitionwithfalcon.json \
  --region "$AWS_REGION"
```

AWS registers the patched JSON as a new task-definition revision.

For example:

text

```
Before: nginx-app:7
After:  nginx-app:8
```

Revision 8 exists in ECS, but the service is still using revision 7 at this point.

This is an important safety boundary.

## Step 11 — Update the ECS service

bash

```
aws ecs update-service \
  --cluster "$ECS_CLUSTER" \
  --service "$ECS_SERVICE" \
  --task-definition "$NEW_TASK_ARN" \
  --region "$AWS_REGION"
```

This tells ECS:

> "Use the new task definition for this service."

ECS then starts replacing old tasks with tasks created from the new revision, following its deployment configuration.

Your existing service name and cluster remain the same.

## Step 12 — Wait for stability and validate

bash

```
aws ecs wait services-stable \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE" \
  --region "$AWS_REGION"
```

The script waits for ECS to report the service as stable.

It then checks the running tasks and looks for NGINX and the Falcon initialization container.

Remember, a stable ECS service does not prove CrowdStrike is reporting telemetry. You must separately confirm sensor health in the Falcon console.

# 5. What happens if something fails?

| Failure | What it means | What to check |
| --- | --- | --- |
| AWS identity check fails | Your CLI can't authenticate. | AWS profile, credentials, SSO session, region. |
| Falcon image not found locally | Docker can't find the exact image tag. | Run `docker image ls` and check the image URI. |
| Patcher fails | The patcher couldn't process the task definition or access an image. | Docker output, pull token, image compatibility, patcher arguments. |
| Registration fails | AWS rejected the task-definition JSON or your permissions. | AWS error message and JSON validation. |
| Service deployment fails | ECS couldn't successfully deploy the new revision. | ECS service events, stopped task reasons, CloudWatch logs. |
| Service is stable but Falcon is absent | ECS is running, but the sensor may not be healthy or reporting. | Sensor initialization logs and the Falcon console. |

The script deliberately does not automatically roll back a failed deployment. If a new revision is deployed and the application fails, you should inspect the service events and explicitly restore the previous task-definition revision if needed.

# 6. Five concepts to remember

- The patcher runs locally. It creates a modified task-definition JSON file.
- Registering a revision is not the same as deploying it. The service changes only when you update it.
- ECR images and task definitions are different things. This workflow changes the task definition, not the NGINX image itself.
- The backup is your reference point. Always keep it and review the diff.
- ECS health and Falcon health are different. Confirm both the application and the sensor are working.

One final caveat: the script is a useful test harness, but the archived CrowdStrike patcher is unsupported. In particular, the validation logic and authentication options should be checked against the exact version you intend to run before using it on even a shared development service.

## Operational Tasks

# CrowdStrike on ECS Fargate: operational guide

Once the initial test works, the next step is understanding what CrowdStrike adds to your ECS task definition, how to maintain the sensor over time, and how to verify it is actually working inside a running container.

I'll organize this into four parts:

1. New CrowdStrike terminology and task-definition changes.
2. Operational maintenance responsibilities and update frequency.
3. A script to automate image and patcher maintenance checks.
4. Validation of a running ECS task.

The key principle is that maintaining the Falcon sensor image, maintaining the patching utility, and deploying a new ECS task definition are three separate activities. Updating one does not automatically update the others.

## 1. New CrowdStrike terminology

These terms come from the archived [CrowdStrike ECS Fargate guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>). The guide says the sensor runs inside each application container, using user-space instrumentation rather than a kernel sensor on the Fargate host. It also explicitly states that the open-source implementation is unsupported.

GitHub

+1

| Term | What it means operationally |
| --- | --- |
| `-ecs-spec-file` | The path to the input ECS task-definition JSON file inside the patcher container. |
| `-ecs-spec` | An alternative patcher input option for passing a task definition through standard input. |
| `-cid` | The Falcon customer ID, including the checksum where required. It tells the sensor which Falcon customer environment to report to. |
| `-image` | The sensor image reference that the patched task definition will use. |
| `-pulltoken` | Temporary registry authentication data used by the patcher to inspect private application images and discover their entry points or commands. |
| `SYS_PTRACE` | A Linux capability the documented approach adds to application containers so the sensor can inspect processes. |
| Falcon initialization container | A container that prepares the sensor files and shared storage before the application starts. |
| Sensor injection | The process of making the sensor run inside the application container without rebuilding the original application image. |
| Falcon sensor image | The container image containing the sensor and the patching utility. |
| Falcon registry credentials | Credentials used to authenticate to CrowdStrike's private container registry and obtain sensor images. |

### What exactly is `-ecs-spec-file`?

Imagine your existing task definition is saved on your laptop:

text

```
taskdefinition.json
```

The script mounts the local directory into Docker:

bash

```
-v "$PWD:/var/run/spec"
```

Inside the patcher container, the file is accessible at:

text

```
/var/run/spec/taskdefinition.json
```

Then this argument tells the patcher where to find it:

bash

```
-ecs-spec-file /var/run/spec/taskdefinition.json
```

The patcher reads that file and generates a modified task definition. It doesn't directly edit your ECS service.

### What is `SYS_PTRACE`?

Think of it as a permission that allows the sensor to inspect processes inside the application container.

The patcher adds this capability to the relevant container definitions. It is a security-sensitive permission, so you should review the change and follow your organization's container hardening policy.

### What changes should you see in the patched task definition?

The exact JSON depends on your existing task definition and patcher version. The documented approach generally changes these areas:

- An additional initialization container: `crowdstrike-falcon-init-container`.
- Shared storage: a volume and mounts used to make the sensor files available to application containers.
- Application entry points: the sensor entry point wraps the original application startup sequence.
- Linux capabilities: `SYS_PTRACE` is added to application containers.
- Sensor image reference and CID: the initialization configuration identifies the sensor image and customer.

One important detail: the initialization container may exit after preparing the sensor. Don't expect it to be a permanently running monitoring container. The sensor itself runs inside each instrumented application container.

GitHub

## 2. What needs regular maintenance?

There are four separate things to manage.

A. Falcon sensor image

Monitor vendor releasesCheck for new supported sensor releases, security fixes, and compatibility changes. Pull the approved version into ECR, test it, and deploy a new task-definition revision when appropriate.

B. Patching utility

Review with every sensor upgradeThe patcher is bundled in the sensor image in the documented approach. Updating the image used to run the patcher also updates the patcher version used for that run. Keep a record of which patcher/sensor version generated each deployed task definition.

C. Registry credentials and tokens

Refresh for each runECR authorization tokens are short-lived. Generate fresh tokens each time the patcher needs private image access. Keep CrowdStrike API credentials in a secret manager, not in the script or source control.

D. Running tasks and sensor health

After every deploymentCheck ECS service stability, container startup, logs, and sensor telemetry in Falcon. Also perform periodic checks that running tasks are still using the expected task-definition revision.

These are recommended operational practices, not official CrowdStrike maintenance intervals.

### Suggested maintenance schedule

| Frequency | Activity |
| --- | --- |
| Each deployment | Validate the patched JSON and review its diff. |
| After each sensor upgrade | Test NGINX startup, ECS health, and Falcon telemetry. |
| Weekly | Check for vendor advisories, new sensor releases, and failed ECS tasks. |
| Monthly | Review deployed sensor versions, ECR image retention, IAM access, and the rollback procedure. |
| After a critical advisory | Assess urgency and deploy the approved fix according to your incident/change process. |

The patcher's ECR pull token needs to be refreshed much more frequently than the sensor image itself. The archived guide specifically warns that ECR credentials expire and need periodic renewal.

GitHub

## 3. Operational automation script

This script is intended for routine sensor upgrades in a lower environment. It:

1. Uses an explicitly selected, version-pinned Falcon image already mirrored into ECR and available locally.
2. Retrieves the existing ECS task definition and backs it up.
3. Runs the patcher with fresh ECR credentials.
4. Shows the diff and asks for approval.
5. Registers a new task-definition revision and asks for deployment approval.
6. Waits for service stability.
7. Uses ECS Exec to inspect the sensor in a running NGINX container.

It deliberately does not choose an unreviewed latest sensor release automatically. Your team should approve the sensor version before using it.

### `maintain-falcon-ecs.sh`

bash

```

#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

die() { echo "ERROR: $*" >&2; exit 1; }

approve() {
  local answer
  read -r -p "$1 Type YES: " answer
  [[ "$answer" == "YES" ]]
}

for cmd in aws docker jq diff base64; do
  command -v "$cmd" >/dev/null || die "Install $cmd first"
done

docker info >/dev/null 2>&1 || die "Docker is not running"

read -r -p "AWS region: " AWS_REGION
read -r -p "ECS cluster: " CLUSTER
read -r -p "ECS service: " SERVICE
read -r -p "Approved Falcon image URI (version-pinned ECR URI): " FALCON_IMAGE
read -r -p "Falcon CID: " FALCON_CID

[[ -n "$AWS_REGION" && -n "$CLUSTER" &&
   -n "$SERVICE" && -n "$FALCON_IMAGE" &&
   -n "$FALCON_CID" ]] || die "All values are required"

docker image inspect "$FALCON_IMAGE" >/dev/null 2>&1 ||
  die "Approved Falcon image is not present locally"

aws sts get-caller-identity --region "$AWS_REGION" ||
  die "AWS authentication failed"

STAMP=$(date +%Y%m%d-%H%M%S)
WORKDIR="./falcon-maintenance-$STAMP"
mkdir -m 700 "$WORKDIR"

echo "Retrieving current service..."
aws ecs describe-services \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --region "$AWS_REGION" > "$WORKDIR/service.json"

[[ "$(jq -r '.services | length' "$WORKDIR/service.json")" == "1" ]] ||
  die "Service not found"

[[ "$(jq -r '.services[0].status' "$WORKDIR/service.json")" == "ACTIVE" ]] ||
  die "Service is not ACTIVE"

OLD_TASK=$(jq -r '.services[0].taskDefinition' "$WORKDIR/service.json")

aws ecs describe-task-definition \
  --task-definition "$OLD_TASK" \
  --region "$AWS_REGION" \
  --query taskDefinition > "$WORKDIR/original-full.json"

jq 'del(
  .taskDefinitionArn, .revision, .status,
  .requiresAttributes, .compatibilities,
  .registeredAt, .registeredBy, .deregisteredAt,
  .tags
)' "$WORKDIR/original-full.json" > "$WORKDIR/input.json"

jq -e '
  (.requiresCompatibilities | index("FARGATE") != null) and
  any(.containerDefinitions[];
    ((.name // "") + " " + (.image // "") | ascii_downcase)
    | contains("nginx"))
' "$WORKDIR/input.json" >/dev/null ||
  die "Expected an NGINX Fargate task definition"

jq -e '
  any(.containerDefinitions[];
    .name == "crowdstrike-falcon-init-container")
' "$WORKDIR/input.json" >/dev/null &&
  die "Task definition already appears patched; review before proceeding"

# Build an ECR pull token for all private ECR registries referenced
# by this task definition and the Falcon image.
mapfile -t REGISTRIES < <(
  {
    printf '%s\n' "$FALCON_IMAGE"
    jq -r '.containerDefinitions[].image' "$WORKDIR/input.json"
  } |
  while IFS= read -r image; do
    registry="${image%%/*}"
    [[ "$registry" =~ ^[0-9]{12}\.dkr\.ecr\.[a-z0-9-]+\.amazonaws\.com$ ]] &&
      printf '%s\n' "$registry"
  done | sort -u
)

AUTH_CONFIG='{"auths":{}}'

for registry in "${REGISTRIES[@]}"; do
  registry_region="${registry#*.dkr.ecr.}"
  registry_region="${registry_region%.amazonaws.com}"

  password=$(aws ecr get-login-password --region "$registry_region") ||
    die "Cannot get ECR token for $registry"

  auth=$(printf 'AWS:%s' "$password" | base64 | tr -d '\r\n')
  unset password

  AUTH_CONFIG=$(jq -c \
    --arg registry "$registry" \
    --arg auth "$auth" \
    '.auths[$registry] = {"auth": $auth}' <<<"$AUTH_CONFIG")
  unset auth
done

PULL_TOKEN=$(printf '%s' "$AUTH_CONFIG" | base64 | tr -d '\r\n')
unset AUTH_CONFIG

echo "Running the patcher..."
docker run --rm --pull=never \
  -v "$PWD/$WORKDIR:/var/run/spec" \
  "$FALCON_IMAGE" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/input.json \
  > "$WORKDIR/patched.json"

unset PULL_TOKEN

jq -e '
  (.containerDefinitions | type == "array") and
  any(.containerDefinitions[];
    .name == "crowdstrike-falcon-init-container")
' "$WORKDIR/patched.json" >/dev/null ||
  die "Patched definition failed validation"

echo
echo "Changes to review:"
diff -u "$WORKDIR/input.json" "$WORKDIR/patched.json" || true

echo
echo "Backup: $WORKDIR"
approve "Register the patched task definition?" ||
  die "Cancelled before registration"

NEW_TASK=$(aws ecs register-task-definition \
  --cli-input-json "file://$WORKDIR/patched.json" \
  --region "$AWS_REGION" \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Registered revision: $NEW_TASK"

approve "Deploy this revision to $SERVICE?" || {
  echo "Service unchanged; revision $NEW_TASK remains registered"
  exit 0
}

# Protect against overwriting a deployment made since inspection.
CURRENT=$(aws ecs describe-services \
  --cluster "$CLUSTER" --services "$SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' --output text)

[[ "$CURRENT" == "$OLD_TASK" ]] ||
  die "Service changed since inspection; refusing to overwrite it"

aws ecs update-service \
  --cluster "$CLUSTER" \
  --service "$SERVICE" \
  --task-definition "$NEW_TASK" \
  --region "$AWS_REGION" >/dev/null

echo "Waiting for service stability..."
aws ecs wait services-stable \
  --cluster "$CLUSTER" --services "$SERVICE" \
  --region "$AWS_REGION" ||
  die "Service did not stabilize; inspect ECS events"

FINAL_TASK=$(aws ecs describe-services \
  --cluster "$CLUSTER" --services "$SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' --output text)

[[ "$FINAL_TASK" == "$NEW_TASK" ]] ||
  die "Service is not using the new task definition"

TASK_ARN=$(aws ecs list-tasks \
  --cluster "$CLUSTER" --service-name "$SERVICE" \
  --desired-status RUNNING --region "$AWS_REGION" \
  --query 'taskArns[0]' --output text)

[[ "$TASK_ARN" == arn:* ]] || die "No running task found"

TASK_JSON=$(aws ecs describe-tasks \
  --cluster "$CLUSTER" --tasks "$TASK_ARN" \
  --region "$AWS_REGION")

# Find the actual application container name.
APP_CONTAINER=$(jq -r '
  [.tasks[0].containers[]
   | select((.name + " " + .image | ascii_downcase) | contains("nginx"))
   | .name][0] // empty
' <<<"$TASK_JSON")

[[ -n "$APP_CONTAINER" ]] || die "NGINX container not found"

echo
echo "Deployment succeeded."
echo "Task: $TASK_ARN"
echo "NGINX container: $APP_CONTAINER"
echo "Next: validate sensor processes with ECS Exec."
echo
echo "ECS Exec must be enabled and SSM permissions/network access configured."
echo "Run the validation command in the next section."
```

Run it:

bash

```
chmod 700 maintain-falcon-ecs.sh
./maintain-falcon-ecs.sh
```

Scope of this script: It automates the task-definition patch and deployment using an already approved image. It does not download a new sensor release from CrowdStrike. Image acquisition and publishing to ECR should be a separate controlled step, so a new release is not deployed merely because a new tag exists.

Also, this script assumes the patcher's `-pulltoken` format matches the version you are running and that private application images are accessible with the generated ECR credentials. Test these assumptions in development before adopting it for routine maintenance.

## 4. Refresh the Falcon sensor image in ECR

Before running the maintenance script, update the image in a controlled process.

The documented flow is:

1. Obtain the approved sensor version from CrowdStrike.
2. Authenticate to CrowdStrike's container registry.
3. Pull the approved sensor image.
4. Tag it for your ECR repository.
5. Push it to ECR.
6. Run `maintain-falcon-ecs.sh` using that exact versioned ECR image URI.

For example, after obtaining an approved upstream sensor tag:

bash

```
export AWS_REGION="us-east-1"
export ACCOUNT_ID="123456789012"
export SENSOR_TAG="YOUR_APPROVED_SENSOR_VERSION"

export ECR_IMAGE="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/falcon-sensor:${SENSOR_TAG}"

# Log in to the CrowdStrike registry first using credentials
# obtained through your approved CrowdStrike API workflow.
docker login registry.crowdstrike.com

# Pull the approved upstream sensor image.
docker pull \
  "registry.crowdstrike.com/falcon-container/us-2/release/falcon-sensor:${SENSOR_TAG}"

# Log in to ECR.
aws ecr get-login-password --region "$AWS_REGION" |
  docker login \
    --username AWS \
    --password-stdin \
    "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Tag and publish the image to ECR.
docker tag \
  "registry.crowdstrike.com/falcon-container/us-2/release/falcon-sensor:${SENSOR_TAG}" \
  "$ECR_IMAGE"

docker push "$ECR_IMAGE"
```

Replace `us-2` with the appropriate CrowdStrike cloud region for your Falcon account. The example assumes the destination ECR repository already exists and your AWS identity has push permissions.

Use the version returned by CrowdStrike's approved release process; do not assume that the last entry in a registry tag list is always the newest compatible version. The archived guide describes the registry authentication and mirroring process.

GitHub

Once pushed, the next deployment should use the new versioned URI, for example:

text

```
123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon-sensor:YOUR_APPROVED_SENSOR_VERSION
```

A push to ECR alone does not update running tasks. The task definition must be patched with the new URI, registered, and deployed.

## 5. Validate Falcon from a running ECS container

ECS service stability is necessary but not sufficient. You also need to inspect the application container and confirm that the sensor is running.

### Step A — Enable ECS Exec

ECS Exec lets you run commands inside a running container. It must be enabled for the service, and your IAM identity, task execution environment, and networking must support it.

For a development service, you can enable it with:

bash

```
aws ecs update-service \
  --cluster dev-cluster \
  --service nginx-service \
  --enable-execute-command \
  --region us-east-1
```

This setting alone does not guarantee ECS Exec will work for already-running tasks; you may need to redeploy them. Review the [AWS ECS Exec prerequisites](<https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html?utm_source=chatgpt.com>).

### Step B — Find the running task

bash

```
aws ecs list-tasks \
  --cluster dev-cluster \
  --service-name nginx-service \
  --desired-status RUNNING \
  --region us-east-1
```

Copy one task ARN from the output.

Then identify the application container name:

bash

```
aws ecs describe-tasks \
  --cluster dev-cluster \
  --tasks "YOUR_TASK_ARN" \
  --region us-east-1 \
  --query 'tasks[0].containers[*].{name:name,status:lastStatus,image:image}'
```

Use the actual NGINX application container name, not the initialization container.

### Step C — Inspect processes inside NGINX

bash

```
aws ecs execute-command \
  --cluster dev-cluster \
  --task "YOUR_TASK_ARN" \
  --container nginx \
  --interactive \
  --command "/bin/sh"
```

Replace `nginx` with the actual application container name.

Inside the shell, try:

sh

```
ps -ef
```

If the container has a compatible process-listing utility, filter for Falcon-related processes:

sh

```
ps -ef | grep -i '[f]alcon'
```

You can also inspect likely sensor-related files and directories if you know the paths used by your approved sensor version.

Not all NGINX images include `ps`, `grep`, or other diagnostic utilities. A minimal or distroless image may need a different supported verification method.

### Step D — Verify the sensor itself

Use these three independent checks:

| Check | What it proves |
| --- | --- |
| ECS task and service health | The task started and the service is stable. |
| Process inspection and sensor logs | Evidence that the sensor was initialized and is running inside the application container. |
| Falcon console telemetry | Evidence that the sensor is communicating with CrowdStrike and reporting as expected. |

The last check is essential. Finding a Falcon-related process is useful, but it does not prove that the sensor is healthy or successfully sending telemetry.

## 6. Operational checklist

Use this checklist for each sensor upgrade:

## Sensor upgrade checklist

0 of 8

[ ]Confirm the new sensor version is approved by security.

[ ]Mirror the versioned sensor image into ECR.

[ ]Run the patcher using the new ECR image and fresh registry tokens.

[ ]Review the task-definition diff and preserve required application settings.

[ ]Register and deploy the new revision to development.

[ ]Confirm NGINX and ECS service health.

[ ]Inspect sensor status and confirm telemetry in Falcon.

[ ]Record the deployed sensor version and rollback revision.

My recommendation is to separate image publishing from deployment. First publish a pinned sensor image to ECR, then patch and deploy it to a lower environment, and only promote that exact tested version after verifying both application health and Falcon telemetry. This gives you a repeatable upgrade process and a clear audit trail without automatically deploying an untested sensor release.

## Network requirements

Yes. The CrowdStrike Falcon Container sensor communicates outbound to CrowdStrike's cloud (SaaS) to send telemetry, detections, and other security information. Your NGINX application can continue serving traffic while the sensor reports security activity to Falcon. The archived ECS Fargate guide explicitly describes this telemetry flow.

GitHub

For ECS Fargate, you may need to verify outbound connectivity, but you usually do not need a new inbound security-group rule.

## 1. How the network traffic flows

ECS Fargate task

NGINX + Falcon sensor

Outbound HTTPS — TCP 443

Through NAT gateway or approved egress path

CrowdStrike Falcon cloud (SaaS)

Receives sensor telemetry and security events

The sensor initiates outbound connections. It does not normally require CrowdStrike to initiate a new inbound connection to your task.

## 2. Do you need to change the security group?

It depends on how your Fargate task connects to the internet.

| Existing network setup | What to do |
| --- | --- |
| Private subnet with NAT gateway and outbound HTTPS allowed | Usually no security-group change is required. |
| Private subnet with restricted egress | Allow outbound TCP 443 to the approved CrowdStrike destinations through your egress controls. |
| Private subnet without NAT or another permitted internet path | Configure an approved egress route or proxy. ECR VPC endpoints alone do not provide access to CrowdStrike SaaS. |
| Public subnet with a public IP | Outbound connectivity may already work, but private subnets are generally preferable for application workloads. |

AWS documents that private Fargate tasks can access the internet through a NAT gateway, or use VPC endpoints for supported AWS services. VPC endpoints do not automatically provide connectivity to third-party SaaS services.

Amazon Elastic Container Service

+1

### Example: existing security group

If your task security group already has this outbound rule:

hcl

```
egress {
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```

then outbound TCP 443 is already permitted by the security group. You may not need to change it.

However, this rule permits all outbound IPv4 traffic, so it may be broader than your organization's security policy allows.

### Example: restricted outbound access

If your security policy requires explicit egress rules, a basic rule could look like this:

hcl

```
egress {
  description = "HTTPS egress for approved external services"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = var.approved_egress_cidrs
}
```

Only use this if `approved_egress_cidrs` contains appropriate, maintained destination ranges for your egress design. Do not guess CrowdStrike IP addresses or assume that a single fixed CIDR covers all Falcon endpoints. If you use an HTTP proxy or AWS Network Firewall, enforce the approved destinations there instead.

## 3. Which CrowdStrike domains should be allowed?

The exact destinations depend on your Falcon cloud region, sensor version, and subscription. Obtain the current sensor network requirements and FQDNs from your CrowdStrike administrator or official documentation.

For a restricted network, you should verify:

- DNS resolution is working.
- Outbound TCP 443 is permitted.
- Your firewall or proxy permits the required CrowdStrike Falcon endpoints.
- TLS inspection, if enabled, does not interfere with sensor communications.
- Your egress logs show successful connections rather than blocked requests.

Avoid allowing every external domain simply to make the sensor work.

## 4\. How to test connectivity from ECS

First, use ECS Exec to enter the running NGINX container:

bash

```
aws ecs execute-command \
  --cluster dev-cluster \
  --task YOUR_TASK_ARN \
  --container nginx \
  --interactive \
  --command "/bin/sh"
```

If the image contains `curl`, you can test DNS and HTTPS connectivity to an approved Falcon endpoint:

sh

```
curl -Iv --connect-timeout 10 https://YOUR_APPROVED_FALCON_ENDPOINT/
```

A successful TLS connection confirms basic network reachability. An HTTP error response can still mean the network connection succeeded, so interpret the status code appropriately.

If NGINX is a minimal image and does not contain `curl`, don't install diagnostic tools into your production application container just for testing. Instead, use an approved diagnostic task in the same subnets and security group, or inspect VPC Flow Logs and firewall logs.

Finally, check the Falcon console for actual sensor telemetry. Network connectivity alone does not prove that the sensor has registered or is reporting successfully.

My recommendation: If your Fargate tasks are already in private subnets with a NAT gateway and unrestricted outbound HTTPS, test first without changing the security group. If outbound access is restricted, add only the approved Falcon destinations to your existing egress controls—no new inbound rule should normally be necessary.****


## 1. Can you test it entirely on your local machine?

Yes, but there are two different kinds of testing.

| Test | Can run locally? | What it proves |
| --- | --- | --- |
| Run the Falcon patching utility using Docker | Yes | The utility can read and modify your task-definition JSON. |
| Run a patched NGINX container with Docker | Potentially, subject to sensor compatibility | The patched image/container startup works in your local Docker environment. |
| Test the actual ECS Fargate deployment | No, not entirely locally | Confirms compatibility with ECS task definitions, Fargate, IAM, networking, and service deployment. |
| Verify telemetry in Falcon SaaS | Requires a real sensor deployment with working connectivity | Confirms the sensor is communicating with CrowdStrike. |

The linked CrowdStrike guide specifically documents running the patcher locally with Docker, but its sensor deployment target is ECS Fargate. The repository was archived in November 2024 and is explicitly unsupported.



## 2. Local Docker test — step by step

### Step 1: Check Docker and the Falcon image

bash

```
docker info
docker image ls
```

Make sure the Falcon image is already available locally under its full ECR image URI, for example:

bash

```
export FALCON_IMAGE="123456789012.dkr.ecr.us-east-1.amazonaws.com/falcon-sensor:YOUR_TAG"
docker image inspect "$FALCON_IMAGE"
```

Use the actual versioned tag you have approved.

### Step 2: Create a simple NGINX task definition

Create a file called `taskdefinition.json`:

json

```
{
  "family": "nginx-local-test",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:stable",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

Replace the account and role values with appropriate values for your environment. This is a task-definition specification for the patcher; it is not yet a running ECS task.

### Step 3: Prepare the patcher's pull token

The patcher may need to inspect private registry images to discover their entry points. For the private ECR Falcon image, generate a fresh token:

bash

```
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID="123456789012"

export PULL_TOKEN=$(
  printf '{"auths":{"%s.dkr.ecr.%s.amazonaws.com":{"auth":"%s"}}}' \
    "$AWS_ACCOUNT_ID" "$AWS_REGION" \
    "$(printf 'AWS:%s' "$(aws ecr get-login-password \
      --region "$AWS_REGION")" | base64 | tr -d '\n')" |
  base64 | tr -d '\n'
)
```

This assumes the Falcon image is in the specified account and region. If you use a different registry, adapt the authentication configuration.

### Step 4: Run the patcher

bash

```
export FALCON_CID="YOUR_CID_WITH_CHECKSUM"

mkdir -p local-test

cp taskdefinition.json local-test/taskdefinition.json

docker run --rm --pull=never \
  -v "$PWD/local-test:/var/run/spec" \
  "$FALCON_IMAGE" \
  -cid "$FALCON_CID" \
  -image "$FALCON_IMAGE" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > local-test/taskdefinitionwithfalcon.json

unset PULL_TOKEN
```

If the patcher exits successfully, inspect the output:

bash

```
jq '.containerDefinitions[] | {
  name,
  image,
  entryPoint,
  command,
  linuxParameters,
  mountPoints,
  dependsOn
}' local-test/taskdefinitionwithfalcon.json
```

Compare the original and patched JSON:

bash

```
diff -u \
  local-test/taskdefinition.json \
  local-test/taskdefinitionwithfalcon.json
```

You should see the expected sensor initialization, shared-volume, entry-point, and capability changes for the version of the patcher you are using. The documented patcher adds `SYS_PTRACE` and injects the sensor into the application containers.

GitHub

### Step 5: Can you run the patched task definition directly with Docker?

Not directly. Docker does not run an ECS task definition JSON as if it were a Docker Compose file. The task definition includes ECS-specific fields such as IAM roles, Fargate networking, and task-level CPU and memory.

You can test the NGINX image independently:

bash

```
docker run --rm -d \
  --name nginx-local-test \
  -p 8080:80 \
  nginx:stable

curl -I http://localhost:8080

docker rm -f nginx-local-test
```

This only verifies that NGINX runs normally. It does not verify that Falcon was injected or that the Falcon sensor can report telemetry.

For a real end-to-end test, deploy the patched task definition to a development ECS Fargate service, then verify the running container and Falcon console.

## 3. Recommended testing strategy

- Local Docker: validate the patcher, JSON changes, and baseline NGINX behavior.
- Development ECS Fargate: validate sensor initialization, container startup, networking, and task health.
- Falcon SaaS: confirm that telemetry arrives and the sensor is healthy.

That separation makes it much easier to identify whether a failure comes from the patcher, the application container, the ECS environment, or Falcon connectivity.




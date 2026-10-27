# Terraform: ECR + IAM + ECS Fargate + Application Load Balancer + Nginx

Below is a complete Terraform project to deploy a test Nginx web application on AWS using:

- ECR private repository to store the Docker image.
- Docker provider + `docker_registry_image` to build and push the image to ECR.
- IAM roles for ECS task execution and application runtime.
- ECS Fargate cluster, task definition, and service.
- Application Load Balancer (ALB) to expose Nginx over HTTP.
- Debian Trixie + Python 3.12 in the custom image.
- Terraform outputs for the ALB DNS name, image URI, repository URL, cluster, service, task family, and IAM roles.

The deployment uses public subnets for a straightforward test environment. The ALB accepts HTTP on port 80, and ECS tasks run in those subnets with public IPs so they can pull the image from ECR without a NAT gateway. This is suitable for testing, not a production security architecture.

## 1. Project structure

terraform-nginx/

provider.tf

AWS, Docker, and Terraform providers

variables.tf

Configurable inputs

data.tf

AWS account, region, and networking data

main.tf

ECR, networking, ALB, and security groups

iam.tf

ECS execution and task roles

ecr.tf

Private ECR repository and Docker image push

ecs.tf

ECS cluster, task definition, and service

output.tf

Deployment details and test URL

Dockerfile

Debian Trixie and Python 3.12 image

nginx.conf

Nginx reverse proxy configuration

app.py

Python test application

requirements.txt

Python dependencies

.dockerignore

Files excluded from the image

## 2. Terraform provider — `provider.tf`

The Docker provider uses the AWS ECR authorization token to authenticate. Terraform's Docker provider can build the local image and push it to the private repository using `docker_registry_image`.

hcl

```

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }

    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "docker" {
  registry_auth {
    address  = data.aws_ecr_authorization_token.ecr.proxy_endpoint
    username = data.aws_ecr_authorization_token.ecr.user_name
    password = data.aws_ecr_authorization_token.ecr.password
  }
}
```

The Docker provider requires a working Docker Engine or Docker Desktop on the machine running Terraform. The AWS identity running Terraform must also be authorized to create the infrastructure and push images to ECR.

## 3. Variables — `variables.tf`

hcl

```

variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Prefix used for resource names"
  type        = string
  default     = "nginx-demo"
}

variable "container_port" {
  description = "Port exposed by the Nginx container"
  type        = number
  default     = 80
}

variable "desired_count" {
  description = "Number of ECS tasks to run"
  type        = number
  default     = 2
}

variable "task_cpu" {
  description = "Fargate task CPU units"
  type        = number
  default     = 256
}

variable "task_memory" {
  description = "Fargate task memory in MiB"
  type        = number
  default     = 512
}

variable "image_tag" {
  description = "Docker image tag"
  type        = string
  default     = "v1"
}

variable "allowed_cidr_blocks" {
  description = "IPv4 CIDRs allowed to access the ALB"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}
```

For a public test, `0.0.0.0/0` allows anyone to reach the load balancer. Restrict this list to your own public IP range when appropriate.

## 4. AWS data sources — `data.tf`

This file discovers the AWS account, availability zones, and ECR credentials. The network itself will be created by Terraform rather than relying on a pre-existing default VPC.

hcl

```

data "aws_caller_identity" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ecr_authorization_token" "ecr" {}
```

## 5. Main infrastructure — `main.tf`

This file creates a VPC, two public subnets in separate availability zones, an internet gateway, a public route table, and security groups for the ALB and ECS tasks.

hcl

```

locals {
  name = var.project_name

  common_tags = {
    Project   = var.project_name
    ManagedBy = "Terraform"
  }
}

resource "aws_vpc" "main" {
  cidr_block           = "10.20.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(local.common_tags, {
    Name = "${local.name}-vpc"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${local.name}-igw"
  })
}

resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  cidr_block              = cidrsubnet("10.20.0.0/16", 8, count.index + 1)
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name}-public-${count.index + 1}"
    Tier = "public"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${local.name}-public-rt"
  })
}

resource "aws_route" "internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Public HTTP access to the ALB.
resource "aws_security_group" "alb" {
  name_prefix = "${local.name}-alb-"
  description = "Allow HTTP access to the ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from permitted CIDRs"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  egress {
    description = "Outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

# Only the ALB can reach the application container.
resource "aws_security_group" "ecs_tasks" {
  name_prefix = "${local.name}-tasks-"
  description = "Allow traffic from the ALB to ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "Nginx from ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Outbound traffic for ECR and AWS APIs"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

resource "aws_lb" "app" {
  name               = substr("${local.name}-alb", 0, 32)
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  tags = local.common_tags
}

resource "aws_lb_target_group" "app" {
  name        = substr("${local.name}-tg", 0, 32)
  port        = var.container_port
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = aws_vpc.main.id

  health_check {
    enabled             = true
    path                = "/health"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }

  tags = local.common_tags
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## 6. IAM roles — `iam.tf`

The execution role allows ECS to pull images from ECR and publish container logs to CloudWatch. The task role is for application code that needs to call AWS APIs; this sample application does not require additional AWS permissions.

hcl

```

data "aws_iam_policy_document" "ecs_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ecs_execution" {
  name               = "${var.project_name}-execution-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_assume_role.json

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "ecs_task" {
  name               = "${var.project_name}-task-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_assume_role.json

  tags = local.common_tags
}
```

The AWS-managed `AmazonECSTaskExecutionRolePolicy` includes the permissions needed for standard ECR image pulls and CloudWatch Logs delivery. If you later add S3, Secrets Manager, or other AWS integrations to your application, grant only the required permissions to the task role.

## 7. Private ECR and Docker image — `ecr.tf`

This is the important part of the image pipeline. The `docker_image` resource builds the local Docker image, and `docker_registry_image` pushes it to the ECR repository. The image rebuild triggers are calculated from the Dockerfile and application source files so that changes can be picked up on subsequent applies.

hcl

```

resource "aws_ecr_repository" "app" {
  name                 = "${var.project_name}/web"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = local.common_tags
}

resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep the latest 10 tagged images"
      selection = {
        tagStatus     = "tagged"
        tagPrefixList = ["v"]
        countType     = "imageCountMoreThan"
        countNumber   = 10
      }
      action = {
        type = "expire"
      }
    }]
  })
}

resource "docker_image" "app" {
  name = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"

  build {
    context    = path.module
    dockerfile = "Dockerfile"
    tag        = ["${aws_ecr_repository.app.repository_url}:${var.image_tag}"]
  }

  triggers = {
    source_hash = sha256(join("", [
      for f in [
        "Dockerfile",
        "nginx.conf",
        "app.py",
        "requirements.txt"
      ] : filesha256("${path.module}/${f}")
    ]))
  }

  depends_on = [
    aws_ecr_repository.app
  ]

  keep_locally = true
}

resource "docker_registry_image" "app" {
  name = docker_image.app.name

  triggers = {
    source_hash = docker_image.app.triggers["source_hash"]
  }

  depends_on = [
    docker_image.app
  ]

  keep_remotely = true
}
```

The ECR repository is private, with image scanning enabled on push. The lifecycle policy removes older tagged images with a `v` prefix once more than ten matching images exist.

Provider detail: the Docker provider separates local image building (`docker_image`) from registry publishing (`docker_registry_image`). This configuration follows that pattern.

Terraform Registry

+1

## 8. ECS cluster, task definition, and service — `ecs.tf`

The task runs on Fargate using `awsvpc` networking. ECS registers each task's private IP with the ALB target group, and the service maintains the requested number of running tasks.

hcl

```

resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.project_name}"
  retention_in_days = 7

  tags = local.common_tags
}

resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = local.common_tags
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project_name}-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.task_cpu
  memory                   = var.task_memory

  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "${var.project_name}-container"
      image     = docker_registry_image.app.name
      essential = true

      portMappings = [
        {
          name          = "http"
          containerPort = var.container_port
          hostPort      = var.container_port
          protocol      = "tcp"
        }
      ]

      healthCheck = {
        command = [
          "CMD-SHELL",
          "python -c \"import urllib.request; urllib.request.urlopen('http://127.0.0.1/health', timeout=3)\" || exit 1"
        ]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 15
      }

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "nginx"
        }
      }
    }
  ])

  tags = local.common_tags

  depends_on = [
    aws_iam_role_policy_attachment.ecs_execution,
    docker_registry_image.app
  ]
}

resource "aws_ecs_service" "app" {
  name            = "${var.project_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"
  enable_ecs_managed_tags = true

  deployment_minimum_healthy_percent = 50
  deployment_maximum_percent         = 200

  network_configuration {
    subnets          = aws_subnet.public[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "${var.project_name}-container"
    container_port   = var.container_port
  }

  health_check_grace_period_seconds = 60

  tags = local.common_tags

  depends_on = [
    aws_lb_listener.http,
    aws_iam_role_policy_attachment.ecs_execution
  ]
}
```

Important configuration details:

- Fargate requires `awsvpc` networking and an IP-type target group.
- The ECS service explicitly depends on the ALB listener and execution-role policy attachment.
- Container logs go to `/ecs/nginx-demo` by default, with a seven-day retention period.
- Both ECS's container health check and the ALB's `/health` check verify that the application responds.

The service configuration follows the AWS provider's ECS service model.

Terraform Registry

+1

## 9. Dockerfile — `Dockerfile`

This image uses Debian Trixie with Python 3.12, installs Nginx, and runs a Python HTTP application behind Nginx.

The `python:3.12-slim-trixie` base image is intended to provide Python 3.12 on Debian Trixie. Nginx handles incoming HTTP traffic on port 80 and proxies requests to Python on port 8000.

dockerfile

```

FROM python:3.12-slim-trixie

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends nginx \
    && rm -rf /var/lib/apt/lists/* \
    && rm -f /etc/nginx/sites-enabled/default

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app.py /app/app.py
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["sh", "-c", "python /app/app.py & exec nginx -g 'daemon off;'"]
```

This single-container setup is intentionally simple for a test. For production, run Nginx and the application in separate containers or use a dedicated application server and a proper process supervisor.

## 10. Nginx configuration — `nginx.conf`

Nginx serves the main page by proxying to the Python application and forwards the health endpoint to the same backend.

nginx

```

server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
    }

    location = /health {
        proxy_pass http://127.0.0.1:8000/health;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }
}
```

## 11. Python application — `app.py`

This small application provides a home page and a health-check endpoint. It uses Python's standard library, so no external Python dependencies are required.

python

```

from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import os
import socket

class AppHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(b'{"status":"healthy"}')
            return

        if self.path == "/":
            body = {
                "message": "Hello from Nginx on ECS Fargate!",
                "application": "nginx-python-demo",
                "hostname": socket.gethostname(),
                "python_version": os.sys.version.split()[0],
                "status": "running",
            }

            payload = (
                "<!doctype html><html><head>"
                "<meta charset='utf-8'>"
                "<title>ECS Nginx Demo</title></head><body>"
                "<h1>Hello from Nginx on ECS Fargate!</h1>"
                "<p>Your custom container is running successfully.</p>"
                f"<pre>{json.dumps(body, indent=2)}</pre>"
                "</body></html>"
            ).encode()

            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.send_header("Content-Length", str(len(payload)))
            self.end_headers()
            self.wfile.write(payload)
            return

        self.send_response(404)
        self.end_headers()
        self.wfile.write(b"Not found")

    def log_message(self, fmt, *args):
        print(f"{self.client_address[0]} - {fmt % args}")

if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8000), AppHandler)
    print("Python application listening on port 8000")
    server.serve_forever()
```

## 12. Python dependencies — `requirements.txt`

text

```

# No third-party Python packages required.
```

## 13. Docker ignore file — `.dockerignore`

text

```

.git
.terraform
*.tfstate
*.tfstate.*
terraform.tfvars
terraform.tfvars.json
crash.log
__pycache__
*.pyc
.env
```

## 14. Terraform outputs — `output.tf`

These outputs expose the URLs and resource identifiers needed to verify the deployment.

hcl

```

output "load_balancer_dns" {
  description = "Public DNS name of the Application Load Balancer"
  value       = aws_lb.app.dns_name
}

output "application_url" {
  description = "Open the Nginx application in a browser"
  value       = "http://${aws_lb.app.dns_name}"
}

output "health_check_url" {
  description = "Application health endpoint"
  value       = "http://${aws_lb.app.dns_name}/health"
}

output "ecr_repository_name" {
  description = "Private ECR repository name"
  value       = aws_ecr_repository.app.name
}

output "ecr_repository_url" {
  description = "Private ECR registry repository URL"
  value       = aws_ecr_repository.app.repository_url
}

output "image_url" {
  description = "Full image URI, including tag"
  value       = docker_registry_image.app.name
}

output "image_digest" {
  description = "Digest of the published image"
  value       = docker_registry_image.app.sha256_digest
}

output "ecs_cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "ecs_cluster_arn" {
  value = aws_ecs_cluster.main.arn
}

output "ecs_service_name" {
  value = aws_ecs_service.app.name
}

output "ecs_service_id" {
  value = aws_ecs_service.app.id
}

output "ecs_task_family" {
  value = aws_ecs_task_definition.app.family
}

output "ecs_task_definition_arn" {
  value = aws_ecs_task_definition.app.arn
}

output "ecs_container_name" {
  value = "${var.project_name}-container"
}

output "ecs_desired_count" {
  value = aws_ecs_service.app.desired_count
}

output "ecs_execution_role_arn" {
  value = aws_iam_role.ecs_execution.arn
}

output "ecs_task_role_arn" {
  value = aws_iam_role.ecs_task.arn
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.app.arn
}

output "cloudwatch_log_group" {
  value = aws_cloudwatch_log_group.app.name
}

output "aws_account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "aws_region" {
  value = var.aws_region
}
```

## 15. Run the Terraform project

### Prerequisites

Install the following on your local machine or CI runner:

- Terraform 1.6 or later.
- Docker Engine or Docker Desktop, running and accessible.
- AWS CLI configured with credentials for the target AWS account.
- AWS permissions to manage ECR, VPC, security groups, ALB, ECS, IAM, and CloudWatch resources.

### Step 1: Configure AWS credentials

For example, if you use an AWS CLI profile:

bash

```
aws configure --profile dev
export AWS_PROFILE=dev

aws sts get-caller-identity
```

On PowerShell, use:

powershell

```
$env:AWS_PROFILE = "dev"
aws sts get-caller-identity
```

Ensure that the identity has `iam:PassRole` permission for the two ECS roles. Creating IAM resources may also require permission to create and attach role policies.

### Step 2: Initialize Terraform

Run these commands from the project directory:

bash

```
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

### Step 3: Deploy the infrastructure

bash

```
terraform apply
```

Review the proposed changes and confirm when prompted.

Terraform will create the network and ECR repository, build the Docker image, push it to ECR, register the ECS task definition, and deploy the ECS service behind the ALB.

### Step 4: View the outputs

bash

```
terraform output
```

To get just the application URL:

bash

```
terraform output -raw application_url
```

To retrieve the image URI:

bash

```
terraform output -raw image_url
```

To get the ECS task family:

bash

```
terraform output -raw ecs_task_family
```

### Step 5: Test Nginx

Open the URL printed by the `application_url` output, or run:

bash

```
curl "$(terraform output -raw application_url)"
```

Test the health endpoint:

bash

```
curl "$(terraform output -raw health_check_url)"
```

Expected health response:

json

```
{"status":"healthy"}
```

The homepage should display Hello from Nginx on ECS Fargate\!, along with the application hostname and Python version.

## 16. Troubleshooting

| Symptom | What to check |
| --- | --- |
| ECR push fails | AWS identity, ECR permissions, Docker daemon, and registry authentication |
| ECS tasks fail to start | ECS stopped-task reason, task execution role, and CloudWatch logs |
| ALB returns 503 | Target group health, container port 80, and ECS task security group |
| Health checks fail | `/health` response, Nginx configuration, and Python application logs |
| Terraform does not rebuild the image | Confirm that the source files are included in the image trigger hash |

For ECS logs, run:

bash

```
aws logs tail "/ecs/nginx-demo" --follow --region us-east-1
```

For the ECS service status:

bash

```
aws ecs describe-services \
  --cluster "$(terraform output -raw ecs_cluster_name)" \
  --services "$(terraform output -raw ecs_service_name)" \
  --region "$(terraform output -raw aws_region)"
```

## 17. Important implementation notes

Before applying this configuration, make the following small adjustments for a smoother deployment.

1. Set the Docker registry address without the scheme.

In `provider.tf`, use the ECR registry hostname rather than its full HTTPS endpoint:

hcl

```
provider "docker" {
  registry_auth {
    address  = trimprefix(data.aws_ecr_authorization_token.ecr.proxy_endpoint, "https://")
    username = data.aws_ecr_authorization_token.ecr.user_name
    password = data.aws_ecr_authorization_token.ecr.password
  }
}
```

2. Understand the Python/Nginx health checks. The application and Nginx are in the same container. The Docker health check uses Python's standard library to query Nginx on port 80; the ALB independently checks `/health`.

3. Be aware of the public-subnet trade-off. ECS tasks receive public IPs so they can pull the private ECR image without NAT gateways or VPC endpoints. Their security group permits inbound application traffic only from the ALB, but they still have outbound internet access. For production, consider private task subnets, VPC endpoints for ECR and CloudWatch Logs, HTTPS with ACM, and a restricted ALB ingress CIDR.

4. Treat image tags carefully. The example uses a configurable `v1` tag for easy testing. If you change application code, the image build trigger rebuilds the local image and the registry resource can republish it. For repeatable production releases, prefer unique version tags or immutable digests.

5. Destroy test resources when finished.

bash

```
terraform destroy
```

The ECR lifecycle policy manages old tagged images, but the repository and all other infrastructure are removed by Terraform during destruction. Make sure you have backed up any images you want to keep.

## Private subnet

Update only the following files. This version uses your existing VPC and two existing private subnets, with both the internal ALB and ECS Fargate tasks deployed in private subnets. It removes the VPC, subnet, internet gateway, route table, and public IP creation resources.

The subnet IDs are supplied as a comma-separated variable. Ensure the private subnets have the required outbound connectivity to pull images from ECR and reach CloudWatch Logs, via NAT or suitable VPC endpoints.

## 1. `variables.tf`

Replace `variables.tf`, `data.tf`, `main.tf`, and `ecs.tf` with the following.

## 1. `variables.tf`

hcl

```

variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Prefix used for resource names"
  type        = string
  default     = "nginx-demo"
}

variable "vpc_id" {
  description = "Existing VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Two existing private subnet IDs, comma-separated"
  type        = string

  validation {
    condition = length([
      for id in split(",", var.private_subnet_ids) :
      trimspace(id) if trimspace(id) != ""
    ]) == 2

    error_message = "Provide exactly two comma-separated private subnet IDs."
  }
}

variable "container_port" {
  description = "Port exposed by the Nginx container"
  type        = number
  default     = 80
}

variable "desired_count" {
  description = "Number of ECS tasks to run"
  type        = number
  default     = 2
}

variable "task_cpu" {
  description = "Fargate task CPU units"
  type        = number
  default     = 256
}

variable "task_memory" {
  description = "Fargate task memory in MiB"
  type        = number
  default     = 512
}

variable "image_tag" {
  description = "Docker image tag"
  type        = string
  default     = "v1"
}

variable "allowed_cidr_blocks" {
  description = "CIDRs permitted to access the internal ALB"
  type        = list(string)
  default     = ["10.0.0.0/8"]
}
```

## 2. `data.tf`

hcl

```

data "aws_caller_identity" "current" {}

data "aws_ecr_authorization_token" "ecr" {}
```

## 3. `main.tf`

This uses the existing VPC and private subnets. The ALB is now internal, and the ECS security group allows application traffic only from the ALB security group.

hcl

```

locals {
  name = var.project_name

  private_subnet_ids = [
    for id in split(",", var.private_subnet_ids) :
    trimspace(id)
  ]

  common_tags = {
    Project   = var.project_name
    ManagedBy = "Terraform"
  }
}

resource "aws_security_group" "alb" {
  name_prefix = "${local.name}-alb-"
  description = "Allow approved clients to access the internal ALB"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTP from approved CIDRs"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  egress {
    description = "Outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

resource "aws_security_group" "ecs_tasks" {
  name_prefix = "${local.name}-tasks-"
  description = "Allow Nginx traffic from the internal ALB"
  vpc_id      = var.vpc_id

  ingress {
    description     = "Nginx from ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Outbound traffic for ECR and AWS APIs"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

resource "aws_lb" "app" {
  name               = substr("${local.name}-alb", 0, 32)
  internal           = true
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = local.private_subnet_ids

  tags = local.common_tags
}

resource "aws_lb_target_group" "app" {
  name        = substr("${local.name}-tg", 0, 32)
  port        = var.container_port
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = var.vpc_id

  health_check {
    enabled             = true
    path                = "/health"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }

  tags = local.common_tags
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## 4. `ecs.tf`

The key change is `assign_public_ip = false`. ECS tasks run only in your existing private subnets.

hcl

```

resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.project_name}"
  retention_in_days = 7

  tags = local.common_tags
}

resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = local.common_tags
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project_name}-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.task_cpu
  memory                   = var.task_memory

  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "${var.project_name}-container"
      image     = docker_registry_image.app.name
      essential = true

      portMappings = [
        {
          name          = "http"
          containerPort = var.container_port
          hostPort      = var.container_port
          protocol      = "tcp"
        }
      ]

      healthCheck = {
        command = [
          "CMD-SHELL",
          "python -c \"import urllib.request; urllib.request.urlopen('http://127.0.0.1/health', timeout=3)\" || exit 1"
        ]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 15
      }

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "nginx"
        }
      }
    }
  ])

  tags = local.common_tags

  depends_on = [
    aws_iam_role_policy_attachment.ecs_execution,
    docker_registry_image.app
  ]
}

resource "aws_ecs_service" "app" {
  name            = "${var.project_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"
  enable_ecs_managed_tags = true

  deployment_minimum_healthy_percent = 50
  deployment_maximum_percent         = 200

  network_configuration {
    subnets          = local.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "${var.project_name}-container"
    container_port   = var.container_port
  }

  health_check_grace_period_seconds = 60

  tags = local.common_tags

  depends_on = [
    aws_lb_listener.http,
    aws_iam_role_policy_attachment.ecs_execution
  ]
}
```

## 5. Supply the existing network IDs

Create or update `terraform.tfvars`:

hcl

```

aws_region         = "us-east-1"
project_name       = "nginx-demo"

vpc_id             = "vpc-0123456789abcdef0"
private_subnet_ids = "subnet-0123456789abcdef0,subnet-0abcdef1234567890"

allowed_cidr_blocks = ["10.20.0.0/16"]

desired_count = 2
task_cpu      = 256
task_memory   = 512
image_tag     = "v1"
```

Replace the example IDs and CIDR with your actual network values.

Run:

bash

```
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Your existing `output.tf`, `iam.tf`, `ecr.tf`, `provider.tf`, Dockerfile, and Nginx application files can remain unchanged.

Important: Because the ALB is internal, access `application_url` from a machine connected to the VPC, such as through VPN, Direct Connect, or a bastion host. Ensure the private subnets have ECR and CloudWatch connectivity through NAT or the required VPC endpoints; otherwise, image pulls or log delivery may fail.


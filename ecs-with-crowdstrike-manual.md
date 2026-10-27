# CrowdStrike Falcon Container Security for AWS ECS — Manual Installation Guide

## 3\. The simple manual process

Since your CDK deployment already works, you do not need to change CDK initially. Use the existing NGINX image and manually create a CrowdStrike-enabled revision of the ECS task definition.

### Step 1 — Export the current task definition

In AWS Console:

1. Open ECS → Task definitions.
2. Select the task definition used by your NGINX service.
3. Download or copy its JSON definition.

Alternatively, use the AWS CLI:

bash

```
aws ecs describe-task-definition \
  --task-definition YOUR_TASK_FAMILY \
  --query taskDefinition \
  --output json > taskdefinition.json
```

Remove AWS-generated fields such as `taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt` and `registeredBy` before re-registering it.

Validate: Confirm the JSON contains your existing NGINX image URI and the correct task configuration.

### Step 2 — Run the CrowdStrike patcher

Follow the [CrowdStrike ECS Fargate guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>) to authenticate to the CrowdStrike registry and obtain the Falcon sensor image.

Since you already have the Falcon sensor image in ECR, use that image to run the patcher:

bash

```
docker run --rm \
  -v "$PWD:/var/run/spec" \
  YOUR_ECR_FALCON_IMAGE_URI \
  -cid "YOUR_FALCON_CID" \
  -image YOUR_ECR_FALCON_IMAGE_URI \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > taskdefinitionwithfalcon.json
```

Replace the placeholders with your actual values. `$PULL_TOKEN` must be a valid ECR pull token in the format expected by the patcher; the patcher needs to inspect private application images if their entry points are not explicitly specified.

Validate: Inspect `taskdefinitionwithfalcon.json` and verify that the NGINX image is preserved and the CrowdStrike initialization and sensor configuration have been added.

### Step 3 — Register the patched revision

bash

```
aws ecs register-task-definition \
  --cli-input-json file://taskdefinitionwithfalcon.json
```

Validate: Record the new task-definition ARN returned by AWS.

### Step 4 — Update the ECS service

Update your existing service to use the new revision:

bash

```
aws ecs update-service \
  --cluster YOUR_CLUSTER \
  --service YOUR_SERVICE \
  --task-definition YOUR_NEW_TASK_DEFINITION_ARN
```

Validate: Wait for the service to reach steady state, confirm the new tasks are healthy, and verify NGINX responds.

### Step 5 — Verify CrowdStrike

Check the ECS task and container logs, then confirm the expected sensor telemetry appears in your Falcon console. A healthy ECS task alone does not prove that the sensor is operating.

## 4\. Important CDK consideration

Your CDK stack still defines the original task definition. That means a later `cdk deploy` can create a new revision without the manual CrowdStrike changes and move the service back to that definition.

For now, you can keep the manual process as a proof of concept. Before making it a permanent production workflow, choose one of these approaches:

- Manual: Reapply the patch after each CDK deployment.
- Automated later: Add a pipeline step that patches the task definition and deploys the patched revision.
- CDK-integrated: Update the CDK-generated task definition or CloudFormation template before deployment.

The referenced repository is archived and unsupported, so validate this workflow with your current CrowdStrike guidance before production use.
-------------------------------------

The repository contains an ECS Fargate guide that explains how to add the CrowdStrike Falcon Container sensor to an existing application task definition, deploy the updated task, and verify that the sensor is running. Important: the repository was archived on November 22, 2024, and its authors state that it is a community project, not an officially supported CrowdStrike product. Treat it as a reference and confirm the current installation procedure with your CrowdStrike documentation before production deployment.

GitHub

+1

## 1. First, understand what we are installing

For this guide, I will focus on Amazon ECS only, using ECS on Fargate as the primary example.

Cloud Icons | Amazon Elastic Container Service

Amazon ECS

Runs your application containers, such as NGINX, in AWS.

CrowdStrike Falcon Endpoint Protection - Features, Reviews & Pricing (February 2026)

CrowdStrike Falcon Container sensor

Monitors activity inside protected application containers and can report security detections to the Falcon console.

Elastic Container Registry SVG and PNG Icons | AWS Icons

Amazon ECR

Stores the Falcon sensor image so ECS can retrieve it during deployment.

The ECS Fargate approach in the repository uses a patching utility to modify your task definition. It injects a CrowdStrike initialization container and adjusts application container startup so the sensor can initialize before the application runs. It also adds the `SYS_PTRACE` Linux capability to the application containers.

GitHub

## 2. Understand the installation flow

Step 1 — Prepare AWS and Falcon access

Check ECS, Docker, AWS CLI and Falcon API permissions

Step 2 — Download the Falcon sensor image

Authenticate to the CrowdStrike registry

Step 3 — Copy the sensor image to ECR

Store the image in your AWS account

Step 4 — Patch the ECS task definition

Generate a new JSON file with CrowdStrike integration

Step 5 — Register and deploy the new task

Run the protected application in ECS

Step 6 — Validate

Check ECS status, logs, sensor operation and Falcon console

This is the manual workflow described in the repository's [ECS Fargate implementation guide](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>).

GitHub

## 3. Step-by-step manual installation

### Step 1 — Confirm your ECS environment

What you do: Identify the ECS cluster and the launch type used by your application.

In the AWS Console:

1. Open Amazon ECS → Clusters.
2. Select your cluster.
3. Identify whether your application uses Fargate or EC2.
4. Open Task definitions and find the task definition used by your application.
5. Note the task family, container image, CPU, memory, networking and IAM roles.

For this guide, assume your application is an NGINX container on Fargate.

Validation: AWS Console → ECS → Cluster → Services → Tasks. Confirm that your current application task is running before making changes.

You can also check from your terminal:

bash

```
aws sts get-caller-identity

aws ecs list-clusters --region us-east-1

aws ecs list-task-definitions \
  --status ACTIVE \
  --region us-east-1
```

Expected result: your AWS identity is returned, your cluster is listed, and your existing task definition can be found.

Important: This installation procedure is specifically for Fargate. If you use ECS on EC2, the correct sensor deployment approach may differ because you control the underlying container instances.

### Step 2 — Create CrowdStrike API credentials

What you do: Allow the installation workstation to authenticate to CrowdStrike and download the sensor image.

In the Falcon console:

1. Open Support → API Clients and Keys.
2. Create a dedicated API client for this installation.
3. Grant the `Falcon Image Download` read permission specified by the repository.
4. Save the client ID and client secret securely.
5. Find your Customer ID (CID), including its checksum, in the Falcon sensor download or host-management area.
6. Identify your Falcon cloud region.

The repository's examples use `us-1`, `us-2` or `eu-1` and their corresponding API endpoints. Confirm which region applies to your Falcon tenant.

GitHub

Set your variables in a private terminal session:

bash

```
export FALCON_CLIENT_ID='YOUR_CLIENT_ID'
read -rsp 'Falcon client secret: ' FALCON_CLIENT_SECRET
export FALCON_CLIENT_SECRET
echo

export FALCON_CID='YOUR_CID_WITH_CHECKSUM'
export FALCON_CLOUD_REGION='us-2'
export FALCON_CLOUD_API='api.us-2.crowdstrike.com'
```

Replace the example region and API endpoint with the values for your tenant.

Validation

- Confirm the client ID and CID are populated.
- Confirm the API endpoint matches your Falcon tenant.
- Do not print the client secret or commit it to Git.
- Confirm that the API client has the required image-download permission.

### Step 3 — Authenticate to the CrowdStrike registry

What you do: Obtain temporary API and registry credentials that allow you to download the sensor image.

First, obtain an OAuth token:

bash

```
export FALCON_API_BEARER_TOKEN=$(
  curl --fail --silent --show-error \
    --request POST \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode "client_id=${FALCON_CLIENT_ID}" \
    --data-urlencode "client_secret=${FALCON_CLIENT_SECRET}" \
    "https://${FALCON_CLOUD_API}/oauth2/token" |
  jq -er '.access_token'
)
```

Next, obtain the registry password:

bash

```
export FALCON_ART_PASSWORD=$(
  curl --fail --silent --show-error \
    --header "Authorization: Bearer ${FALCON_API_BEARER_TOKEN}" \
    "https://${FALCON_CLOUD_API}/container-security/entities/image-registry-credentials/v1" |
  jq -er '.resources[0].token'
)
```

Construct the registry username:

bash

```
export FALCON_ART_USERNAME="fc-$(
  printf '%s' "${FALCON_CID%%-*}" | tr '[:upper:]' '[:lower:]'
)"
```

The commands follow the registry authentication pattern in the repository.

GitHub

Validation

bash

```
test -n "$FALCON_API_BEARER_TOKEN" && echo "API token obtained"
test -n "$FALCON_ART_PASSWORD" && echo "Registry credential obtained"
```

Expected result:

text

```
API token obtained
Registry credential obtained
```

These checks confirm that values were returned; they do not independently prove that the registry login will succeed.

### Step 4 — Download the CrowdStrike sensor image

What you do: Discover the available sensor version and download the correct image for the Fargate installation.

Log in to the CrowdStrike registry:

bash

```
printf '%s' "$FALCON_ART_PASSWORD" |
  docker login \
    --username "$FALCON_ART_USERNAME" \
    --password-stdin \
    registry.crowdstrike.com
```

Expected result:

text

```
Login Succeeded
```

Then discover the registry bearer token and sensor tag using the repository's method:

bash

```
export SENSORTYPE='falcon-container'

export REGISTRYBEARER=$(
  curl --fail --silent --show-error \
    --user "${FALCON_ART_USERNAME}:${FALCON_ART_PASSWORD}" \
    "https://registry.crowdstrike.com/v2/token?=${FALCON_ART_USERNAME}&scope=repository:${SENSORTYPE}/${FALCON_CLOUD_REGION}/release/falcon-sensor:pull&service=registry.crowdstrike.com" |
  jq -er '.token'
)
```

bash

```
export FALCON_IMAGE_REPO="registry.crowdstrike.com/falcon-container/${FALCON_CLOUD_REGION}/release/falcon-sensor"

export FALCON_IMAGE_TAG=$(
  curl --fail --silent --show-error \
    --header "Authorization: Bearer ${REGISTRYBEARER}" \
    "https://registry.crowdstrike.com/v2/falcon-container/${FALCON_CLOUD_REGION}/release/falcon-sensor/tags/list" |
  jq -er '.tags[-1]'
)
```

Check the discovered values:

bash

```
printf 'Image repository: %s\n' "$FALCON_IMAGE_REPO"
printf 'Sensor tag: %s\n' "$FALCON_IMAGE_TAG"
```

Validation: Confirm that the registry returned a sensor tag and that the image repository corresponds to your Falcon cloud region. The newest tag returned by the legacy example should not automatically be assumed to be the recommended production version; verify the supported version in your current Falcon documentation.

### Step 5 — Copy the sensor image into Amazon ECR

What you do: Store the sensor image in your own AWS account so the ECS deployment can retrieve it.

Set your AWS values:

bash

```
export AWS_REGION='us-east-1'
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account --output text)

export ECR_REPOSITORY='falcon-sensor'
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
```

Create the repository if it does not already exist:

bash

```
aws ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY" \
  --region "$AWS_REGION" >/dev/null 2>&1 ||
aws ecr create-repository \
  --repository-name "$ECR_REPOSITORY" \
  --region "$AWS_REGION"
```

Authenticate to ECR:

bash

```
aws ecr get-login-password --region "$AWS_REGION" |
  docker login \
    --username AWS \
    --password-stdin "$ECR_REGISTRY"
```

Pull the sensor image:

bash

```
docker pull "${FALCON_IMAGE_REPO}:${FALCON_IMAGE_TAG}"
```

Tag and push it to ECR:

bash

```
export FALCON_ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${FALCON_IMAGE_TAG}"

docker tag \
  "${FALCON_IMAGE_REPO}:${FALCON_IMAGE_TAG}" \
  "$FALCON_ECR_IMAGE"

docker push "$FALCON_ECR_IMAGE"
```

The repository describes this process as mirroring the Falcon sensor image into a private AWS ECR repository.

GitHub

+1

Validation

bash

```
aws ecr describe-images \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$FALCON_IMAGE_TAG" \
  --region "$AWS_REGION"
```

Expected result: ECR returns image details, including the image digest.

### Step 6 — Prepare the existing ECS task definition

What you do: Download the existing task definition as JSON, without changing the currently running service.

First, find the task definition used by your application:

bash

```
aws ecs describe-services \
  --cluster YOUR_CLUSTER \
  --services YOUR_SERVICE \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' \
  --output text
```

Use the returned ARN to download the definition:

bash

```
export TASK_DEFINITION_ARN='PASTE_TASK_DEFINITION_ARN_HERE'

aws ecs describe-task-definition \
  --task-definition "$TASK_DEFINITION_ARN" \
  --region "$AWS_REGION" \
  --query taskDefinition \
  > taskdefinition-original.json
```

The AWS response includes read-only fields that must be removed before registering the JSON again. Use this command to produce a registration-ready input:

bash

```
jq 'del(
  .taskDefinitionArn,
  .revision,
  .status,
  .requiresAttributes,
  .compatibilities,
  .registeredAt,
  .registeredBy
)' taskdefinition-original.json > taskdefinition.json
```

Validation

bash

```
jq -e '.family and (.containerDefinitions | length > 0)' \
  taskdefinition.json
```

Expected result: `true`.

Open `taskdefinition.json` and confirm that your original application container, image, environment variables, port mappings, roles, and logging settings remain present.

### Step 7 — Run the CrowdStrike task-definition patcher

What you do: Use the patching utility in the Falcon container image to generate a new task definition that includes the sensor integration.

The patcher needs permission to inspect the images already referenced in your task definition. For images stored in private ECR, the repository uses a Docker pull token.

Create the token for your ECR registry. On Linux:

bash

```
export PULL_TOKEN=$(
  printf '{"auths":{"%s":{"auth":"%s"}}}' \
    "$ECR_REGISTRY" \
    "$(printf 'AWS:%s' "$(aws ecr get-login-password --region "$AWS_REGION")" |
      base64 -w 0)" |
  base64 -w 0
)
```

The token is temporary because ECR authentication expires. Generate it again when needed. Treat the token as a secret.

Run the patcher against the original task definition:

bash

```
mkdir -p ecs-task-files
cp taskdefinition.json ecs-task-files/

docker run --rm \
  -v "$PWD/ecs-task-files:/var/run/spec" \
  "$FALCON_ECR_IMAGE" \
  -cid "$FALCON_CID" \
  -image "$FALCON_ECR_IMAGE" \
  -pulltoken "$PULL_TOKEN" \
  -ecs-spec-file /var/run/spec/taskdefinition.json \
  > taskdefinitionwithfalcon.json
```

This follows the legacy repository's task-definition patching pattern.

GitHub

Validation

Inspect the generated file:

bash

```
jq -e '.containerDefinitions | length > 0' \
  taskdefinitionwithfalcon.json
```

Then inspect the container names, images, entry points and Linux capabilities:

bash

```
jq '.containerDefinitions[] |
  {
    name,
    image,
    entryPoint,
    command,
    linuxParameters,
    mountPoints
  }' taskdefinitionwithfalcon.json
```

Look for the CrowdStrike initialization container and the expected sensor-related changes. Check that your application container's original command and startup behavior have been preserved appropriately.

Important: The patcher may need to retrieve image metadata for your application containers. If it cannot access a private registry, the patching step may fail. Also, use a copy of your task definition and inspect the output before registering it. Do not assume that successful JSON generation proves the sensor will run correctly.

### Step 8 — Register the new task definition

What you do: Upload the patched JSON to ECS as a new revision. This does not replace your existing task definition revision.

bash

```
aws ecs register-task-definition \
  --cli-input-json file://taskdefinitionwithfalcon.json \
  --region "$AWS_REGION"
```

Save the new `taskDefinitionArn` returned by AWS.

Validation

bash

```
aws ecs describe-task-definition \
  --task-definition YOUR_NEW_TASK_DEFINITION_ARN \
  --region "$AWS_REGION" \
  --query 'taskDefinition.{Family:family,Revision:revision,Containers:containerDefinitions[*].name}'
```

Expected result: AWS returns the new revision and the container names, including the CrowdStrike initialization container if the patcher generated it.

If registration fails, inspect the AWS error message and confirm that the JSON contains no unsupported response-only fields.

### Step 9 — Deploy to ECS

Recommended approach: First deploy to a test service or a separate test task, not your production service.

For a standalone Fargate task:

bash

```
aws ecs run-task \
  --cluster YOUR_CLUSTER \
  --task-definition YOUR_NEW_TASK_DEFINITION_ARN \
  --launch-type FARGATE \
  --network-configuration \
    "awsvpcConfiguration={subnets=[YOUR_SUBNET_ID],securityGroups=[YOUR_SECURITY_GROUP_ID],assignPublicIp=DISABLED}" \
  --region "$AWS_REGION"
```

Use a subnet with the required network access. Private subnets need suitable NAT or VPC endpoints to reach the required AWS services and CrowdStrike endpoints.

Validation

Copy the task ARN from the command output and run:

bash

```
aws ecs describe-tasks \
  --cluster YOUR_CLUSTER \
  --tasks YOUR_TASK_ARN \
  --region "$AWS_REGION" \
  --query 'tasks[0].{LastStatus:lastStatus,DesiredStatus:desiredStatus,StoppedReason:stoppedReason,Containers:containers[*].{Name:name,Status:lastStatus,Reason:reason}}'
```

Expected result:

- The task reaches `RUNNING`.
- The application container reaches `RUNNING`.
- The initialization container completes as expected for its role.
- There are no image-pull, startup or initialization errors.

For a production ECS service, update the service to use the new task-definition revision through the ECS Console or `aws ecs update-service`, then monitor the deployment and health checks.

### Step 10 — Validate CrowdStrike sensor operation

This is the most important step. A task running successfully does not necessarily mean that the sensor has registered with CrowdStrike.

Perform these checks:

1. ECS task health: Confirm the task and application container remain running.
2. CloudWatch logs: Inspect the application's log stream and any available sensor or initialization logs for startup errors.
3. Falcon console: Open the relevant workload/container security views in your Falcon tenant and check for the protected workload and expected telemetry.
4. Sensor troubleshooting: If the sensor does not appear, check the CID, Falcon region, outbound connectivity and any required installation-token settings.

The archived guide also describes enabling sensor tracing with `-falconctl-opts "--trace=info"` and using ECS Exec to investigate the running sensor. The exact commands and supported diagnostics should be confirmed against the current sensor version.

GitHub

Validation checklist

### Deployment validation

0 of 6

[ ]Task reaches RUNNINGECS task status and application container status are healthy

[ ]Initialization completesNo sensor initialization or image-pull errors

[ ]Application worksNGINX or the application responds as expected

[ ]Logs are cleanNo recurring sensor startup or permission errors

[ ]Falcon receives telemetryExpected workload or sensor evidence appears in the Falcon console

[ ]Detection validationRun an approved CrowdStrike test procedure in a non-production environment

A detection test should use a test procedure approved by CrowdStrike, rather than running potentially harmful commands against a production workload.

## 4. Common problems and their fixes

| Problem | What to check |
| --- | --- |
| CrowdStrike registry login fails | API scopes, client credentials, CID and cloud region |
| Docker cannot pull the sensor | Registry authentication, image tag and registry permissions |
| Patcher fails to inspect an image | Private ECR pull token, image permissions and token expiration |
| ECS task stops during startup | Task stopped reason, initialization logs, networking and memory |
| Application no longer starts correctly | Patched entry point, original command, permissions and application logs |
| Sensor does not appear in Falcon | CID, outbound connectivity, installation-token requirements and sensor logs |
| Task definition registration fails | Remove AWS-managed response fields and validate the JSON |

## 5. Production considerations

- Use least privilege: Separate Falcon API download credentials from AWS ECS execution permissions.
- Protect secrets: Avoid storing API secrets or ECR pull tokens in source control, task-definition plaintext fields or build logs.
- Pin versions: Use a tested sensor image version rather than automatically promoting the newest tag.
- Plan upgrades: Repeat the patching and validation process when you upgrade the sensor or application image.
- Plan rollback: Keep the prior ECS task-definition revision available so the service can be returned to the previous version if health checks fail.
- Confirm support: The referenced GitHub repository is archived. Check the current CrowdStrike ECS deployment documentation before using this legacy workflow in production.

## 6. Reference links

- [CrowdStrike Container-Security repository](<https://github.com/CrowdStrike/Container-Security?utm_source=chatgpt.com>)
- [ECS Fargate installation guide in that repository](<https://github.com/CrowdStrike/Container-Security/blob/main/aws-ecs/ecs-fargate-guide.md?utm_source=chatgpt.com>)
- [CrowdStrike ECS Fargate Terraform module](<https://github.com/CrowdStrike/terraform-aws-ecs-fargate?utm_source=chatgpt.com>)
- [AWS ECS task-definition documentation](<https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html?utm_source=chatgpt.com>)
- [AWS ECS Exec documentation](<https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html?utm_source=chatgpt.com>)

One final distinction: These steps target ECS Fargate, where the Falcon Container sensor runs within the application task. ECS on EC2 has a different host-security model; do not apply the Fargate patching instructions to EC2 instances without confirming the appropriate CrowdStrike deployment method.

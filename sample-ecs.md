Below is a sample Amazon ECS task definition to run an NGINX web server with a custom HTML page. It uses the official NGINX Docker image and works with either ECS on Fargate or ECS on EC2.

The example uses a simple HTML page served by NGINX, with the page content mounted into the container through an ECS task-definition volume. This avoids having to build a custom Docker image for the demo.

## 1. Sample ECS task definition

Save as `nginx-task-definition.json`.

json

```

{
  "family": "nginx-web-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:stable",
      "essential": true,
      "portMappings": [
        {
          "name": "http",
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "nginx-html",
          "containerPath": "/usr/share/nginx/html",
          "readOnly": true
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx-web-task",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "nginx",
          "awslogs-create-group": "true"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "nginx-html",
      "host": {}
    }
  ]
}
```

Important: This is a structural example, not a directly deployable Fargate definition. Fargate does not support the EC2-style host bind mount shown here for supplying arbitrary host files, and the task definition does not automatically copy your local HTML file into the container. Use the Docker image approach below for a reliable, self-contained deployment.

## 2. Sample web page content

Create a file named `index.html`:

HTMLWrap linesCopy codePreview

```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My ECS NGINX Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f0f4f8;
            text-align: center;
            padding: 60px 20px;
        }
        .container {
            background: white;
            max-width: 650px;
            margin: auto;
            padding: 35px;
            border-radius: 12px;
            box-shadow: 0 4px 15px #00000015;
        }
        h1 { color: #146eb4; }
        p { color: #444; line-height: 1.6; }
        .status { color: #16803c; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Hello from Amazon ECS!</h1>
        <p>Your NGINX web server is running successfully.</p>
        <p class="status">Status: Application is healthy</p>
        <p>Deployed using Amazon ECS and Docker.</p>
    </div>
</body>
</html>
```

## 3. Package the HTML page into an NGINX image

Create a `Dockerfile` in the same directory as `index.html`.

dockerfile

```

FROM nginx:stable
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Your directory should look like this:

text

```
nginx-ecs/
├── Dockerfile
└── index.html
```

Build and test locally:

bash

```
docker build -t nginx-ecs-demo:latest .
docker run --rm -p 8080:80 nginx-ecs-demo:latest
```

Open `http://localhost:8080` to verify the page.

## 4. Push the image to Amazon ECR

Replace the AWS account ID and region with your own values.

bash

```
aws ecr create-repository \
  --repository-name nginx-ecs-demo \
  --region us-east-1

aws ecr get-login-password --region us-east-1 |
  docker login \
    --username AWS \
    --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

docker tag nginx-ecs-demo:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx-ecs-demo:latest

docker push \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx-ecs-demo:latest
```

## 5. Update the task definition

Use the following container configuration in the task definition from section 1. This version removes the host volume and uses your custom image instead.

json

```

{
  "family": "nginx-web-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx-ecs-demo:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx-web-task",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "nginx"
        }
      }
    }
  ]
}
```

Register the task definition:

bash

```
aws ecs register-task-definition \
  --cli-input-json file://nginx-task-definition.json
```

Before deploying, ensure that:

- The ECS task execution role has the standard ECS execution permissions, including ECR image pulls and CloudWatch Logs access.
- The CloudWatch log group `/ecs/nginx-web-task` exists in `us-east-1`, or create it beforehand.
- Your ECS cluster has Fargate networking configured, with a subnet and a security group permitting inbound TCP port 80 from your intended clients.
- For a public website, use a public IP with suitable routing or, preferably, an Application Load Balancer.

Result: ECS runs your custom NGINX image, and requests to port 80 display your sample HTML page.

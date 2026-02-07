# .NET ECS App

A basic ASP.NET Core Web API containerized and deployable to AWS ECS Fargate using Terraform.

## Project Structure

```
dotnet-ecs-app/
├── src/Api/                    # .NET Web API
│   ├── Program.cs              # App entry with logging
│   ├── Controllers/
│   │   ├── HealthController.cs # /health endpoint
│   │   └── WeatherForecastController.cs
│   └── appsettings.json
├── terraform/                  # AWS Infrastructure
│   ├── main.tf                 # Provider config
│   ├── variables.tf            # Input variables
│   ├── outputs.tf              # ALB URL output
│   ├── ecr.tf                  # Container registry
│   ├── vpc.tf                  # Networking
│   ├── alb.tf                  # Load balancer
│   └── ecs.tf                  # ECS Fargate cluster/service
├── Dockerfile                  # Multi-stage build
├── .dockerignore
└── .gitignore
```

## Prerequisites

- .NET 8.0 SDK
- Docker
- Terraform
- AWS CLI (configured with credentials)

## Local Development

### Run with .NET CLI

```bash
cd src/Api
dotnet run
```

Visit http://localhost:8080/health

### Run with Docker

```bash
# Build
docker build -t dotnet-ecs-app:latest .

# Run
docker run -p 8080:8080 dotnet-ecs-app:latest
```

```sh
curl -v http://localhost:8123/weatherforecast 2>&1 | head -30
```

## Deploy to AWS

### 1. Initialize and Apply Terraform

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

This creates:
- ECR repository
- VPC with public subnets
- ECS Fargate cluster and service
- Application Load Balancer
- CloudWatch log group

### 2. Build and Push Docker Image

```bash
# Get ECR URL from terraform output
ECR_URL=$(terraform output -raw ecr_repository_url)

# Login to ECR
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin $ECR_URL

# Build and push
docker build -t ${ECR_URL}:latest .
docker push ${ECR_URL}:latest
```

### 3. Deploy to ECS

After the first push, force ECS to pull the new image:

```bash
aws ecs update-service \
  --cluster dotnet-ecs-app-cluster \
  --service dotnet-ecs-app-service \
  --force-new-deployment
```

### 4. Access the Application

```bash
# Get the ALB URL
terraform output app_url
```

## API Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Health check - returns status and timestamp |
| `GET /weatherforecast` | Sample weather data |
| `GET /swagger` | Only works in Development mode (not in the container by default) |

## Configuration

Terraform variables can be customized in `terraform/variables.tf`:

| Variable | Default | Description |
|----------|---------|-------------|
| `aws_region` | `ap-southeast-2` | AWS region |
| `project_name` | `dotnet-ecs-app` | Project name for resource naming |
| `task_cpu` | `256` | Fargate task CPU units |
| `task_memory` | `512` | Fargate task memory (MB) |
| `desired_count` | `1` | Number of ECS tasks |

## Logs

Application logs are sent to CloudWatch Logs at `/ecs/dotnet-ecs-app`.

## Cleanup

Remove the ECR image (or let it be force deleted by the below)

```bash
cd terraform
terraform destroy
```

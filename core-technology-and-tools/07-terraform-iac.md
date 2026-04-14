# Terraform & Infrastructure as Code (IaC) - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Deployed Node.js services using Terraform, enabling efficient, scalable, and automated infrastructure management"*

---

## 1. What is Infrastructure as Code?

IaC manages infrastructure (servers, networks, databases) through **machine-readable configuration files** instead of manual processes.

### IaC Benefits
- **Version controlled** — track infrastructure changes in Git
- **Reproducible** — identical environments every time
- **Self-documenting** — code IS the documentation
- **Automated** — no manual clicks in console
- **Testable** — validate before deploying

### IaC Tools Comparison

| Tool | Type | Language | State | Provider Support |
|------|------|----------|-------|-----------------|
| **Terraform** | Declarative | HCL | Remote state | Multi-cloud |
| CloudFormation | Declarative | JSON/YAML | AWS-managed | AWS only |
| Pulumi | Imperative | TS/Python/Go | Pulumi Cloud | Multi-cloud |
| CDK | Imperative | TS/Python | CloudFormation | AWS (primary) |

---

## 2. Terraform Fundamentals

### Core Concepts
```
┌─────────────────────────────────────────────┐
│              Terraform Workflow              │
│                                             │
│   Write (.tf) → Plan → Apply → Destroy     │
│                                             │
│   ┌──────┐    ┌──────┐    ┌──────────┐     │
│   │ .tf  │───►│ Plan │───►│  Cloud   │     │
│   │files │    │(diff)│    │Resources │     │
│   └──────┘    └──────┘    └──────────┘     │
│       ▲                        │            │
│       │    ┌──────────┐        │            │
│       └────│  State   │◄───────┘            │
│            │  File    │                     │
│            └──────────┘                     │
└─────────────────────────────────────────────┘
```

### HCL (HashiCorp Configuration Language)

```hcl
# Provider configuration
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = var.project_name
    }
  }
}
```

---

## 3. Deploying Node.js on AWS Lambda

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project_name" {
  type    = string
  default = "user-service"
}
```

```hcl
# lambda.tf

# IAM Role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "${var.project_name}-${var.environment}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# IAM Policy
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_dynamodb" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ]
      Resource = aws_dynamodb_table.users.arn
    }]
  })
}

# Lambda Function
resource "aws_lambda_function" "api" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-${var.environment}"
  role             = aws_iam_role.lambda_role.arn
  handler          = "dist/handler.handler"
  runtime          = "nodejs20.x"
  memory_size      = 256
  timeout          = 30
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      NODE_ENV    = var.environment
      TABLE_NAME  = aws_dynamodb_table.users.name
    }
  }

  tracing_config {
    mode = "Active"  # Enable X-Ray
  }
}

# Package Lambda code
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/../dist"
  output_path = "${path.module}/../lambda.zip"
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/${aws_lambda_function.api.function_name}"
  retention_in_days = 14
}
```

```hcl
# api-gateway.tf

resource "aws_apigatewayv2_api" "api" {
  name          = "${var.project_name}-${var.environment}-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = ["*"]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
    max_age       = 3600
  }
}

resource "aws_apigatewayv2_stage" "default" {
  api_id      = aws_apigatewayv2_api.api.id
  name        = "$default"
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_logs.arn
    format = jsonencode({
      requestId    = "$context.requestId"
      ip           = "$context.identity.sourceIp"
      method       = "$context.httpMethod"
      path         = "$context.path"
      status       = "$context.status"
      responseTime = "$context.responseLatency"
    })
  }
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id                 = aws_apigatewayv2_api.api.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.api.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "proxy" {
  api_id    = aws_apigatewayv2_api.api.id
  route_key = "ANY /{proxy+}"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.api.execution_arn}/*/*"
}
```

```hcl
# dynamodb.tf

resource "aws_dynamodb_table" "users" {
  name         = "${var.project_name}-users-${var.environment}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }

  attribute {
    name = "email"
    type = "S"
  }

  global_secondary_index {
    name            = "email-index"
    hash_key        = "email"
    projection_type = "ALL"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Name = "${var.project_name}-users"
  }
}
```

```hcl
# outputs.tf
output "api_url" {
  value       = aws_apigatewayv2_api.api.api_endpoint
  description = "API Gateway endpoint URL"
}

output "lambda_function_name" {
  value = aws_lambda_function.api.function_name
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.users.name
}
```

---

## 4. Terraform Commands

```bash
# Initialize (download providers)
terraform init

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Plan (preview changes)
terraform plan -out=tfplan

# Apply changes
terraform apply tfplan

# Destroy all resources
terraform destroy

# State management
terraform state list
terraform state show aws_lambda_function.api
terraform state mv aws_s3_bucket.old aws_s3_bucket.new

# Import existing resources
terraform import aws_s3_bucket.my_bucket my-bucket-name

# Workspace management (environments)
terraform workspace new staging
terraform workspace select prod
terraform workspace list
```

---

## 5. Terraform Modules

```hcl
# modules/lambda/main.tf
variable "function_name" { type = string }
variable "handler" { type = string }
variable "runtime" { type = string default = "nodejs20.x" }
variable "source_dir" { type = string }
variable "environment_variables" { type = map(string) default = {} }
variable "memory_size" { type = number default = 256 }
variable "timeout" { type = number default = 30 }

resource "aws_lambda_function" "this" {
  function_name    = var.function_name
  handler          = var.handler
  runtime          = var.runtime
  memory_size      = var.memory_size
  timeout          = var.timeout
  # ... full implementation
}

output "function_arn" { value = aws_lambda_function.this.arn }
output "function_name" { value = aws_lambda_function.this.function_name }
```

```hcl
# Using the module
module "user_service" {
  source        = "./modules/lambda"
  function_name = "user-service-${var.environment}"
  handler       = "dist/handler.handler"
  source_dir    = "../dist"
  environment_variables = {
    TABLE_NAME = aws_dynamodb_table.users.name
  }
}

module "notification_service" {
  source        = "./modules/lambda"
  function_name = "notification-service-${var.environment}"
  handler       = "dist/notification.handler"
  source_dir    = "../notification-dist"
}
```

---

## 6. State Management

```hcl
# Remote state with S3 backend
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "services/user-service/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# State locking table
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

## 7. Best Practices

| Practice | Why |
|----------|-----|
| Use remote state | Team collaboration, no local state conflicts |
| State locking | Prevent concurrent modifications |
| Use variables & locals | DRY, environment flexibility |
| Module everything | Reusable, testable components |
| Version pin providers | Reproducible builds |
| Use `terraform plan` | Always review before applying |
| Separate environments | Different state files per env |
| Encrypt state | Contains sensitive data |
| Tag all resources | Cost tracking, ownership |
| Use `prevent_destroy` | Protect critical resources |

---

## 8. Interview Questions

1. **What is Terraform state?** — JSON file tracking resource-to-config mapping. Essential for plan/apply to know what exists.
2. **State locking purpose?** — Prevents concurrent terraform apply, avoiding resource corruption.
3. **`terraform plan` vs `terraform apply`?** — Plan: dry-run showing what will change. Apply: executes the changes.
4. **What are Terraform modules?** — Reusable, encapsulated configurations. Like functions for infrastructure.
5. **How to manage secrets?** — Use `sensitive = true`, AWS Secrets Manager, SSM Parameter Store. Never hardcode.
6. **What is `terraform import`?** — Brings existing cloud resources under Terraform management.
7. **Explain `count` vs `for_each`** — `count`: create N copies (index-based). `for_each`: create from map/set (key-based, more stable).
8. **What are data sources?** — Read-only queries to fetch existing resource info (e.g., AMI IDs, VPC details).

---

## 9. Practice Exercises

1. Deploy a Lambda + API Gateway + DynamoDB stack with Terraform
2. Create a reusable module for Lambda functions
3. Set up remote state with S3 + DynamoDB locking
4. Use `terraform workspace` for multi-environment deployments
5. Import an existing AWS resource into Terraform state

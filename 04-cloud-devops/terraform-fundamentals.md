# Terraform Fundamentals for Java Backend Engineers

---

## What Is Terraform?

Terraform is an Infrastructure as Code (IaC) tool that manages cloud resources declaratively. You describe the desired state, Terraform figures out what changes to make.

```hcl
# main.tf — Create an ECS service for Spring Boot

terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "myapp/prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "myapp-prod"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# ECS Service
resource "aws_ecs_service" "myapp" {
  name            = "myapp"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.myapp.arn
  desired_count   = 3

  load_balancer {
    target_group_arn = aws_lb_target_group.myapp.arn
    container_name   = "myapp"
    container_port   = 8080
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true  # Auto-rollback on deployment failure
  }
}

# RDS Instance
resource "aws_db_instance" "postgres" {
  identifier        = "myapp-prod"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 100

  db_name  = "myapp"
  username = "admin"
  password = var.db_password  # Use variable, not hardcoded

  multi_az               = true
  backup_retention_period = 7
  deletion_protection    = true

  tags = { Environment = "prod" }
}
```

---

## Terraform Commands

```bash
terraform init        # Initialize providers and backend
terraform plan        # Preview changes (SAFE — no changes made)
terraform apply       # Apply changes (prompted for confirmation)
terraform destroy     # Destroy all managed resources
terraform show        # Show current state
terraform output      # Show outputs
terraform import      # Import existing resource into state
```

---

## Key Terraform Concepts

**State file:** Terraform tracks current infrastructure state. Store remotely (S3 + DynamoDB locking) for team use.

**Variables:** Parameterize configurations for different environments.

**Modules:** Reusable infrastructure components (like functions for infrastructure).

**Workspaces:** Multiple state files from same configuration (dev/staging/prod).

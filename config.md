***terraform/
│── main.tf           # provider + backend (state storage)
│── variables.tf      # all input variables
│── vpc.tf            # networking (VPC, subnets, IGW, NAT, routes)
│── security.tf       # security groups
│── rds.tf            # MySQL database
│── redis.tf          # ElastiCache Redis
│── s3.tf             # S3 bucket for media
│── ecr.tf            # ECR repo for container images
│── ecs.tf            # ECS cluster (optional for compute)
│── outputs.tf        # outputs (endpoints, IDs, etc.)
│── terraform.tfvars  # your values (region, project name, etc.)
***

***Main.tf***
```sh
terraform {
  required_version = ">= 1.2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

```

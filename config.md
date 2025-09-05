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

**Variables.tf**
```sh
variable "aws_region" { default = "ap-south-1" }
variable "project"    { default = "bagisto" }
variable "vpc_cidr"   { default = "10.0.0.0/16" }

variable "public_subnet_cidrs" {
  default = ["10.0.10.0/24", "10.0.20.0/24"]
}

variable "private_subnet_cidrs" {
  default = ["10.0.101.0/24", "10.0.102.0/24"]
}

variable "db_name"     { default = "bagistodb" }
variable "db_username" { default = "bagisto" }
variable "db_password" { default = "" } # leave blank to auto-generate
```

🔹 vpc.tf
```sh
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "${var.project}-vpc" }
}

resource "aws_subnet" "public" {
  for_each          = toset(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  map_public_ip_on_launch = true
  tags = { Name = "${var.project}-public-${each.key}" }
}

resource "aws_subnet" "private" {
  for_each          = toset(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  map_public_ip_on_launch = false
  tags = { Name = "${var.project}-private-${each.key}" }
}
```
**🔹 security.tf**
```sh
resource "aws_security_group" "app" {
  vpc_id = aws_vpc.main.id
  name   = "${var.project}-app-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "db" {
  vpc_id = aws_vpc.main.id
  name   = "${var.project}-db-sg"

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```
**🔹 rds.tf**
```sh
resource "random_password" "db" {
  length  = 16
  special = true
}

locals {
  db_password_final = var.db_password != "" ? var.db_password : random_password.db.result
}

resource "aws_db_instance" "mysql" {
  allocated_storage    = 20
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  name                 = var.db_name
  username             = var.db_username
  password             = local.db_password_final
  vpc_security_group_ids = [aws_security_group.db.id]
  skip_final_snapshot  = true
}
```
**🔹 redis.tf**
```sh
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id          = "${var.project}-redis"
  replication_group_description = "Redis cluster for Bagisto"
  node_type                     = "cache.t3.micro"
  number_cache_clusters         = 1
  engine_version                = "6.x"
}
```
**🔹 s3.tf**
```sh
resource "aws_s3_bucket" "media" {
  bucket = "${var.project}-media-${substr(md5(var.project),0,6)}"
  acl    = "private"
}
```
**🔹 ecr.tf**
```sh
resource "aws_ecr_repository" "app" {
  name = "${var.project}-app"
}
```
**🔹 outputs.tf**
```sh
output "mysql_endpoint" {
  value = aws_db_instance.mysql.address
}
output "redis_endpoint" {
  value = aws_elasticache_replication_group.redis.primary_endpoint_address
}
output "s3_bucket" {
  value = aws_s3_bucket.media.id
}
output "ecr_repo" {
  value = aws_ecr_repository.app.repository_url
}
```
👉 Result

Now your infra is split and easy to read/manage.

main.tf → provider setup

variables.tf → inputs

vpc.tf → networking

security.tf → security groups

rds.tf → MySQL DB

redis.tf → cache

s3.tf → storage

ecr.tf → container registry

outputs.tf → endpoints

Do you want me to add ECS/EKS compute layer (so your Bagisto containers actually run on AWS), or keep infra limited to DB + Redis + S3 + ECR for now?

You said:
yes add 
ChatGPT said:

Perfect 🚀 — let’s extend your split Terraform setup with a compute layer so Bagisto containers can actually run.
You have two main choices:

ECS (Fargate) → simpler, AWS-managed container service.

EKS (Kubernetes) → more flexible/scalable, but more complex to manage.

Since you’re already thinking monitoring + scaling with Prometheus/Grafana, I’d recommend EKS (Kubernetes). But I’ll show both ECS (simple) and EKS (scalable) so you can pick.

📂 Updated Project Structure
terraform/
│── main.tf
│── variables.tf
│── vpc.tf
│── security.tf
│── rds.tf
│── redis.tf
│── s3.tf
│── ecr.tf
│── ecs.tf        # ECS Fargate option
│── eks.tf        # EKS option
│── outputs.tf
│── terraform.tfvars

**🔹 ecs.tf (ECS Fargate option)**

**# ECS Cluster**
```sh
resource "aws_ecs_cluster" "main" {
  name = "${var.project}-ecs-cluster"
}
```
**# Task Execution Role**
```sh
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "${var.project}-ecs-task-execution"
  assume_role_policy = data.aws_iam_policy_document.ecs_task_assume.json
}

data "aws_iam_policy_document" "ecs_task_assume" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role_policy_attachment" "ecs_task_exec_attach" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```
**# Task Definition**
```sh
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"

  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "bagisto"
      image     = "${aws_ecr_repository.app.repository_url}:latest"
      essential = true
      portMappings = [{
        containerPort = 80
        hostPort      = 80
      }]
      environment = [
        { name = "DB_HOST",  value = aws_db_instance.mysql.address },
        { name = "DB_USER",  value = var.db_username },
        { name = "DB_PASS",  value = local.db_password_final },
        { name = "REDIS_HOST", value = aws_elasticache_replication_group.redis.primary_endpoint_address }
      ]
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "${var.project}-svc"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = values(aws_subnet.private)[*].id
    security_groups = [aws_security_group.app.id]
  }
}

🔹 eks.tf (EKS option – recommended for scaling + monitoring)

Using AWS EKS module (simplifies a lot):

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "${var.project}-eks"
  cluster_version = "1.29"
  subnets         = values(aws_subnet.private)[*].id
  vpc_id          = aws_vpc.main.id

  eks_managed_node_groups = {
    default = {
      desired_size = 2
      min_size     = 2
      max_size     = 5

      instance_types = ["t3.medium"]
      disk_size      = 20
    }
  }

  tags = {
    Project = var.project
  }
}
```

After applying:

aws eks update-kubeconfig --region ap-south-1 --name bagisto-eks
kubectl get nodes


Then you can:

Deploy your Dockerized Bagisto app with Kubernetes manifests (Deployment, Service, HPA).

Install Prometheus + Grafana with Helm (kube-prometheus-stack).

Configure autoscaling (HPA + Cluster Autoscaler).

**🔹 outputs.tf (add compute outputs)**
```sh
output "ecs_cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "eks_cluster_name" {
  value       = module.eks.cluster_name
  description = "EKS cluster name"
}

output "eks_cluster_endpoint" {
  value       = module.eks.cluster_endpoint
}

output "eks_cluster_kubeconfig" {
  value       = module.eks.kubeconfig_filename
  description = "Kubeconfig file path for kubectl"
}
```

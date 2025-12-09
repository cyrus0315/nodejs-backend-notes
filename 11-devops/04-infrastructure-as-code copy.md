# Infrastructure as Code (IaC)

Infrastructure as Code 是现代云架构的基石，本文深入讲解 Terraform、Pulumi 和 AWS CloudFormation。

## 目录
- [IaC 概述](#iac-概述)
- [Terraform](#terraform)
- [Pulumi](#pulumi)
- [AWS CloudFormation](#aws-cloudformation)
- [IaC 最佳实践](#iac-最佳实践)
- [常见面试题](#常见面试题)

---

## IaC 概述

### 什么是 Infrastructure as Code？

```
传统方式（手动）              IaC 方式（代码）
┌─────────────────┐          ┌─────────────────┐
│  登录 AWS 控制台 │          │  编写 Terraform │
│        ↓        │          │      代码       │
│  手动创建 VPC   │          │        ↓        │
│        ↓        │          │   terraform     │
│  手动创建 EC2   │          │     apply       │
│        ↓        │          │        ↓        │
│  手动配置安全组  │          │ 自动创建所有资源 │
└─────────────────┘          └─────────────────┘
   难以复现                     版本控制
   易出错                       可重复
   无审计                       可审计
```

### IaC 工具对比

| 特性 | Terraform | Pulumi | CloudFormation |
|------|-----------|--------|----------------|
| 语言 | HCL | TypeScript/Python/Go | YAML/JSON |
| 提供商 | 多云 | 多云 | 仅 AWS |
| 状态管理 | 需要后端 | 托管/自管理 | AWS 管理 |
| 学习曲线 | 中等 | 低（熟悉语言） | 低（AWS 用户） |
| 社区 | 最大 | 快速增长 | AWS 官方 |
| 适用场景 | 多云、大规模 | 需要编程逻辑 | 纯 AWS |

### 核心概念

```typescript
// 声明式 vs 命令式

// 声明式（Terraform/CloudFormation）
// "我想要什么"
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}

// 命令式（传统脚本）
// "如何做"
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro
```

---

## Terraform

### 安装与配置

```bash
# macOS
brew install terraform

# 验证安装
terraform version

# 配置 AWS 凭证
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="us-east-1"

# 或使用 AWS Profile
export AWS_PROFILE=myprofile
```

### 基础语法

```hcl
# main.tf

# 提供商配置
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  # 远程状态存储
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
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

# 变量定义
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Project name"
  type        = string
}

# 本地变量
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
  
  name_prefix = "${var.project_name}-${var.environment}"
}

# 数据源
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

# 输出
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "account_id" {
  description = "AWS Account ID"
  value       = data.aws_caller_identity.current.account_id
}
```

### Node.js 应用基础设施

```hcl
# vpc.tf - VPC 配置
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

# 公共子网
resource "aws_subnet" "public" {
  count = length(data.aws_availability_zones.available.names)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 4, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Type = "public"
  }
}

# 私有子网
resource "aws_subnet" "private" {
  count = length(data.aws_availability_zones.available.names)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 4, count.index + 4)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Type = "private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

# NAT Gateway
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"
  
  tags = {
    Name = "${local.name_prefix}-nat-eip"
  }
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
  
  tags = {
    Name = "${local.name_prefix}-nat"
  }
  
  depends_on = [aws_internet_gateway.main]
}

# 路由表
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${local.name_prefix}-public-rt"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[0].id
    }
  }
  
  tags = {
    Name = "${local.name_prefix}-private-rt"
  }
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

```hcl
# ecs.tf - ECS 集群配置
resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = local.common_tags
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }
  
  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4
  }
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "${local.name_prefix}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name      = "app"
      image     = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"
      essential = true
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "PORT"
          value = "3000"
        }
      ]
      
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = aws_secretsmanager_secret.db_credentials.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }
      
      healthCheck = {
        command     = ["CMD-SHELL", "wget -q --spider http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
  
  tags = local.common_tags
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "${local.name_prefix}-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"
  
  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 3000
  }
  
  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }
  
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
  
  lifecycle {
    ignore_changes = [task_definition, desired_count]
  }
  
  depends_on = [aws_lb_listener.https]
  
  tags = local.common_tags
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${local.name_prefix}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

```hcl
# alb.tf - 负载均衡器
resource "aws_lb" "main" {
  name               = "${local.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = var.environment == "production"
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    prefix  = "alb"
    enabled = true
  }
  
  tags = local.common_tags
}

resource "aws_lb_target_group" "app" {
  name        = "${local.name_prefix}-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    matcher             = "200"
  }
  
  deregistration_delay = 30
  
  tags = local.common_tags
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"
  
  default_action {
    type = "redirect"
    
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

```hcl
# rds.tf - 数据库
resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-db-subnet"
  subnet_ids = aws_subnet.private[*].id
  
  tags = {
    Name = "${local.name_prefix}-db-subnet"
  }
}

resource "aws_rds_cluster" "main" {
  cluster_identifier     = "${local.name_prefix}-aurora"
  engine                 = "aurora-postgresql"
  engine_version         = "15.3"
  database_name          = var.db_name
  master_username        = var.db_username
  master_password        = random_password.db_password.result
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  backup_retention_period      = var.environment == "production" ? 35 : 7
  preferred_backup_window      = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  
  storage_encrypted   = true
  deletion_protection = var.environment == "production"
  skip_final_snapshot = var.environment != "production"
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  serverlessv2_scaling_configuration {
    max_capacity = 16.0
    min_capacity = 0.5
  }
  
  tags = local.common_tags
}

resource "aws_rds_cluster_instance" "main" {
  count = var.db_instance_count
  
  identifier           = "${local.name_prefix}-aurora-${count.index + 1}"
  cluster_identifier   = aws_rds_cluster.main.id
  instance_class       = "db.serverless"
  engine               = aws_rds_cluster.main.engine
  engine_version       = aws_rds_cluster.main.engine_version
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  performance_insights_enabled = true
  
  tags = local.common_tags
}

# 密码生成
resource "random_password" "db_password" {
  length  = 32
  special = false
}

# Secrets Manager 存储
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${local.name_prefix}/db-credentials"
  
  tags = local.common_tags
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = aws_rds_cluster.main.master_username
    password = random_password.db_password.result
    host     = aws_rds_cluster.main.endpoint
    port     = aws_rds_cluster.main.port
    database = aws_rds_cluster.main.database_name
  })
}
```

### 模块化

```hcl
# modules/vpc/main.tf
variable "name" {
  type = string
}

variable "cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "azs" {
  type = list(string)
}

variable "private_subnets" {
  type = list(string)
}

variable "public_subnets" {
  type = list(string)
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = var.name
  }
}

# ... 更多资源

output "vpc_id" {
  value = aws_vpc.this.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

```hcl
# 使用模块
module "vpc" {
  source = "./modules/vpc"
  
  name            = "${local.name_prefix}-vpc"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}

# 使用公共模块
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
  
  tags = local.common_tags
}
```

### Terraform 命令

```bash
# 初始化
terraform init

# 格式化
terraform fmt -recursive

# 验证
terraform validate

# 计划
terraform plan -var-file=production.tfvars -out=tfplan

# 应用
terraform apply tfplan

# 销毁
terraform destroy -var-file=production.tfvars

# 导入已有资源
terraform import aws_vpc.main vpc-12345678

# 状态管理
terraform state list
terraform state show aws_vpc.main
terraform state mv aws_vpc.main aws_vpc.primary
terraform state rm aws_vpc.main

# 工作空间
terraform workspace list
terraform workspace new staging
terraform workspace select production
```

---

## Pulumi

Pulumi 使用真正的编程语言（TypeScript/Python/Go）定义基础设施。

### 安装与配置

```bash
# 安装 Pulumi
brew install pulumi

# 配置 AWS
pulumi config set aws:region us-east-1

# 登录（使用 Pulumi Cloud 或本地状态）
pulumi login
# 或本地存储状态
pulumi login --local
```

### TypeScript 项目

```typescript
// index.ts - Pulumi 基础设施
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

// 配置
const config = new pulumi.Config();
const environment = config.require("environment");
const projectName = config.require("projectName");

const namePrefix = `${projectName}-${environment}`;

// VPC（使用高级抽象）
const vpc = new awsx.ec2.Vpc(`${namePrefix}-vpc`, {
  cidrBlock: "10.0.0.0/16",
  numberOfAvailabilityZones: 3,
  natGateways: {
    strategy: environment === "production" 
      ? awsx.ec2.NatGatewayStrategy.OnePerAz 
      : awsx.ec2.NatGatewayStrategy.Single,
  },
  tags: {
    Name: `${namePrefix}-vpc`,
    Environment: environment,
  },
});

// ECR 仓库
const repo = new aws.ecr.Repository(`${namePrefix}-repo`, {
  name: namePrefix,
  imageTagMutability: "IMMUTABLE",
  imageScanningConfiguration: {
    scanOnPush: true,
  },
});

// ECS 集群
const cluster = new aws.ecs.Cluster(`${namePrefix}-cluster`, {
  name: namePrefix,
  settings: [{
    name: "containerInsights",
    value: "enabled",
  }],
});

// RDS Aurora
const dbPassword = new pulumi.secret(config.requireSecret("dbPassword"));

const dbSubnetGroup = new aws.rds.SubnetGroup(`${namePrefix}-db-subnet`, {
  subnetIds: vpc.privateSubnetIds,
  tags: {
    Name: `${namePrefix}-db-subnet`,
  },
});

const dbSecurityGroup = new aws.ec2.SecurityGroup(`${namePrefix}-db-sg`, {
  vpcId: vpc.vpcId,
  description: "RDS security group",
  ingress: [{
    protocol: "tcp",
    fromPort: 5432,
    toPort: 5432,
    securityGroups: [ecsSecurityGroup.id],
  }],
});

const dbCluster = new aws.rds.Cluster(`${namePrefix}-aurora`, {
  clusterIdentifier: `${namePrefix}-aurora`,
  engine: "aurora-postgresql",
  engineVersion: "15.3",
  databaseName: "myapp",
  masterUsername: "admin",
  masterPassword: dbPassword,
  dbSubnetGroupName: dbSubnetGroup.name,
  vpcSecurityGroupIds: [dbSecurityGroup.id],
  storageEncrypted: true,
  serverlessv2ScalingConfiguration: {
    minCapacity: 0.5,
    maxCapacity: 16,
  },
  skipFinalSnapshot: environment !== "production",
});

const dbInstance = new aws.rds.ClusterInstance(`${namePrefix}-aurora-1`, {
  clusterIdentifier: dbCluster.id,
  instanceClass: "db.serverless",
  engine: "aurora-postgresql",
  engineVersion: dbCluster.engineVersion,
  dbSubnetGroupName: dbSubnetGroup.name,
});

// Secrets Manager
const dbSecret = new aws.secretsmanager.Secret(`${namePrefix}-db-secret`, {
  name: `${namePrefix}/db-credentials`,
});

const dbSecretValue = new aws.secretsmanager.SecretVersion(`${namePrefix}-db-secret-value`, {
  secretId: dbSecret.id,
  secretString: pulumi.interpolate`{
    "username": "${dbCluster.masterUsername}",
    "password": "${dbPassword}",
    "host": "${dbCluster.endpoint}",
    "port": ${dbCluster.port},
    "database": "${dbCluster.databaseName}"
  }`,
});

// ECS 安全组
const ecsSecurityGroup = new aws.ec2.SecurityGroup(`${namePrefix}-ecs-sg`, {
  vpcId: vpc.vpcId,
  description: "ECS tasks security group",
  ingress: [{
    protocol: "tcp",
    fromPort: 3000,
    toPort: 3000,
    securityGroups: [albSecurityGroup.id],
  }],
  egress: [{
    protocol: "-1",
    fromPort: 0,
    toPort: 0,
    cidrBlocks: ["0.0.0.0/0"],
  }],
});

// ALB
const albSecurityGroup = new aws.ec2.SecurityGroup(`${namePrefix}-alb-sg`, {
  vpcId: vpc.vpcId,
  description: "ALB security group",
  ingress: [
    { protocol: "tcp", fromPort: 80, toPort: 80, cidrBlocks: ["0.0.0.0/0"] },
    { protocol: "tcp", fromPort: 443, toPort: 443, cidrBlocks: ["0.0.0.0/0"] },
  ],
  egress: [{
    protocol: "-1",
    fromPort: 0,
    toPort: 0,
    cidrBlocks: ["0.0.0.0/0"],
  }],
});

const alb = new aws.lb.LoadBalancer(`${namePrefix}-alb`, {
  loadBalancerType: "application",
  securityGroups: [albSecurityGroup.id],
  subnets: vpc.publicSubnetIds,
  enableDeletionProtection: environment === "production",
});

const targetGroup = new aws.lb.TargetGroup(`${namePrefix}-tg`, {
  port: 3000,
  protocol: "HTTP",
  targetType: "ip",
  vpcId: vpc.vpcId,
  healthCheck: {
    enabled: true,
    path: "/health",
    healthyThreshold: 2,
    unhealthyThreshold: 3,
    timeout: 5,
    interval: 30,
  },
  deregistrationDelay: 30,
});

// HTTPS Listener
const certificate = aws.acm.getCertificate({
  domain: "*.example.com",
  statuses: ["ISSUED"],
});

const httpsListener = new aws.lb.Listener(`${namePrefix}-https`, {
  loadBalancerArn: alb.arn,
  port: 443,
  protocol: "HTTPS",
  sslPolicy: "ELBSecurityPolicy-TLS-1-2-2017-01",
  certificateArn: certificate.then(c => c.arn),
  defaultActions: [{
    type: "forward",
    targetGroupArn: targetGroup.arn,
  }],
});

// ECS Task Definition
const logGroup = new aws.cloudwatch.LogGroup(`${namePrefix}-logs`, {
  name: `/ecs/${namePrefix}`,
  retentionInDays: 30,
});

const executionRole = new aws.iam.Role(`${namePrefix}-execution-role`, {
  assumeRolePolicy: JSON.stringify({
    Version: "2012-10-17",
    Statement: [{
      Action: "sts:AssumeRole",
      Effect: "Allow",
      Principal: { Service: "ecs-tasks.amazonaws.com" },
    }],
  }),
});

new aws.iam.RolePolicyAttachment(`${namePrefix}-execution-policy`, {
  role: executionRole.name,
  policyArn: "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy",
});

const taskDefinition = new aws.ecs.TaskDefinition(`${namePrefix}-task`, {
  family: namePrefix,
  networkMode: "awsvpc",
  requiresCompatibilities: ["FARGATE"],
  cpu: "256",
  memory: "512",
  executionRoleArn: executionRole.arn,
  containerDefinitions: pulumi.interpolate`[
    {
      "name": "app",
      "image": "${repo.repositoryUrl}:latest",
      "essential": true,
      "portMappings": [{"containerPort": 3000}],
      "environment": [
        {"name": "NODE_ENV", "value": "${environment}"}
      ],
      "secrets": [
        {"name": "DATABASE_URL", "valueFrom": "${dbSecret.arn}"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "${logGroup.name}",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]`,
});

// ECS Service
const service = new aws.ecs.Service(`${namePrefix}-service`, {
  name: namePrefix,
  cluster: cluster.arn,
  taskDefinition: taskDefinition.arn,
  desiredCount: 2,
  launchType: "FARGATE",
  networkConfiguration: {
    subnets: vpc.privateSubnetIds,
    securityGroups: [ecsSecurityGroup.id],
    assignPublicIp: false,
  },
  loadBalancers: [{
    targetGroupArn: targetGroup.arn,
    containerName: "app",
    containerPort: 3000,
  }],
  deploymentConfiguration: {
    maximumPercent: 200,
    minimumHealthyPercent: 100,
  },
});

// Auto Scaling
const scalingTarget = new aws.appautoscaling.Target(`${namePrefix}-scaling-target`, {
  maxCapacity: 10,
  minCapacity: 2,
  resourceId: pulumi.interpolate`service/${cluster.name}/${service.name}`,
  scalableDimension: "ecs:service:DesiredCount",
  serviceNamespace: "ecs",
});

new aws.appautoscaling.Policy(`${namePrefix}-cpu-scaling`, {
  policyType: "TargetTrackingScaling",
  resourceId: scalingTarget.resourceId,
  scalableDimension: scalingTarget.scalableDimension,
  serviceNamespace: scalingTarget.serviceNamespace,
  targetTrackingScalingPolicyConfiguration: {
    predefinedMetricSpecification: {
      predefinedMetricType: "ECSServiceAverageCPUUtilization",
    },
    targetValue: 70,
    scaleInCooldown: 300,
    scaleOutCooldown: 60,
  },
});

// 输出
export const vpcId = vpc.vpcId;
export const clusterArn = cluster.arn;
export const albDns = alb.dnsName;
export const dbEndpoint = dbCluster.endpoint;
export const repoUrl = repo.repositoryUrl;
```

### Pulumi 命令

```bash
# 新建项目
pulumi new aws-typescript

# 预览
pulumi preview

# 部署
pulumi up

# 销毁
pulumi destroy

# Stack 管理
pulumi stack init staging
pulumi stack select production
pulumi stack ls

# 配置
pulumi config set environment production
pulumi config set --secret dbPassword "mysecret"

# 输出
pulumi stack output albDns

# 导出状态
pulumi stack export --file state.json
```

---

## AWS CloudFormation

### 基础模板

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Node.js Application Infrastructure'

Parameters:
  Environment:
    Type: String
    AllowedValues:
      - development
      - staging
      - production
    Default: development
  
  ProjectName:
    Type: String
    Default: my-nodejs-app
  
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
  
  ImageTag:
    Type: String
    Default: latest

Conditions:
  IsProduction: !Equals [!Ref Environment, production]

Mappings:
  EnvironmentConfig:
    development:
      InstanceCount: 1
      InstanceSize: 256
    staging:
      InstanceCount: 2
      InstanceSize: 512
    production:
      InstanceCount: 3
      InstanceSize: 1024

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-${Environment}-vpc'
        - Key: Environment
          Value: !Ref Environment

  # Public Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [0, !Cidr [!Ref VpcCidr, 8, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-${Environment}-public-1'

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [1, !Cidr [!Ref VpcCidr, 8, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-${Environment}-public-2'

  # Private Subnets
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [2, !Cidr [!Ref VpcCidr, 8, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-${Environment}-private-1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [3, !Cidr [!Ref VpcCidr, 8, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-${Environment}-private-2'

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # NAT Gateway
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1

  # Route Tables
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub '${ProjectName}-${Environment}'
      ClusterSettings:
        - Name: containerInsights
          Value: enabled

  # ECR Repository
  ECRRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Sub '${ProjectName}-${Environment}'
      ImageScanningConfiguration:
        ScanOnPush: true

  # ALB Security Group
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  # ECS Security Group
  ECSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ECS Tasks Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3000
          ToPort: 3000
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${ProjectName}-${Environment}-alb'
      Scheme: internet-facing
      Type: application
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2

  # Target Group
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${ProjectName}-${Environment}-tg'
      Port: 3000
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: ip
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3

  # HTTP Listener (Redirect to HTTPS)
  HTTPListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: redirect
          RedirectConfig:
            Protocol: HTTPS
            Port: '443'
            StatusCode: HTTP_301

  # Task Definition
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${ProjectName}-${Environment}'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceSize]
      Memory: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceSize]
      ExecutionRoleArn: !GetAtt ExecutionRole.Arn
      TaskRoleArn: !GetAtt TaskRole.Arn
      ContainerDefinitions:
        - Name: app
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${ECRRepository}:${ImageTag}'
          Essential: true
          PortMappings:
            - ContainerPort: 3000
          Environment:
            - Name: NODE_ENV
              Value: !Ref Environment
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs

  # ECS Service
  ECSService:
    Type: AWS::ECS::Service
    DependsOn: HTTPListener
    Properties:
      ServiceName: !Sub '${ProjectName}-${Environment}'
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref TaskDefinition
      DesiredCount: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceCount]
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          SecurityGroups:
            - !Ref ECSSecurityGroup
          Subnets:
            - !Ref PrivateSubnet1
            - !Ref PrivateSubnet2
      LoadBalancers:
        - ContainerName: app
          ContainerPort: 3000
          TargetGroupArn: !Ref TargetGroup

  # IAM Roles
  ExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

  TaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole

  # CloudWatch Log Group
  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/${ProjectName}-${Environment}'
      RetentionInDays: !If [IsProduction, 90, 30]

  # Auto Scaling
  AutoScalingTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 10
      MinCapacity: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceCount]
      ResourceId: !Sub 'service/${ECSCluster}/${ECSService.Name}'
      RoleARN: !Sub 'arn:aws:iam::${AWS::AccountId}:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService'
      ScalableDimension: ecs:service:DesiredCount
      ServiceNamespace: ecs

  AutoScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: cpu-scaling
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref AutoScalingTarget
      TargetTrackingScalingPolicyConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ECSServiceAverageCPUUtilization
        TargetValue: 70
        ScaleInCooldown: 300
        ScaleOutCooldown: 60

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${ProjectName}-${Environment}-VPCId'

  ALBDNSName:
    Description: ALB DNS Name
    Value: !GetAtt ApplicationLoadBalancer.DNSName

  ClusterArn:
    Description: ECS Cluster ARN
    Value: !GetAtt ECSCluster.Arn

  RepositoryUri:
    Description: ECR Repository URI
    Value: !GetAtt ECRRepository.RepositoryUri
```

### CloudFormation 命令

```bash
# 创建 Stack
aws cloudformation create-stack \
  --stack-name my-app-production \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=production \
    ParameterKey=ProjectName,ParameterValue=my-app \
  --capabilities CAPABILITY_IAM

# 更新 Stack
aws cloudformation update-stack \
  --stack-name my-app-production \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=production \
  --capabilities CAPABILITY_IAM

# 验证模板
aws cloudformation validate-template \
  --template-body file://template.yaml

# 查看变更集
aws cloudformation create-change-set \
  --stack-name my-app-production \
  --change-set-name my-changes \
  --template-body file://template.yaml

# 删除 Stack
aws cloudformation delete-stack \
  --stack-name my-app-production
```

---

## IaC 最佳实践

### 1. 状态管理

```hcl
# Terraform 远程状态
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # 状态锁
  }
}

# 创建状态存储基础设施
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### 2. 环境隔离

```bash
# 目录结构
infrastructure/
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   └── ...
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
└── global/
    ├── iam/
    └── dns/
```

### 3. 敏感数据处理

```hcl
# 使用 Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/db-password"
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)
}

# 或使用变量（通过 CI/CD 注入）
variable "db_password" {
  type      = string
  sensitive = true
}
```

### 4. 代码审查流程

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
      
      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/environments/production
      
      - name: Terraform Format
        run: terraform fmt -check -recursive
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Comment PR
        uses: actions/github-script@v6
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${process.env.PLAN}
            \`\`\`
            `;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
```

---

## 常见面试题

### 1. Terraform state 的作用是什么？

**作用**：
1. **资源映射**：记录配置到真实资源的映射
2. **性能优化**：缓存资源属性，减少 API 调用
3. **依赖跟踪**：记录资源间依赖关系
4. **团队协作**：使用远程状态，多人可共同操作

**最佳实践**：
- 使用远程后端（S3 + DynamoDB）
- 启用状态锁
- 启用版本控制
- 加密存储

### 2. 如何处理 Terraform 状态漂移？

**检测漂移**：
```bash
terraform plan  # 会显示差异
terraform refresh  # 更新状态（不推荐）
```

**处理方式**：
1. **代码同步**：更新代码以匹配实际状态
2. **资源替换**：`terraform apply` 恢复到期望状态
3. **状态导入**：`terraform import` 导入手动创建的资源
4. **忽略变更**：使用 `lifecycle { ignore_changes = [...] }`

### 3. Terraform vs Pulumi 如何选择？

**选择 Terraform**：
- 团队熟悉 HCL
- 需要大量社区模块
- 多云、大规模项目
- 对编程语言无要求

**选择 Pulumi**：
- 团队擅长 TypeScript/Python
- 需要复杂逻辑（循环、条件）
- 喜欢类型安全
- 测试基础设施代码

### 4. IaC 安全最佳实践？

1. **代码审查**：所有变更需 PR 审核
2. **最小权限**：CI/CD 使用最小权限
3. **敏感数据**：使用 Secrets Manager
4. **状态加密**：加密状态文件
5. **策略即代码**：使用 OPA/Sentinel
6. **漂移检测**：定期检查状态漂移
7. **审计日志**：记录所有变更

---

## 总结

### IaC 工具选择指南

| 场景 | 推荐 |
|------|------|
| 纯 AWS | CloudFormation 或 Terraform |
| 多云 | Terraform |
| 复杂逻辑 | Pulumi |
| 快速开始 | CloudFormation（AWS 用户） |
| 大规模 | Terraform |

### 关键要点

1. **版本控制所有代码**
2. **使用远程状态和状态锁**
3. **环境隔离**
4. **自动化 CI/CD**
5. **代码审查流程**
6. **敏感数据安全处理**

---

**上一篇**：[Kubernetes](./03-kubernetes.md)  
**下一篇**：[GitOps](./05-gitops.md)


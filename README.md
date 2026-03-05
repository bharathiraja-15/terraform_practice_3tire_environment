I'll help you create a comprehensive Terraform 4-tier architecture with 3 environments (dev, stage, prod) using CI/CD pipeline. Here's a step-by-step guide:

## 📁 Project Structure
```
terraform-multi-tier/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── stage/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── security/
├── pipeline/
│   ├── Jenkinsfile
│   └── gitlab-ci.yml
└── scripts/
    └── terraform-helper.sh
```

## Step 1: Create Terraform Modules

### Module 1: Networking (`modules/networking/main.tf`)
```hcl
# VPC Module
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Public Subnets (for Load Balancers)
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-public-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "Public"
  }
}

# App Tier Subnets (Private)
resource "aws_subnet" "app" {
  count             = length(var.app_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.app_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-app-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "App"
  }
}

# Database Tier Subnets (Private)
resource "aws_subnet" "db" {
  count             = length(var.db_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.db_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-db-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "Database"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

# NAT Gateways
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0
  domain = "vpc"

  tags = {
    Name        = "${var.environment}-nat-eip-${count.index + 1}"
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gateway ? length(var.availability_zones) : 0
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name        = "${var.environment}-nat-gw-${count.index + 1}"
    Environment = var.environment
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.environment}-public-rt"
    Environment = var.environment
  }
}

resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 1
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = var.enable_nat_gateway ? aws_nat_gateway.main[count.index].id : null
    }
  }

  tags = {
    Name        = "${var.environment}-private-rt-${count.index + 1}"
    Environment = var.environment
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.app_subnet_cidrs)
  subnet_id      = aws_subnet.app[count.index].id
  route_table_id = var.enable_nat_gateway ? aws_route_table.private[count.index].id : aws_route_table.private[0].id
}

# Security Groups
resource "aws_security_group" "alb" {
  name        = "${var.environment}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from internet"
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

  tags = {
    Name        = "${var.environment}-alb-sg"
    Environment = var.environment
  }
}

resource "aws_security_group" "app" {
  name        = "${var.environment}-app-sg"
  description = "Security group for App Tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-app-sg"
    Environment = var.environment
  }
}

resource "aws_security_group" "db" {
  name        = "${var.environment}-db-sg"
  description = "Security group for Database Tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "MySQL from App Tier"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  tags = {
    Name        = "${var.environment}-db-sg"
    Environment = var.environment
  }
}

# VPC Endpoints for S3 and DynamoDB
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = aws_route_table.private[*].id

  tags = {
    Name        = "${var.environment}-s3-endpoint"
    Environment = var.environment
  }
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.dynamodb"
  route_table_ids = aws_route_table.private[*].id

  tags = {
    Name        = "${var.environment}-dynamodb-endpoint"
    Environment = var.environment
  }
}
```

### Module 2: Compute (`modules/compute/main.tf`)
```hcl
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.alb_security_group_id]
  subnets           = var.public_subnet_ids

  enable_deletion_protection = var.environment == "prod" ? true : false

  tags = {
    Name        = "${var.environment}-alb"
    Environment = var.environment
  }
}

resource "aws_lb_target_group" "app" {
  name     = "${var.environment}-app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
  }

  tags = {
    Name        = "${var.environment}-app-tg"
    Environment = var.environment
  }
}

resource "aws_lb_listener" "front_end" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Launch Template for App Tier
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [var.app_security_group_id]

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
    db_address  = var.database_address
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.environment}-app-instance"
      Environment = var.environment
      Tier        = "App"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name               = "${var.environment}-app-asg"
  vpc_zone_identifier = var.app_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  min_size           = var.min_size
  max_size           = var.max_size
  desired_capacity   = var.desired_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.environment}-app-asg"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }

  dynamic "tag" {
    for_each = var.additional_tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}

# Auto Scaling Policies
resource "aws_autoscaling_policy" "cpu_policy" {
  name                   = "${var.environment}-cpu-policy"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown              = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_cloudwatch_metric_alarm" "cpu_alarm_high" {
  alarm_name          = "${var.environment}-cpu-alarm-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "CPUUtilization"
  namespace          = "AWS/EC2"
  period             = "120"
  statistic          = "Average"
  threshold          = "80"
  alarm_description  = "This metric monitors ec2 cpu utilization"
  alarm_actions      = [aws_autoscaling_policy.cpu_policy.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}
```

### Module 3: Database (`modules/database/main.tf`)
```hcl
# RDS Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = var.db_subnet_ids

  tags = {
    Name        = "${var.environment}-db-subnet-group"
    Environment = var.environment
  }
}

# RDS Parameter Group
resource "aws_db_parameter_group" "main" {
  name   = "${var.environment}-mysql-params"
  family = "mysql8.0"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  tags = {
    Name        = "${var.environment}-db-params"
    Environment = var.environment
  }
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier = "${var.environment}-mysql"

  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class

  allocated_storage     = var.allocated_storage
  storage_type         = "gp3"
  storage_encrypted    = true

  db_name  = var.db_name
  username = var.db_username
  password = random_password.db_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [var.db_security_group_id]

  backup_retention_period = var.environment == "prod" ? 30 : 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  multi_az               = var.environment == "prod" ? true : false
  skip_final_snapshot    = var.environment == "prod" ? false : true
  final_snapshot_identifier = var.environment == "prod" ? "${var.environment}-mysql-final-${formatdate("YYYY-MM-DD-hhmm", timestamp())}" : null

  enabled_cloudwatch_logs_exports = ["error", "general", "slowquery"]

  tags = {
    Name        = "${var.environment}-mysql"
    Environment = var.environment
  }
}

# Random password for database
resource "random_password" "db_password" {
  length  = 16
  special = false
}

# Store secret in AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.environment}-db-credentials"

  tags = {
    Name        = "${var.environment}-db-secret"
    Environment = var.environment
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = aws_db_instance.main.username
    password = random_password.db_password.result
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = aws_db_instance.main.db_name
  })
}
```

## Step 2: Create Environment Configurations

### Dev Environment (`environments/dev/main.tf`)
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = "4-Tier-App"
      ManagedBy   = "Terraform"
    }
  }
}

# Networking Module
module "networking" {
  source = "../../modules/networking"

  environment         = var.environment
  region             = var.aws_region
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  app_subnet_cidrs     = var.app_subnet_cidrs
  db_subnet_cidrs      = var.db_subnet_cidrs
  enable_nat_gateway   = var.enable_nat_gateway
}

# Database Module
module "database" {
  source = "../../modules/database"
  
  environment           = var.environment
  db_subnet_ids         = module.networking.db_subnet_ids
  db_security_group_id  = module.networking.db_security_group_id
  db_instance_class     = var.db_instance_class
  db_name              = var.db_name
  db_username          = var.db_username
  allocated_storage    = var.allocated_storage
}

# Compute Module
module "compute" {
  source = "../../modules/compute"
  
  environment            = var.environment
  vpc_id                 = module.networking.vpc_id
  public_subnet_ids      = module.networking.public_subnet_ids
  app_subnet_ids         = module.networking.app_subnet_ids
  alb_security_group_id  = module.networking.alb_security_group_id
  app_security_group_id  = module.networking.app_security_group_id
  database_address       = module.database.database_address
  ami_id                 = var.ami_id
  instance_type          = var.instance_type
  min_size              = var.min_size
  max_size              = var.max_size
  desired_capacity      = var.desired_capacity
  
  additional_tags = {
    CostCenter = "Development"
  }
}
```

### Dev Environment Variables (`environments/dev/terraform.tfvars`)
```hcl
environment = "dev"
aws_region  = "us-east-1"

# Networking
vpc_cidr           = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
app_subnet_cidrs     = ["10.0.10.0/24", "10.0.11.0/24"]
db_subnet_cidrs      = ["10.0.20.0/24", "10.0.21.0/24"]
enable_nat_gateway   = false

# Compute
ami_id           = "ami-0c02fb55956c7d316" # Amazon Linux 2
instance_type    = "t3.micro"
min_size         = 1
max_size         = 2
desired_capacity = 1

# Database
db_instance_class = "db.t3.micro"
db_name          = "appdb"
db_username      = "admin"
allocated_storage = 20
```

### Dev Backend (`environments/dev/backend.tf`)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### Stage Environment (`environments/stage/terraform.tfvars`)
```hcl
environment = "stage"
aws_region  = "us-east-1"

# Networking
vpc_cidr           = "10.1.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
app_subnet_cidrs     = ["10.1.10.0/24", "10.1.11.0/24", "10.1.12.0/24"]
db_subnet_cidrs      = ["10.1.20.0/24", "10.1.21.0/24", "10.1.22.0/24"]
enable_nat_gateway   = true

# Compute
ami_id           = "ami-0c02fb55956c7d316"
instance_type    = "t3.small"
min_size         = 2
max_size         = 4
desired_capacity = 2

# Database
db_instance_class = "db.t3.small"
db_name          = "appdb"
db_username      = "admin"
allocated_storage = 50
```

### Production Environment (`environments/prod/terraform.tfvars`)
```hcl
environment = "prod"
aws_region  = "us-east-1"

# Networking
vpc_cidr           = "10.2.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
public_subnet_cidrs  = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
app_subnet_cidrs     = ["10.2.10.0/24", "10.2.11.0/24", "10.2.12.0/24"]
db_subnet_cidrs      = ["10.2.20.0/24", "10.2.21.0/24", "10.2.22.0/24"]
enable_nat_gateway   = true

# Compute
ami_id           = "ami-0c02fb55956c7d316"
instance_type    = "t3.medium"
min_size         = 3
max_size         = 10
desired_capacity = 3

# Database
db_instance_class = "db.t3.medium"
db_name          = "appdb"
db_username      = "admin"
allocated_storage = 100
```

## Step 3: CI/CD Pipeline Configuration

### Jenkins Pipeline (`pipeline/Jenkinsfile`)
```groovy
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'prod'],
            description: 'Select the environment to deploy'
        )
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Select Terraform action'
        )
    }
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_VAR_aws_access_key = credentials('aws-access-key-id')
        TF_VAR_aws_secret_key = credentials('aws-secret-access-key')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Init') {
            steps {
                dir("environments/${params.ENVIRONMENT}") {
                    sh '''
                        terraform init \
                            -backend-config=bucket=my-terraform-state-bucket \
                            -backend-config=key=${ENVIRONMENT}/terraform.tfstate \
                            -backend-config=region=us-east-1 \
                            -backend-config=dynamodb_table=terraform-state-lock \
                            -backend-config=encrypt=true
                    '''
                }
            }
        }
        
        stage('Terraform Validate') {
            steps {
                dir("environments/${params.ENVIRONMENT}") {
                    sh 'terraform validate'
                }
            }
        }
        
        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'plan' || params.ACTION == 'apply' }
            }
            steps {
                dir("environments/${params.ENVIRONMENT}") {
                    sh 'terraform plan -out=tfplan'
                }
            }
        }
        
        stage('Approval') {
            when {
                expression { params.ACTION == 'apply' && params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Approve deployment to production?', ok: 'Deploy'
            }
        }
        
        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                dir("environments/${params.ENVIRONMENT}") {
                    sh 'terraform apply tfplan'
                }
            }
        }
        
        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                dir("environments/${params.ENVIRONMENT}") {
                    sh 'terraform destroy -auto-approve'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

### GitLab CI/CD (`pipeline/gitlab-ci.yml`)
```yaml
stages:
  - init
  - validate
  - plan
  - apply
  - destroy

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/environments/${ENVIRONMENT}
  TF_IN_AUTOMATION: "true"

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - cd ${TF_ROOT}
  - terraform --version
  - terraform init -reconfigure

.terraform-base: &terraform-base
  image: hashicorp/terraform:1.5.0
  variables:
    ENVIRONMENT: "dev"
  before_script:
    - cd ${TF_ROOT}
    - terraform init -reconfigure

init:
  stage: init
  extends: .terraform-base
  script:
    - terraform init

validate:
  stage: validate
  extends: .terraform-base
  script:
    - terraform validate
  dependencies:
    - init

plan:dev:
  stage: plan
  extends: .terraform-base
  variables:
    ENVIRONMENT: "dev"
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/plan.tfplan
  only:
    - main
    - develop

plan:stage:
  stage: plan
  extends: .terraform-base
  variables:
    ENVIRONMENT: "stage"
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/plan.tfplan
  only:
    - main

plan:prod:
  stage: plan
  extends: .terraform-base
  variables:
    ENVIRONMENT: "prod"
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/plan.tfplan
  only:
    - tags
  when: manual

apply:dev:
  stage: apply
  extends: .terraform-base
  variables:
    ENVIRONMENT: "dev"
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:dev
  only:
    - develop

apply:stage:
  stage: apply
  extends: .terraform-base
  variables:
    ENVIRONMENT: "stage"
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:stage
  only:
    - main

apply:prod:
  stage: apply
  extends: .terraform-base
  variables:
    ENVIRONMENT: "prod"
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:prod
  only:
    - tags
  when: manual
  environment:
    name: production

destroy:dev:
  stage: destroy
  extends: .terraform-base
  variables:
    ENVIRONMENT: "dev"
  script:
    - terraform destroy -auto-approve
  only:
    - schedules
  when: manual

destroy:stage:
  stage: destroy
  extends: .terraform-base
  variables:
    ENVIRONMENT: "stage"
  script:
    - terraform destroy -auto-approve
  only:
    - schedules
  when: manual
```

## Step 4: Helper Scripts

### Terraform Helper (`scripts/terraform-helper.sh`)
```bash
#!/bin/bash

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to display usage
usage() {
    echo "Usage: $0 {init|plan|apply|destroy|output} {dev|stage|prod} [options]"
    exit 1
}

# Check if environment is provided
if [ $# -lt 2 ]; then
    usage
fi

ACTION=$1
ENVIRONMENT=$2
shift 2

# Set working directory
WORKING_DIR="environments/${ENVIRONMENT}"

# Check if directory exists
if [ ! -d "${WORKING_DIR}" ]; then
    echo -e "${RED}Error: Environment directory ${WORKING_DIR} does not exist${NC}"
    exit 1
fi

# Function to run terraform commands
run_terraform() {
    cd "${WORKING_DIR}"
    
    case $ACTION in
        init)
            echo -e "${GREEN}Initializing Terraform for ${ENVIRONMENT}...${NC}"
            terraform init "$@"
            ;;
        plan)
            echo -e "${GREEN}Planning Terraform changes for ${ENVIRONMENT}...${NC}"
            terraform plan "$@"
            ;;
        apply)
            echo -e "${YELLOW}Applying Terraform changes for ${ENVIRONMENT}...${NC}"
            echo -e "${RED}WARNING: This will apply changes to ${ENVIRONMENT} environment!${NC}"
            
            if [ "${ENVIRONMENT}" == "prod" ]; then
                read -p "Are you sure you want to apply to PRODUCTION? (yes/no): " confirmation
                if [ "${confirmation}" != "yes" ]; then
                    echo -e "${RED}Cancelled${NC}"
                    exit 1
                fi
            fi
            
            terraform apply "$@"
            ;;
        destroy)
            echo -e "${RED}Destroying Terraform resources for ${ENVIRONMENT}...${NC}"
            echo -e "${RED}WARNING: This will destroy all resources in ${ENVIRONMENT} environment!${NC}"
            
            read -p "Are you sure you want to destroy? (yes/no): " confirmation
            if [ "${confirmation}" != "yes" ]; then
                echo -e "${RED}Cancelled${NC}"
                exit 1
            fi
            
            terraform destroy "$@"
            ;;
        output)
            echo -e "${GREEN}Getting Terraform outputs for ${ENVIRONMENT}...${NC}"
            terraform output "$@"
            ;;
        *)
            echo -e "${RED}Error: Unknown action ${ACTION}${NC}"
            usage
            ;;
    esac
}

# Check if terraform is installed
if ! command -v terraform &> /dev/null; then
    echo -e "${RED}Error: Terraform is not installed${NC}"
    exit 1
fi

# Run the command
run_terraform "$@"
```

## Step 5: Make Helper Script Executable
```bash
chmod +x scripts/terraform-helper.sh
```

## Step 6: Usage Examples

### Initialize environment
```bash
./scripts/terraform-helper.sh init dev
./scripts/terraform-helper.sh init stage
./scripts/terraform-helper.sh init prod
```

### Plan changes
```bash
./scripts/terraform-helper.sh plan dev
./scripts/terraform-helper.sh plan stage
./scripts/terraform-helper.sh plan prod
```

### Apply changes
```bash
./scripts/terraform-helper.sh apply dev -auto-approve
./scripts/terraform-helper.sh apply stage -auto-approve
./scripts/terraform-helper.sh apply prod
```

### Get outputs
```bash
./scripts/terraform-helper.sh output dev
```

## Step 7: S3 Bucket and DynamoDB Table for State Locking

Create a separate Terraform configuration for state backend:

```hcl
# backend-setup/main.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket-unique-name"
  
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

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

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

## Step 8: Best Practices

1. **Use Terragrunt** for managing multiple environments
2. **Implement proper tagging** strategy
3. **Use remote state** with locking
4. **Implement cost controls** especially for dev environments
5. **Set up monitoring and alerting**
6. **Use workspaces** if environments are similar
7. **Implement proper IAM roles and policies**
8. **Use modules from Terraform Registry** where possible

This setup provides:
- ✅ 4-tier architecture (Web/ALB, App, DB, Networking)
- ✅ 3 environments (dev, stage, prod)
- ✅ CI/CD pipeline integration (Jenkins/GitLab)
- ✅ Remote state with locking
- ✅ Security best practices
- ✅ Auto-scaling capabilities
- ✅ Helper scripts for easy management

The pipeline will automatically handle different environments based on branches/tags and includes approval gates for production deployments.

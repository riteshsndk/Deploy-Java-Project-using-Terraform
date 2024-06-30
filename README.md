# Install git in ubuntu 22.04 
1) sudo apt install git -y
2) git init 
3) git clone https://github.com/dtchanpura/test-project-java

# Install and Use Docker on Ubuntu 22.04
1) sudo apt install docker.io
2) vi .bashrc
   add below command in .bashrc file
sudo chown $USER /var/run/docker.sock

.bashrc file is typically used to change the ownership of the Docker socket file (/var/run/docker.sock) to the current
user ($USER). This is necessary to allow the current user to interact with the Docker daemon without needing to use sudo
for every Docker command
------------------------------------------------------------------------------------------------------------------------------

# Create Docker Configuration
 cd test-project-java/
 
Create Dockerfile
1) sudo systemctl start docker
2) sudo systemctl status docker
3) vi Dockerfile 
----------------------
FROM openjdk:17-jdk-slim

# Install Maven and Make
RUN apt-get update \
    && apt-get install -y maven make \
    && apt-get clean;

# Set the working directory in the container
WORKDIR /app

# Copy the project source code and pom.xml file into the container
COPY . /app/
COPY pom.xml /app

# Run Maven clean install to build the application
RUN make

RUN cp /app/target/*.jar /app/app.jar

# Expose the port that the application will run on
#EXPOSE 80

# Command to run the application
CMD ["java", "-jar", "app.jar", "--server.port=80"]


------------------------
4) sudo docker build -t my-test-project-java-image:test-v0.0.2 .
5) sudo docker images
6) sudo docker run -itd 025d9c433634(Image ID)
7) sudo docker run -itp 80:80 -d 025d9c433634 (Image ID) ......(check container would be up and site is it live or not )

# Download AWS CLI
Download AWS CLI 
sudo apt install zip unzip net-tools -y
mkdir aws_cli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
aws --version ( Check CLI Version) 

# Attach Role to EC2 Instance
ACustomAdminFullAccessRole attach to Ec2 Server 


# Push this image into ECR using below commands
![image](https://github.com/riteshsndk/Deploy-Java-Project-using-Terraform/assets/168401480/ea8a9296-f626-4955-a4ff-68ac7bcf9f33)



# Install Terraform on Ubuntu 22.04

1) sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
2) wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
3) gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
4) echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

5) sudo apt update
6) sudo apt-get install terraform
7) terraform --version
-----------------------------------------------------------------------------------------------------------------------------------------------------------

-----------------------------------------------------------------------------------------------------------------------------------------------------------




# README.md file:

# Terraform Configuration Files #

1) ec2.tf file

# EC2 Terraform Configuration

This `ec2.tf` file sets up AWS resources for ECS tasks on EC2 instances, including IAM roles, a launch template, an auto-scaling group, and an ALB.

## IAM Role and Instance Profile
- **IAM Role**: `ecs_instance_role` allows EC2 instances to interact with ECS.
- **Instance Profile**: `ecs_instance_profile` associates the role with EC2 instances.
- **Role Policy Attachment**: Grants necessary permissions with `AmazonEC2ContainerServiceforEC2Role`.

## Launch Template
Defines a launch template `ecs_lt` with:
- AMI: `ami-0326a0330a9e7731c`
- Instance Type: `t2.medium`
- Security Group: `aws_security_group.security_group.id`
- Key Name: `mumbai_region_key`
- EBS Volume: 30 GB `gp2`
- User Data: `ecs.sh` script

## Auto Scaling Group
Creates `ecs_asg` with:
- Subnets: `aws_subnet.subnet.id`, `aws_subnet.subnet2.id`
- Desired Capacity: 2
- Max Size: 3
- Min Size: 1
- Launch Template: `ecs_lt`

## Application Load Balancer (ALB)
- **ALB**: `ecs_alb` distributes traffic to instances.
- **Listener**: `ecs_alb_listener` handles HTTP traffic on port 80.
- **Target Group**: `ecs_tg` routes traffic to instances.


This configuration sets up a scalable and load-balanced environment for ECS tasks on EC2 instances with the necessary permissions and configurations.
-----------------------------------------------------------------------------------------------------------------------------------


2) main.tf file
   # ECS Cluster Terraform Configuration

This Terraform configuration sets up an ECS cluster with a capacity provider, task definition, and service.

## Overview

The configuration performs the following:

1. **Creates an ECS Cluster**: 
   - Defines an ECS cluster named `my-ecs-cluster`.

2. **Configures a Capacity Provider**:
   - Sets up a capacity provider `test1` using an auto-scaling group with managed scaling.
   - Associates the capacity provider with the ECS cluster and specifies a default capacity provider strategy.

3. **Defines an ECS Task Definition**:
   - Specifies the ECS task definition `my-ecs-task`, including network mode, execution role, CPU, runtime platform, and container definitions (using an image from ECR).

4. **Creates an ECS Service**:
   - Defines an ECS service `my-ecs-service` that runs the task `my-ecs-task`.
   - Configures network settings, load balancer, capacity provider strategy, and ensures new deployment upon updates.

## Configuration

```hcl
# Create ECS Cluster
resource "aws_ecs_cluster" "ecs_cluster" {
  name = "my-ecs-cluster"
}

# ECS Capacity Provider
resource "aws_ecs_capacity_provider" "ecs_capacity_provider" {
  name = "test1"
  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.ecs_asg.arn
    managed_scaling {
      maximum_scaling_step_size = 1000
      minimum_scaling_step_size = 1
      status                    = "ENABLED"
      target_capacity           = 3
    }
  }
}

# Associate Capacity Provider with Cluster
resource "aws_ecs_cluster_capacity_providers" "example" {
  cluster_name = aws_ecs_cluster.ecs_cluster.name
  capacity_providers = [aws_ecs_capacity_provider.ecs_capacity_provider.name]
  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = aws_ecs_capacity_provider.ecs_capacity_provider.name
  }
}

# ECS Task Definition
resource "aws_ecs_task_definition" "ecs_task_definition" {
  family             = "my-ecs-task"
  network_mode       = "awsvpc"
  execution_role_arn = "arn:aws:iam::654654464326:role/ecsTaskExecutionRole"
  cpu                = 2048
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64"
  }
  container_definitions = jsonencode([
    {
      name      = "my-test-project-java-image"
      image     = "public.ecr.aws/o9u1w8l7/my-test-project-java-image:test-v0.0.1"
      cpu       = 512
      memory    = 512
      essential = true
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "ecs_service" {
  name            = "my-ecs-service"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  task_definition = aws_ecs_task_definition.ecs_task_definition.arn
  desired_count   = 2

  network_configuration {
    subnets         = [aws_subnet.subnet.id, aws_subnet.subnet2.id]
    security_groups = [aws_security_group.security_group.id]
  }

  force_new_deployment = true
  placement_constraints {
    type = "distinctInstance"
  }

  triggers = {
    redeployment = timestamp()
  }

  capacity_provider_strategy {
    capacity_provider = aws_ecs_capacity_provider.ecs_capacity_provider.name
    weight            = 100
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.ecs_tg.arn
    container_name   = "my-test-project-java-image"
    container_port   = 80
  }

  depends_on = [aws_autoscaling_group.ecs_asg]
}

-----------------------------------------------------------------------------------------------------------------------------------
3) provider.tf
# Terraform Configuration

This Terraform configuration sets up the required providers for Docker and AWS.

## Providers Configuration

The configuration includes:

1. **Required Providers**:
   - Docker provider (`kreuzwerker/docker` version `~>2.20.0`).
   - AWS provider (`hashicorp/aws` version `~>4.0`).

2. **Provider Definitions**:
   - Initializes the Docker provider.
   - Initializes the AWS provider with a region specified by the `aws_region` variable.

## Configuration Code

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~>2.20.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "docker" {}

provider "aws" {
  region = var.aws_region
}

--------------------------------------------------------------------------------------------------------------------------------------
4) vpc.tf
# VPC Configuration

This Terraform configuration sets up a Virtual Private Cloud (VPC) along with subnets, an internet gateway, a route table, and a security group.

## Overview

The configuration includes:

1. **VPC**:
   - Creates a VPC with the specified CIDR block and enables DNS hostnames.

2. **Subnets**:
   - Creates two public subnets in different availability zones within the VPC.

3. **Internet Gateway**:
   - Creates an internet gateway and attaches it to the VPC.

4. **Route Table**:
   - Sets up a route table with a default route to the internet gateway.
   - Associates the route table with both subnets.

5. **Security Group**:
   - Creates a security group allowing all inbound and outbound traffic.

## Configuration Code

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = {
    name = "main"
  }
}

resource "aws_subnet" "subnet" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 1)
  map_public_ip_on_launch = true
  availability_zone       = "ap-south-1a"
}

resource "aws_subnet" "subnet2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 2)
  map_public_ip_on_launch = true
  availability_zone       = "ap-south-1b"
}

resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "internet_gateway"
  }
}

resource "aws_route_table" "route_table" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.internet_gateway.id
  }
}

resource "aws_route_table_association" "subnet_route" {
  subnet_id      = aws_subnet.subnet.id
  route_table_id = aws_route_table.route_table.id
}

resource "aws_route_table_association" "subnet2_route" {
  subnet_id      = aws_subnet.subnet2.id
  route_table_id = aws_route_table.route_table.id
}

resource "aws_security_group" "security_group" {
  name   = "ecs-security-group"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = -1
    self        = "false"
    cidr_blocks = ["0.0.0.0/0"]
    description = "any"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
-----------------------------------------------------------------------------------------------------------------------------------------------
5) variables.tf

# Terraform Variables

This Terraform configuration defines variables for specifying AWS region, VPC CIDR block, and availability zone.

## Variables

- **aws_region**: Specifies the AWS region where resources will be deployed. Default value is `ap-south-1`.

- **vpc_cidr**: Defines the CIDR block for the main VPC. Default value is `10.0.0.0/16`.

- **availability_zones**: Specifies the availability zone to use for subnet creation. Default value is `ap-south-1a`.

These variables provide flexibility and customization options when deploying infrastructure components such as VPCs and subnets in AWS.
------------------------------------------------------------------------------------------------------------------------------------
6) ecs.sh

# ECS Configuration Script

This Bash script (`ecs.sh`) is used to configure an Amazon ECS cluster by appending a cluster configuration to `/etc/ecs/ecs.config`.

## Script Explanation

The script performs the following actions:

- Appends `ECS_CLUSTER=my-ecs-cluster` to the `/etc/ecs/ecs.config` file.

This configuration line specifies the ECS cluster that this ECS agent should join.

## Usage

Ensure this script is executed on ECS instances to automatically configure them to join the specified ECS cluster (`my-ecs-cluster` in this example).

```bash
#!/bin/bash
echo ECS_CLUSTER=my-ecs-cluster >> /etc/ecs/ecs.config
----------------------------------------------------------------------------------------------------------------------------------------------
# Apply terraform Commands
terrafrom init
terrafrom plan
terrafrom apply
----------------------------------------------------------------------------------------------------------------------------------------
# Check all the resources created
and hit dns address of ELB on browser or add CName in domain provider




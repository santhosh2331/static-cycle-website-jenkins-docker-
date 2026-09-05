Cycle Website — AWS DevOps Project
A containerized cycle e-commerce website deployed on AWS using Terraform, Docker, GitHub Actions, Amazon RDS, Application Load Balancer and CloudWatch.
The project demonstrates an end-to-end DevOps workflow covering:
* Infrastructure as Code
* AWS networking
* Docker containerization
* CI/CD automation
* Staging and production environments
* Manual production approval
* Centralized logging
* Infrastructure and application monitoring
* CloudWatch dashboards and alerts.

Project Overview:
The goal of this project was to take an existing static cycle website and build a complete AWS-based DevOps deployment around it.
The infrastructure is provisioned using Terraform, the website runs inside Docker containers using Nginx, and GitHub Actions handles the CI/CD pipeline.
The application has separate staging and production environments.
CloudWatch is used for centralized logging and monitoring of the infrastructure and application traffic.
The application is deployed inside a custom AWS VPC
Note: The current website is a static frontend, so the PostgreSQL database is provisioned as part of the infrastructure but is not directly used by the website application.
AWS Infrastructure
Terraform provisions the following AWS resources.

Networking :
* Custom VPC
* Internet Gateway
* 2 public subnets
* 2 private subnets
* Public route table
* Private route table
* Route table associations
* Two Availability Zones

Compute :
Two Amazon EC2 instances are used:
Environment	EC2	                Purpose
Staging	        Amazon Linux + Docker	Application testing
Production	Amazon Linux + Docker	Production application
Both instances run the website using Docker and Nginx.

Load Balancer :
An Application Load Balancer (ALB) is deployed across the two public subnets.

The ALB:
* Accepts HTTP traffic on port 80
* Performs health checks
* Routes traffic to the staging application
* Provides a single entry point for the application

Database :
Amazon RDS PostgreSQL is deployed in the private subnets.
Configuration includes:
* PostgreSQL
* Private accessibility
* Dedicated DB subnet group
* Security group allowing PostgreSQL traffic from the EC2 security group
The database is intentionally not publicly accessible.

Security Groups :
Three security groups are used:
1. ALB Security Group Allows -->  Internet → ALB :80
2.EC2 Security Group Allows --> ALB → EC2 :8080 & SSH → EC2 :22
Port 8080 is also exposed for production testing in this implementation.
3.RDS Security Group Allows --> EC2 Security Group → RDS :5432. This prevents direct public access to PostgreSQL.

Docker
The website is packaged into a Docker image. The application uses Nginx as the web server.
The website files are copied into the Nginx web root.
The container listens on:
Container: 80
The EC2 host exposes:
EC2: 8080

Therefore:

ALB :80
    ↓
EC2 :8080
    ↓
Docker :80
    ↓
Nginx
    ↓
Website

Application Traffic Flow

For staging:

Internet
   │
   ▼
Application Load Balancer :80
   │
   ▼
Staging EC2 :8080
   │
   ▼
Docker Container :80
   │
   ▼
Nginx
   │
   ▼
Cycle Website
Production is currently accessed directly through the production EC2 instance:
Internet
   │
   ▼
Production EC2 :8080
   │
   ▼
Docker Container :80
   │
   ▼
Nginx
   │
   ▼
Cycle Website

The environments are kept separate so that changes can be validated in staging before production deployment.
Infrastructure as Code — Terraform
Terraform is used to provision and manage the AWS infrastructure.
The Terraform configuration is divided into logical files.

terraform/
│
├── versions.tf
├── provider.tf
├── variables.tf
├── terraform.tfvars
│
├── vpc.tf
├── subnets.tf
├── routes.tf
├── security_groups.tf
│
├── ec2.tf
├── alb.tf
├── rds.tf
│
├── cloudwatch.tf
└── outputs.tf

Important Terraform Concepts Used

Variables :
Environment-specific configuration is defined using variables.tf.
Examples:
aws_region
vpc_cidr
instance_type
db_name
db_username
db_password
key_name

State : Terraform state tracks the infrastructure that Terraform manages.
Outputs : Terraform outputs important infrastructure information such as:
* VPC ID
* ALB DNS name
* Website URL
* EC2 public IP
* Production public IP
* RDS endpoint
* RDS port

CI/CD Pipeline
GitHub Actions is used for continuous integration and deployment.
The workflow is defined in:

.github/
└── workflows/
    └── cicd.yaml

1. Pull Request Validation
When a Pull Request is created against main, GitHub Actions:
* Checks out the source code
* Verifies required website files exist
* Builds the Docker image
This helps identify basic issues before code is merged.

2. Docker Image Build
After code is merged into main, GitHub Actions builds the Docker image.
cycle-website:latest

3. Container Registry
The Docker image is pushed to:
GitHub Container Registry (GHCR)
ghcr.io/varshitha2303/cycle-website:latest
The EC2 servers pull the image from GHCR during deployment.
Staging Deployment
After the Docker image is pushed, the workflow automatically deploys it to the staging EC2.

The deployment process is:

Pull latest image
       ↓
Stop existing container
       ↓
Remove existing container
       ↓
Start new container
       ↓
Clean unused Docker images
       ↓
Verify application

Application verification is performed using:
curl -f http://localhost:8080
This ensures that the container is running and responding.

Production Deployment
Production deployment is intentionally separated from staging.
GitHub Actions uses a GitHub production environment with required reviewer approval.
This prevents an unreviewed change from being deployed directly to production.

Monitoring & Logging
Amazon CloudWatch is used for observability.
The CloudWatch Agent is installed on both EC2 instances.
It collects:
* Docker container logs
* Memory usage
* Disk usage

Centralized Logging

Docker logs are collected from:
/var/lib/docker/containers/*/*.log
Logs are sent to separate CloudWatch Log Groups:
/cycle-website/staging
/cycle-website/production
This provides centralized visibility into both environments without having to SSH into each server.

Metrics:
EC2
Infrastructure metrics include:
* CPU utilization
* Network traffic
* Memory utilization
* Disk utilization

ALB:
Application traffic metrics include:
* Request count
* Target response time
* HTTP 2XX responses
* HTTP 4XX responses
* HTTP 5XX responses

RDS
Database metrics include:
* CPU utilization
* Database connections
* Free storage space

Cloudwatch Dashboards

Two dashboards were created.
1. Infrastructure Dashboard
CycleWebsite-Infrastructure
Provides an overview of:
* EC2 health
* CPU usage
* Network traffic
* ALB traffic
* ALB errors
* RDS health
* Database connections
* Database storage

2. Application Dashboard
CycleWebsite-Application
Provides visibility into:
* Application request volume
* Response time
* HTTP status codes
* 4XX errors
* 5XX errors
* Staging logs
* Production logs
This separates infrastructure monitoring from application-level monitoring.

# Manual Deployment and High Availability of a Banking Application Using Amazon ECR, ECS, and Application Load Balancer (ALB)

## 📌 Project Overview

This project demonstrates how to manually deploy a containerized banking web application on AWS using Amazon Elastic Container Registry (ECR), Amazon Elastic Container Service (ECS) with AWS Fargate, and an Application Load Balancer (ALB).

The application is first containerized using Docker on an EC2 instance. The Docker image is then pushed to Amazon ECR. ECS Fargate pulls the image from ECR and runs multiple containers using an ECS Service. An Application Load Balancer distributes incoming traffic across healthy containers.

This architecture improves application availability, scalability, and reliability.

## 🎯 Project Objectives

- Containerize a banking web application using Docker.
- Store Docker images securely in Amazon ECR.
- Deploy containers using Amazon ECS with AWS Fargate.
- Configure an Application Load Balancer.
- Run multiple application tasks for high availability.
- Distribute incoming traffic across healthy ECS tasks.
- Monitor application health using ECS and CloudWatch.

## 🛠️ AWS Services and Technologies Used

| Service / Technology | Purpose |
|---|---|
| Amazon EC2 | Build the Docker image |
| Docker | Containerize the banking application |
| Git | Download application source code |
| Amazon ECR | Store Docker images |
| Amazon ECS | Manage and run containers |
| AWS Fargate | Run containers without managing servers |
| Application Load Balancer | Distribute incoming traffic |
| IAM | Manage AWS permissions |
| Amazon VPC | Provide networking infrastructure |
| Amazon CloudWatch | Monitor application logs |

## 🏗️ Architecture

### Architecture Diagram

```text
                    Internet Users
                          |
                          |
                          v
              Application Load Balancer
                     (HTTP : 80)
                          |
              -----------------------
              |                     |
              v                     v
        ECS Fargate Task 1    ECS Fargate Task 2
              |                     |
              |                     |
              -----------+-----------
                         |
                         v
                   Amazon ECR
               (Docker Image Repository)
                         ^
                         |
                    Docker Image
                         |
                         |
                   Amazon EC2
              (Docker Image Build)
                         |
                         |
                 Application Source
                     from GitHub
```

### Application Flow

1. Download the application source code from GitHub.
2. Build a Docker image on an EC2 instance.
3. Create an Amazon ECR repository.
4. Push the Docker image to ECR.
5. Create an ECS Fargate cluster.
6. Create an ECS task definition.
7. Configure an Application Load Balancer and target group.
8. Create an ECS service with two desired tasks.
9. Access the application using the ALB DNS name.

## 📋 Prerequisites

Before starting the project, ensure that you have:

- An AWS account.
- Basic knowledge of AWS and Docker.
- An EC2 instance running Amazon Linux.
- An IAM role with ECR permissions attached to EC2.
- Docker and Git installed on EC2.
- An Amazon VPC with public and private subnets.
- An Application Load Balancer security group.
- An ECS task execution role.

**AWS Region:** us-east-1 (N. Virginia)

**EC2 Instance Type:** t3.micro

**ECS Launch Type:** AWS Fargate

---

# 🚀 Step-by-Step Implementation

## Step 1: Launch an EC2 Instance

1. Log in to the AWS Management Console.
2. Navigate to EC2.
3. Click Launch Instance.
4. Enter the instance name.
5. Select Amazon Linux as the operating system.
6. Select the instance type `t3.micro`.
7. Select an existing key pair or create a new one.
8. Configure the security group to allow SSH access on port 22 from your IP address.
9. Launch the instance.

Connect to the EC2 instance using SSH.

## Step 2: Install Docker and Git

Update the system and install Docker.

```bash
sudo dnf update -y

sudo dnf install docker -y

sudo systemctl enable docker

sudo systemctl start docker

docker --version
```

Install Git.

```bash
sudo dnf install git -y

git --version
```

Verify that Docker is running.

```bash
sudo systemctl status docker
```

## Step 3: Download the Application Source Code

Clone the banking application repository.

```bash
git clone https://github.com/rohitsatpute627-pixel/ECR-ECS-project
```

Navigate to the application directory.

```bash
cd hdfcwebsite

cd insurance
```

Verify the application files.

```bash
ls -l
```

## Step 4: Build and Run the Docker Image

Build the Docker image.

```bash
docker build -t hdfcwebsite:latest .
```

Verify the image.

```bash
docker images
```

Run the Docker container.

```bash
docker run -itd \
  --name cont1 \
  -p 80:80 \
  hdfcwebsite:latest
```

Check the running container.

```bash
docker ps
```

Test the application.

```text
http://<EC2-PUBLIC-IP>
```

The application should be accessible through the EC2 public IP address.

## Step 5: Create an Amazon ECR Repository

Amazon ECR is a fully managed container image registry used to store Docker images securely.

1. Open the AWS Management Console.
2. Search for Amazon ECR.
3. Click Create Repository.
4. Select Private Repository.
5. Enter the repository name:

```text
hdfcwebsite
```

6. Click Create Repository.

Copy the ECR repository URI.

Example:

```text
<AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite
```

Replace the placeholder with your AWS account ID and actual repository URI.

## Step 6: Create an IAM Role for EC2

The EC2 instance needs permission to push Docker images to Amazon ECR.

1. Open IAM in the AWS Console.
2. Navigate to Roles.
3. Click Create Role.
4. Select AWS Service.
5. Choose EC2.
6. Attach the required ECR permissions.
7. Create the role.
8. Attach the role to your EC2 instance.

For this project, the ECR permissions can be provided using:

```text
AmazonEC2ContainerRegistryPowerUser
```

For production environments, use a least-privilege IAM policy instead of broad permissions.

## Step 7: Authenticate Docker with Amazon ECR

Set the AWS Region.

```bash
aws configure
```

Enter your AWS credentials if you are not using an attached IAM role.

Install the AWS CLI if it is not available.

Authenticate Docker with ECR.

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
<AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com
```

Replace `<AWS-ACCOUNT-ID>` with your AWS account ID.

## Step 8: Tag and Push the Docker Image to ECR

Tag the Docker image.

```bash
docker tag hdfcwebsite:latest \
<AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite:latest
```

Push the image to Amazon ECR.

```bash
docker push \
<AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite:latest
```

Verify the image in the AWS Console.

Navigate to:

```text
Amazon ECR → Repositories → hdfcwebsite → Images
```

The Docker image should be visible in the repository.

## Step 9: Create an ECS Fargate Cluster

1. Open Amazon ECS.
2. Navigate to Clusters.
3. Click Create Cluster.
4. Enter the cluster name:

```text
website-cluster
```

5. Select AWS Fargate as the infrastructure.
6. Keep the remaining settings as default.
7. Click Create.

Verify that the cluster status is Active.

## Step 10: Create an ECS Task Execution Role

The ECS task execution role allows ECS Fargate to pull container images from ECR and send logs to CloudWatch.

1. Open IAM.
2. Navigate to Roles.
3. Click Create Role.
4. Select AWS Service.
5. Select Elastic Container Service.
6. Choose Elastic Container Service Task.
7. Attach the policy:

```text
AmazonECSTaskExecutionRolePolicy
```

8. Enter the role name:

```text
ecsTaskExecutionRole
```

9. Create the role.

## Step 11: Create an ECS Task Definition

A task definition is a blueprint that defines how a container should run.

Navigate to:

```text
Amazon ECS → Task Definitions → Create New Task Definition
```

Configure the following settings.

| Configuration | Value |
|---|---|
| Task Definition Family | website-task |
| Launch Type | AWS Fargate |
| Operating System | Linux |
| Architecture | X86_64 |
| Network Mode | awsvpc |
| Task CPU | 0.25 vCPU |
| Task Memory | 0.5 GB |
| Task Execution Role | ecsTaskExecutionRole |
| Container Name | website-container |
| Container Image | Your ECR Image URI |
| Container Port | 80 |
| Protocol | TCP |
| Log Collection | CloudWatch Logs |
| Log Group | /ecs/website-task |

Replace the container image URI with the image URI copied from Amazon ECR.

Click Create to register the task definition.

## Step 12: Create a Target Group

The target group allows the Application Load Balancer to route traffic to ECS tasks.

1. Navigate to EC2.
2. Open Target Groups.
3. Click Create Target Group.
4. Configure the following:

| Configuration | Value |
|---|---|
| Target Type | IP |
| Target Group Name | website-tg |
| Protocol | HTTP |
| Port | 80 |
| Health Check Path | / |
| VPC | Your ECS VPC |

5. Click Create Target Group.

The target type must be IP because ECS Fargate tasks use the `awsvpc` network mode.

## Step 13: Create an Application Load Balancer

1. Navigate to EC2.
2. Open Load Balancers.
3. Click Create Load Balancer.
4. Select Application Load Balancer.

Configure the following settings.

| Configuration | Value |
|---|---|
| Load Balancer Name | website-alb |
| Scheme | Internet-facing |
| IP Address Type | IPv4 |
| VPC | Your ECS VPC |
| Availability Zones | Select 2 public subnets |
| Listener | HTTP : 80 |
| Default Action | Forward to website-tg |

Create a security group for the ALB.

Inbound rule:

```text
Type: HTTP
Port: 80
Source: 0.0.0.0/0
```

Click Create Load Balancer.

## Step 14: Create an ECS Service

An ECS service maintains the required number of running tasks.

1. Open Amazon ECS.
2. Navigate to Clusters.
3. Select `website-cluster`.
4. Click Create Service.

Configure the service.

| Configuration | Value |
|---|---|
| Launch Type | Fargate |
| Task Definition | website-task |
| Service Name | website-service |
| Desired Tasks | 2 |
| VPC | Your ECS VPC |
| Subnets | Private subnets recommended |
| Load Balancer | Application Load Balancer |
| ALB | website-alb |
| Listener | HTTP : 80 |
| Target Group | website-tg |

For tasks running in private subnets, ensure that the required network access is available for pulling images from ECR and sending logs to CloudWatch.

Disable public IP assignment when using private subnets.

Create the ECS service.

## Step 15: Configure Security Groups

Security groups control the network traffic allowed between the ALB and ECS tasks.

### ALB Security Group

Inbound:

```text
HTTP - Port 80 - Source: 0.0.0.0/0
```

Outbound:

```text
Allow traffic to ECS tasks on port 80
```

### ECS Task Security Group

Inbound:

```text
HTTP - Port 80 - Source: ALB Security Group
```

Outbound:

```text
Allow required outbound traffic
```

This configuration ensures that users access the application through the ALB instead of directly accessing the ECS tasks.

## Step 16: Verify the Deployment

Wait approximately 2–5 minutes for the ECS service to start the tasks.

Navigate to:

```text
ECS → Clusters → website-cluster → Services
```

Verify the service status.

```text
Desired Tasks: 2
Running Tasks: 2
Pending Tasks: 0
```

Navigate to:

```text
EC2 → Target Groups → website-tg → Targets
```

Verify that the registered ECS task IP addresses show a healthy status.

## Step 17: Access the Banking Application

Navigate to:

```text
EC2 → Load Balancers → website-alb
```

Copy the DNS name of the Application Load Balancer.

Open the DNS name in your browser.

```text
http://<ALB-DNS-NAME>
```

The banking application should load successfully.

The ALB distributes incoming requests across the healthy ECS tasks.

---

# 🔍 High Availability

High availability is achieved by running multiple ECS tasks behind an Application Load Balancer.

In this project:

- Two ECS tasks are configured using an ECS service.
- The ALB distributes traffic across healthy tasks.
- ECS maintains the desired number of running tasks.
- If a task fails, ECS can launch a replacement task.
- Tasks can be deployed across multiple Availability Zones.

This reduces the risk of application downtime caused by a single container failure.

# 📊 Monitoring and Troubleshooting

## Monitor ECS Tasks

Navigate to:

```text
ECS → Clusters → website-cluster → Services → website-service
```

Check:

- Running task count.
- Task health.
- Service events.
- Deployment status.

## Monitor CloudWatch Logs

Navigate to:

```text
CloudWatch → Logs → Log Groups → /ecs/website-task
```

Review application logs to troubleshoot container issues.

## Common Troubleshooting Scenarios

| Problem | Possible Solution |
|---|---|
| ECS task fails to start | Check task definition and container image URI |
| ECR image pull error | Verify task execution role permissions |
| ALB returns 503 | Check target group health and ECS task status |
| Target group unhealthy | Verify port 80 and health check path |
| Website not loading | Verify ALB listener and security groups |
| ECS tasks remain pending | Check service events and available resources |

# 🧹 Cleanup

To avoid unnecessary AWS charges, delete the resources after completing the project.

1. Delete the ECS service.
2. Delete the ECS cluster.
3. Deregister the ECS task definition revisions.
4. Delete the Application Load Balancer.
5. Delete the target group.
6. Delete the ECR repository and its images.
7. Delete the IAM roles created for this project.
8. Terminate the EC2 instance.
9. Delete any unused networking resources created for this project.

**Important:** Review your AWS resources before deletion. Do not delete shared VPCs, IAM roles, or networking resources used by other applications.

# 📚 Key Learnings

Through this project, I learned:

- How to build and run Docker containers.
- How to push Docker images to Amazon ECR.
- How to deploy containers using ECS Fargate.
- How to create ECS task definitions and services.
- How to configure an Application Load Balancer.
- How to configure target groups and health checks.
- How to implement high availability using multiple ECS tasks.
- How to troubleshoot container deployment and networking issues.

# 👨‍💻 Author

**Rohit Satpute**

Cloud Engineer | AWS | DevOps

- GitHub: [rohitsatpute627-pixel](https://github.com/rohitsatpute627-pixel)
- LinkedIn: [Rohit Satpute](https://www.linkedin.com/in/rohit-satpute-8370a0211/)

---

⭐ If you find this project useful, feel free to star the repository.

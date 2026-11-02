# Manual Deployment and High Availability of a Banking Application Using Amazon ECR, ECS, and ALB

## 1. Project Overview

This project demonstrates how to containerize a web application using Docker, store the Docker image in **Amazon Elastic Container Registry (ECR)**, and deploy it on **Amazon Elastic Container Service (ECS) with AWS Fargate**.

An **Application Load Balancer (ALB)** distributes incoming HTTP traffic across multiple ECS tasks. The ECS service is configured with a desired count of two tasks to improve availability.

> **Note:** This is a learning project. Use a test application and follow your organization's security practices before deploying any real banking or production workload.

## 2. Project Objectives

- Build a Docker image for the web application.
- Store and version the image in a private Amazon ECR repository.
- Run the application using ECS Fargate.
- Configure an ECS service to maintain two running tasks.
- Distribute user traffic through an internet-facing Application Load Balancer.
- Check application availability using ALB DNS and target health.

## 3. Architecture

```text
User / Browser
      |
      | HTTP : 80
      v
Application Load Balancer (Internet-facing)
      |
      v
Target Group (IP targets, HTTP : 80)
      |
      +---------------------+
      |                     |
      v                     v
ECS Fargate Task 1     ECS Fargate Task 2
(Website container)    (Website container)
      |                     |
      +----------+----------+
                 |
                 v
       Docker image pulled from
          Amazon ECR Repository

EC2 build machine:
Git source code -> Docker build -> Amazon ECR push
```

## 4. AWS Services and Tools

| Service / Tool | Purpose |
|---|---|
| Amazon EC2 | Temporary machine used to build and push the Docker image |
| Docker | Packages the web application into a container image |
| Git | Downloads the application source code |
| Amazon ECR | Private repository for storing the Docker image |
| AWS IAM | Grants the EC2 machine and ECS tasks required permissions |
| Amazon ECS | Runs and manages the containers |
| AWS Fargate | Runs ECS tasks without managing EC2 worker servers |
| Application Load Balancer | Routes HTTP requests to healthy ECS tasks |
| Amazon CloudWatch Logs | Collects container logs |

## 5. Prerequisites

Before starting, make sure you have:

- An AWS account and access to the AWS Management Console.
- Permission to create EC2, IAM roles, ECR, ECS, target groups, and load balancers.
- An AWS Region selected. This project uses **US East (N. Virginia) — `us-east-1`**.
- Basic knowledge of Linux, Git, Docker, and AWS networking.
- A Dockerfile and application source code.

**Important:** AWS resources can incur charges. Review pricing and delete resources when you finish the project.

## 6. Step-by-Step Deployment

### Step 1: Launch an EC2 build machine and install Docker and Git

1. Open the AWS Console and go to **EC2 → Instances → Launch instance**.
2. Select an Amazon Linux AMI and instance type such as `t3.micro` (if available and suitable).
3. Configure a key pair and security group. Allow SSH (port 22) only from your own IP address.
4. Launch the instance and connect to it using SSH.
5. Install and start Docker:

```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

6. Install Git:

```bash
sudo dnf install git -y
git --version
```

### Step 2: Download the application and build the Docker image

Clone the sample application repository and enter its application directory:

```bash
git clone https://github.com/rohitsatpute627-pixel/ECR-ECS-project
cd hdfcwebsite
cd insurance
```

Build the image:

```bash
docker build -t hdfcwebsite:latest .
```

Check that the image exists:

```bash
docker images
```

### Step 3: Run and test the container locally

Start the container and map host port 80 to container port 80:

```bash
docker run -d --name cont1 -p 80:80 hdfcwebsite:latest
```

Check the container:

```bash
docker ps
docker logs cont1
```

Open the EC2 instance's public IPv4 address in a browser using HTTP. For this temporary test, the instance security group must allow inbound HTTP (port 80) from your intended test source. Avoid exposing the build machine unnecessarily; remove this rule or stop the test container after validation.

### Step 4: Create an Amazon ECR repository

1. Open **Amazon ECR → Private repositories → Create repository**.
2. Set the repository name to `hdfcwebsite`.
3. Create the repository.
4. Open the repository and copy its URI. It will look similar to:

```text
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite
```

Replace `<ACCOUNT_ID>` with your AWS account ID.

### Step 5: Create and attach an EC2 IAM role for ECR

1. Go to **IAM → Roles → Create role**.
2. Select **AWS service → EC2** as the trusted entity.
3. Attach an ECR permission policy appropriate for pushing images. The project notes mention `AmazonEC2ContainerRegistryPowerUser` or `AmazonEC2ContainerRegistryFullAccess`; prefer least-privilege permissions for real workloads.
4. Name the role, for example `EC2-ECR-Push-Role`.
5. Attach the role to the EC2 build instance using **EC2 → Instance → Actions → Security → Modify IAM role**.

The EC2 instance role lets the instance authenticate to ECR without putting long-lived AWS access keys in the machine.

### Step 6: Authenticate Docker and push the image to ECR

Use the ECR repository's **View push commands** instructions in the AWS Console. The commands generally follow this pattern:

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

Tag the local image:

```bash
docker tag hdfcwebsite:latest \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite:latest
```

Push it:

```bash
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hdfcwebsite:latest
```

Verify that the image appears under **ECR → Repositories → hdfcwebsite → Images**.

> The AWS CLI must be installed on the EC2 machine. If it is missing, install AWS CLI v2 using the official AWS instructions before running the ECR login command.

### Step 7: Create an ECS Fargate cluster

1. Open **Amazon ECS → Clusters → Create cluster**.
2. Cluster name: `website-cluster`.
3. Choose AWS Fargate / serverless infrastructure.
4. Keep other settings at their defaults unless your environment requires changes.
5. Create the cluster and confirm its status is **Active**.

Fargate manages the compute infrastructure, so you do not need to create ECS container instances or an Auto Scaling group for this setup.

### Step 8: Create the ECS task execution role

1. Go to **IAM → Roles → Create role**.
2. Select **AWS service → Elastic Container Service → Elastic Container Service Task**.
3. Attach `AmazonECSTaskExecutionRolePolicy`.
4. Name the role `ecsTaskExecutionRole`.
5. Create the role.

This execution role allows ECS to pull the private image from ECR and send logs to CloudWatch Logs.

### Step 9: Create an ECS task definition

1. Open **ECS → Task definitions → Create new task definition**.
2. Configure the task:

| Setting | Example value |
|---|---|
| Task definition family | `website-task` |
| Launch type / infrastructure | AWS Fargate |
| OS / architecture | Linux / X86_64 |
| Network mode | `awsvpc` |
| Task CPU | `0.25 vCPU` |
| Task memory | `0.5 GB` |
| Task execution role | `ecsTaskExecutionRole` |
| Container name | `website-container` |
| Image URI | Your ECR image URI with tag |
| Container port | `80` |
| Protocol | TCP |
| CloudWatch log group | `/ecs/website-task` |

3. Enable CloudWatch log collection.
4. Create the task definition.

Make sure the container serves the website on port 80 and that the image URI points to the image you pushed to ECR.

### Step 10: Create a target group

1. Go to **EC2 → Target Groups → Create target group**.
2. Target type: **IP addresses** (required for Fargate tasks using `awsvpc`).
3. Name: `website-tg`.
4. Protocol: HTTP; port: `80`.
5. Select the VPC used by the ECS service.
6. Health check path: `/`.
7. Create the target group.

### Step 11: Create an Application Load Balancer

1. Go to **EC2 → Load Balancers → Create load balancer**.
2. Select **Application Load Balancer**.
3. Name: `website-alb`.
4. Scheme: **Internet-facing**.
5. IP address type: IPv4.
6. Select the VPC and at least two public subnets in different Availability Zones.
7. Attach a security group that allows inbound HTTP (port 80) from the intended users.
8. Add a listener: HTTP : 80.
9. Set its default action to forward to `website-tg`.
10. Create the load balancer.

### Step 12: Create the ECS service

1. Open **ECS → Clusters → website-cluster → Services → Create**.
2. Compute option: Fargate.
3. Select the task definition created above.
4. Service name: `website-service`.
5. Desired tasks: **2**.
6. Select the VPC and subnets:
   - Recommended: private subnets with suitable outbound access (for example, NAT) so tasks can pull images and access required AWS services.
   - For a lab, public subnets may be used if configured appropriately.
7. Select a task security group that allows inbound TCP port 80 **from the ALB security group**.
8. For private subnets, keep public IP assignment disabled. For public-subnet testing, configure public IP assignment as required.
9. Configure the service to use the Application Load Balancer:
   - Load balancer: `website-alb`
   - Listener: HTTP : 80
   - Target group: `website-tg`
   - Container: `website-container`, port 80
10. Create the service.

### Step 13: Verify the deployment

1. Open **ECS → Clusters → website-cluster → Services → website-service**.
2. Wait for the tasks to start and pass their health checks.
3. Confirm that the service has **2 running tasks** and no pending tasks.
4. Open **EC2 → Target Groups → website-tg → Targets** and check that the targets are healthy.
5. Open **EC2 → Load Balancers → website-alb** and copy its DNS name.
6. Visit the DNS name in a browser:

```text
http://<ALB-DNS-NAME>
```

The website should load through the ALB. The ALB routes requests to healthy ECS tasks.

## 7. Troubleshooting

| Issue | What to check |
|---|---|
| Docker build fails | Confirm you are in the directory containing the `Dockerfile`; check the file name and build context. |
| ECR login or push fails | Check the AWS Region, repository URI, EC2 IAM role permissions, and AWS CLI installation. |
| ECS task cannot pull image | Confirm the ECR image URI/tag and task execution role; check task networking and ECR access. |
| ECS tasks keep stopping | Review ECS task stopped reason and CloudWatch container logs. |
| ALB target is unhealthy | Confirm target group uses IP targets, health check path is correct, container listens on port 80, and security groups allow ALB-to-task traffic. |
| ALB DNS does not open | Check ALB listener on port 80, ALB security group inbound rule, public subnet routing, and target health. |
| Only one task is running | Check desired count, available subnet IP addresses, task events, and Fargate capacity/errors. |

## 8. Key Concepts Learned

- Docker image creation and container testing.
- Amazon ECR as a private container image registry.
- ECS cluster, task definition, task, and service.
- AWS Fargate serverless container deployment.
- Application Load Balancer and target group health checks.
- IAM roles for EC2 image push and ECS image pull.
- Basic troubleshooting of container deployments and AWS networking.

## 9. Cleanup — Avoid Unnecessary AWS Charges

When you finish testing, delete resources you no longer need. A typical cleanup order is:

1. Delete the ECS service (or set desired count to zero, then delete it).
2. Delete the ECS cluster after its services are removed.
3. Delete the ALB.
4. Delete the target group after it is no longer associated with the ALB.
5. Delete the ECR repository and its images if you no longer need them.
6. Terminate the EC2 build instance.
7. Remove unused IAM roles and security groups, after checking that nothing else uses them.

Check the AWS console for any remaining resources that may incur charges.

## 10. Resume Project Description

**Manual Deployment and High Availability of a Banking Application Using Amazon ECR, ECS, and ALB**  
Containerized a web application using Docker, pushed the image to Amazon ECR, and deployed it on AWS ECS Fargate. Configured an ECS service with two tasks behind an internet-facing Application Load Balancer, using target group health checks to route traffic to healthy containers.

## 11. Repository Structure

```text
ECR-ECS-project/
├── README.md
└── (add your Dockerfile, application files, screenshots, or diagrams here if applicable)
```

---

**Author:** ROHIT SATPUTE


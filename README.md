# Overview

This repository contains a React frontend, and an Express backend that the frontend connects to.

# Objective

Deploy the frontend and backend to somewhere publicly accessible over the internet. The AWS Free Tier should be more than sufficient to run this project, but you may use any platform and tooling you'd like for your solution.

Fork this repo as a base. You may change any code in this repository to suit the infrastructure you build in this code challenge.

# Submission

1. A github repo that has been forked from this repo with all your code.
2. Modify this README file with instructions for:

- Any tools needed to deploy your infrastructure
- All the steps needed to repeat your deployment process
- URLs to the your deployed frontend.

# Evaluation

You will be evaluated on the ease to replicate your infrastructure. This is a combination of quality of the instructions, as well as any scripts to automate the overall setup process.

# Setup your environment

Install nodejs. Binaries and installers can be found on nodejs.org.
https://nodejs.org/en/download/

For macOS or Linux, Nodejs can usually be found in your preferred package manager.
https://nodejs.org/en/download/package-manager/

Depending on the Linux distribution, the Node Package Manager `npm` may need to be installed separately.

# Running the project

The backend and the frontend will need to run on separate processes. The backend should be started first.

```
cd backend
npm ci
npm start
```

The backend should response to a GET request on `localhost:8080`.

With the backend started, the frontend can be started.

```
cd frontend
npm ci
npm start
```

The frontend can be accessed at `localhost:3000`. If the frontend successfully connects to the backend, a message saying "SUCCESS" followed by a guid should be displayed on the screen. If the connection failed, an error message will be displayed on the screen.

# Configuration

The frontend has a configuration file at `frontend/src/config.js` that defines the URL to call the backend. This URL is used on `frontend/src/App.js#12`, where the front end will make the GET call during the initial load of the page.

The backend has a configuration file at `backend/config.js` that defines the host that the frontend will be calling from. This URL is used in the `Access-Control-Allow-Origin` CORS header, read in `backend/index.js#14`.

# Optional Extras

The core requirement for this challenge is to get the provided application up and running for consumption over the public internet. That being said, there are some opportunities in this code challenge to demonstrate your skill sets that are above and beyond the core requirement.

A few examples of extras for this coding challenge:

1. Dockerizing the application
2. Scripts to set up the infrastructure
3. Providing a pipeline for the application deployment
4. Running the application in a serverless environment

This is not an exhaustive list of extra features that could be added to this code challenge. At the end of the day, this section is for you to demonstrate any skills you want to show that’s not captured in the core requirement.

# Solution Overview

The application was deployed to AWS using Docker, Terraform, Jenkins, Amazon ECR, Amazon ECS Fargate, and an Application Load Balancer.

The React frontend and Express backend run as separate ECS services. Both services run in private subnets. The public Application Load Balancer receives requests from the internet and forwards them to the correct ECS service.

The ALB uses the following routing rules:

1. `/` and other default paths are forwarded to the frontend service on port `3000`.
2. `/api` and `/api/*` are forwarded to the backend service on port `8080`.

The application is available at:
http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com

The current deployment uses HTTP. HTTPS can be added later by configuring a domain, requesting an AWS Certificate Manager certificate, and creating an HTTPS listener on the load balancer.

# Architecture

The AWS infrastructure includes:

1. One VPC with the CIDR range `10.0.0.0/16`
2. Two public subnets in `us-east-2a` and `us-east-2b`
3. Two private subnets in `us-east-2a` and `us-east-2b`
4. One internet gateway
5. One NAT gateway
6. One Application Load Balancer
7. Separate frontend and backend target groups
8. One ECS Fargate cluster
9. Separate frontend and backend ECS services
10. Separate frontend and backend ECR repositories
11. CloudWatch log groups for both services
12. ECS Service Auto Scaling
13. One EC2 instance running Jenkins

The request flow is:

```
Internet
   |
Application Load Balancer
   |
   |-- Default traffic --> Frontend Target Group --> Frontend ECS Service
   |
   |-- /api traffic ----> Backend Target Group ---> Backend ECS Service
```

# Tools Needed

The following tools are required to repeat this deployment:

1. Git
2. GitHub account
3. Node.js and npm
4. Docker
5. AWS CLI v2
6. Terraform
7. AWS account
8. Jenkins
9. Siege for load testing

Verify the local tools:

```
git --version
node --version
npm --version
docker --version
aws --version
terraform --version
```

# AWS CLI Setup

Configure the AWS CLI:

```
aws configure
```

Enter the IAM access key, secret access key, default region, and output format when prompted. This deployment uses the region `us-east-2`.

Verify the AWS identity:

```
aws sts get-caller-identity
```

The account number and ARN returned by this command should match the AWS account and IAM user intended for the deployment.

# Application Configuration for AWS

The frontend URL is configured in `frontend/src/config.js`.

```
const API_URL =
  'http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com/api';

export default API_URL;
```

The `/api` path should only be added once. If `frontend/src/App.js` already adds `/api`, the base URL in `config.js` should not include it.

The backend CORS origin is configured in `backend/config.js`.

```
module.exports = {
  CORS_ORIGIN:
    'http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com',
};
```

# Docker Setup

The frontend and backend each have a Dockerfile.

The frontend container listens on port `3000`. The final lines of the frontend Dockerfile are:

```
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

The `-l` option must include the hyphen. Without the hyphen, the container exits because `serve` interprets the remaining values as multiple path arguments.

Build the images locally:

```
docker build -t frontend:latest ./frontend
docker build -t backend:latest ./backend
```

# Terraform Guide

All Terraform configuration files are stored inside the `terraform` directory.

The root Terraform variables include:

1. AWS region
2. Project name
3. Environment
4. VPC CIDR
5. Frontend and backend ports
6. Frontend and backend container images
7. Fargate CPU and memory values
8. Minimum, desired, and maximum task counts
9. CPU target for autoscaling

Before applying the infrastructure, confirm that the EC2 key pair referenced by `jenkins.tf` exists in `us-east-2`.

Enter the Terraform directory:

```
cd terraform
```

Initialize Terraform:

```
terraform init
```

Format the Terraform files:

```
terraform fmt
```

Validate the configuration:

```
terraform validate
```

Preview the resources:

```
terraform plan
```

Create the infrastructure:

```
terraform apply
```

Enter `yes` when prompted.

Display Terraform outputs:

```
terraform output
```

Terraform state files and provider binaries must not be committed to GitHub. The `.gitignore` file should include:

```
**/.terraform/**
*.tfstate
*.tfstate.*
*.tfplan
tfplan
*.tfvars
*.tfvars.json
**/node_modules/
frontend/build/
```

The `.terraform.lock.hcl` file should remain tracked by Git.

# Verify the AWS Infrastructure

After `terraform apply` completes, verify the following resources in AWS:

1. The VPC and four subnets exist.
2. The frontend and backend ECR repositories exist.
3. The ECS cluster contains the frontend and backend services.
4. The Application Load Balancer is active.
5. The frontend and backend target groups report healthy targets.
6. CloudWatch receives logs from both ECS services.

Check the ECS services:

```
aws ecs describe-services \
  --cluster devops-challenge-cluster \
  --services devops-challenge-frontend-service devops-challenge-backend-service \
  --region us-east-2
```

# Jenkins Setup

Move the downloaded EC2 private key to the SSH directory on the local computer and restrict its permissions:

```
mv ~/Downloads/1PU.pem ~/.ssh/
chmod 400 ~/.ssh/1PU.pem
```

Connect to the Jenkins EC2 instance:

```
ssh -i ~/.ssh/1PU.pem ec2-user@JENKINS_PUBLIC_IP
```

Install and start Docker on Amazon Linux 2023:

```
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Exit and reconnect so the new Docker group membership takes effect:

```
exit
```

Verify Docker:

```
docker ps
getent group docker
```

Create persistent storage for Jenkins:

```
sudo mkdir -p /var/jenkins_home
sudo chown -R 1000:1000 /var/jenkins_home
sudo chmod 755 /var/jenkins_home
```

Record the Docker group number returned by `getent group docker`. The group number was `994` in this environment, but it should be verified on every server.

Start Jenkins:

```
docker run -d --name jenkins --restart unless-stopped -p 8080:8080 -v /var/jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --group-add 994 jenkins/jenkins:lts
```

Verify that Jenkins is running:

```
docker ps
docker logs -f jenkins
```

Retrieve the initial Jenkins password:

```
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open Jenkins in a browser:

```
http://JENKINS_PUBLIC_IP:8080
```

Ports `22` and `8080` should be restricted to a trusted administrator IP address in the Jenkins security group.

# Install Docker and AWS CLI in Jenkins

Enter the Jenkins container as root:

```
docker exec -u root -it jenkins bash
```

Install the Docker CLI:

```
apt-get update
apt-get install -y docker.io
docker --version
docker ps
```

Install AWS CLI v2 if it is not already installed:

```
apt-get install -y curl unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip /tmp/awscliv2.zip -d /tmp
/tmp/aws/install
aws --version
```

Installing these tools interactively is suitable for this exercise. A production environment should build a custom Jenkins Docker image so the tools remain available whenever the container is recreated.

# Jenkins Plugins

Install the following plugins from `Manage Jenkins`, then `Plugins`:

1. Pipeline
2. Git
3. GitHub
4. Credentials Binding
5. AWS Credentials
6. Workspace Cleanup

# Jenkins Credentials

Create the credentials under `Manage Jenkins`, then `Credentials`, then `System`, then `Global credentials`.

Create a GitHub credential using the ID:

```
github-PAT
```

Create an AWS credential using the ID:

```
aws-PAT
```

Credential IDs are case-sensitive and must exactly match the IDs referenced in the Jenkinsfile. Credentials must not be written directly into the Jenkinsfile or committed to GitHub.

# Jenkins Pipeline

Create a Pipeline job and use the following settings:

1. Definition: Pipeline script from SCM
2. SCM: Git
3. Repository URL: `https://github.com/uukomadu/devops-code-challenge1`
4. Credentials: `github-PAT`
5. Branch: `*/main`
6. Script Path: `Jenkinsfile`

The pipeline performs the following stages:

1. Checkout SCM
2. Checkout code
3. Build the frontend and backend Docker images
4. Authenticate to Amazon ECR
5. Tag and push images to ECR
6. Force new frontend and backend ECS deployments
7. Clean the Jenkins workspace

The AWS values used by the pipeline are:

```
AWS Region: us-east-2
ECS Cluster: devops-challenge-cluster
Frontend Service: devops-challenge-frontend-service
Backend Service: devops-challenge-backend-service
```

# GitHub Webhook

Open the GitHub repository settings and add a webhook.

Use the following configuration:

1. Payload URL: `http://JENKINS_PUBLIC_IP:8080/github-webhook/`
2. Content type: `application/json`
3. Event: Push events

Enable the GitHub hook trigger in the Jenkins pipeline job. A push to the `main` branch should start the pipeline automatically.

# Verify the Deployment

A successful Jenkins pipeline confirms that the images were built, pushed to ECR, and that ECS accepted the deployment requests. ECS may still require several minutes to replace its tasks and complete the ALB health checks.

Wait for both services to stabilize:

```
aws ecs wait services-stable \
  --cluster devops-challenge-cluster \
  --services devops-challenge-frontend-service devops-challenge-backend-service \
  --region us-east-2
```

Open the deployed frontend:
http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com

A successful deployment displays `SUCCESS` followed by a guid returned by the backend.

# Install Siege

Siege was compiled from source because it was not available in the default Amazon Linux 2023 repositories.

Install the required build tools:

```
sudo dnf install -y gcc make openssl-devel zlib-devel tar gzip
```

Download and extract Siege:

```
cd /tmp
curl -LO https://download.joedog.org/siege/siege-latest.tar.gz
tar -xzf siege-latest.tar.gz
ls -ld siege-*
```

Enter the extracted directory using the exact directory name displayed by `ls`:

```
cd siege-4.1.7
```

Compile and install Siege:

```
./configure
make
sudo make install
```

Verify the installation:

```
siege --version
```

# Scaling Test

The ECS services use target-tracking autoscaling with the following settings:

1. Minimum tasks: `1`
2. Desired tasks: `1`
3. Maximum tasks: `4`
4. Target average CPU utilization: `50%`

Start with a small test:

```
siege -c 10 -t 30S http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com/
```

The larger scaling test uses:

```
siege -c 250 -t 2M http://devops-challenge-alb-1270001619.us-east-2.elb.amazonaws.com/
```

Only one protocol should appear in the URL. This deployment currently uses `http://` and does not have an HTTPS listener.

Monitor the task count during the test:

```
aws ecs describe-services \
  --cluster devops-challenge-cluster \
  --services devops-challenge-frontend-service devops-challenge-backend-service \
  --region us-east-2 \
  --query 'services[*].{Service:serviceName,Desired:desiredCount,Running:runningCount,Pending:pendingCount}' \
  --output table
```

CloudWatch can also be used to monitor `CPUUtilization` for the frontend and backend ECS services.

# Scaling Results

The deployment and scaling test preparation produced the following verified results:

1. The Application Load Balancer was publicly accessible.
2. The frontend target became healthy.
3. The frontend successfully called the backend through the `/api` ALB rule.
4. Jenkins built and pushed both images to ECR.
5. Jenkins successfully forced new ECS deployments.
6. Siege was installed successfully on Amazon Linux 2023.
7. The Siege command was corrected to use one `http://` protocol.

The following measurements should be copied from the successful Siege output, ECS service task counts, and CloudWatch metrics:

```
Frontend tasks before test:
Frontend peak tasks:
Frontend tasks after cooldown:
Backend tasks before test:
Backend peak tasks:
Backend tasks after cooldown:
Frontend peak CPU utilization:
Backend peak CPU utilization:
Siege availability:
Siege transaction rate:
Siege average response time:
```

Actual measurements should be recorded before claiming that the service scaled from one task to a higher task count. High request concurrency does not always produce enough CPU utilization to exceed the configured target.

# Troubleshooting

If the ALB displays `503 Service Temporarily Unavailable`, check the ECS service running count, service events, target group health, and CloudWatch logs. A 503 normally means that the ALB has no healthy target available.

View frontend logs:

```
aws logs tail /ecs/devops-challenge-frontend --since 30m --region us-east-2
```

If the browser displays the following error, the frontend expected JSON but received an HTML page:

```
Unexpected token '<', "<!doctype" is not valid JSON
```

Verify that the frontend request contains `/api` exactly once and that the ALB sends both `/api` and `/api/*` to the backend target group.

If Jenkins cannot find `frontend`, `backend`, or `Jenkinsfile`, verify that the files are tracked by Git:

```
git ls-files frontend
git ls-files backend
git ls-files Jenkinsfile
```

# Security Improvements

The following changes are recommended for a production deployment:

1. Configure HTTPS using a domain, ACM certificate, and ALB HTTPS listener.
2. Redirect HTTP traffic to HTTPS.
3. Replace long-lived Jenkins AWS keys with an EC2 IAM role.
4. Apply least-privilege IAM permissions.
5. Restrict Jenkins ports `22` and `8080` to trusted IP addresses.
6. Store Terraform state in an encrypted remote S3 backend with state locking.
7. Use immutable image tags such as the Git commit SHA instead of only `latest`.
8. Add CloudWatch alarms for high CPU, unhealthy targets, and failed deployments.
9. Upgrade the unsupported Node.js base images.

# Cleanup

Destroy the Terraform-managed resources when they are no longer needed:

```
cd terraform
terraform destroy
```

Review the destroy plan and enter `yes` when prompted. Confirm that NAT gateways, Elastic IP addresses, load balancers, EC2 instances, ECS services, ECR images, and CloudWatch log groups have been removed to avoid additional AWS charges.

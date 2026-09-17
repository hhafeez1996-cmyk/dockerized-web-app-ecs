# Dockerized Web Application Deployment on AWS ECS

## 1. Project Overview

This project demonstrates the deployment of a containerized web application on Amazon ECS using AWS Fargate.

The application was containerized using Docker, stored in Amazon ECR, and deployed as an ECS Fargate service behind an Application Load Balancer (ALB).

The deployment was designed with multiple Availability Zones, private ECS task subnets, security groups, IAM permissions, AWS Secrets Manager, CloudWatch Logs, health checks, and multiple running tasks.

---

## 2. Architecture Overview

The application architecture consists of:

**Internet → Application Load Balancer → Target Group → ECS Service → ECS Fargate Tasks → Containerized React Application**

```text
                         Internet / User
                                │
                                │ HTTP :80
                                ▼
                    ┌─────────────────────────┐
                    │ Application Load         │
                    │ Balancer (myweb-ALB)     │
                    │ Public Subnets           │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Target Group (myweb-tg)  │
                    │ Health Check: /          │
                    │ Port: 3000               │
                    └────────────┬────────────┘
                                 │
                         ECS Service
                      Desired Count = 2
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌──────────────────┐      ┌──────────────────┐
          │ Fargate Task 1   │      │ Fargate Task 2   │
          │ Private Subnet   │      │ Private Subnet   │
          │ eu-north-1a      │      │ eu-north-1b      │
          │ Port 3000        │      │ Port 3000        │
          └──────────────────┘      └──────────────────┘
                    │                         │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    Containerized React App


Supporting AWS Services
────────────────────────────────────────────────────

 Amazon ECR ──────────────► ECS Fargate Tasks
   myweb:latest               Docker Image

 Secrets Manager ─────────► ECS Task Execution Role
   myweb/app-secret           IAM

 ECS Fargate Tasks ───────► CloudWatch Logs

 Private Subnets
       │
       ▼
 Private Route Table
       │
       ▼
 NAT Gateway
       │
       ▼
 Outbound Internet Connectivity
```

The application uses an internet-facing Application Load Balancer as the public entry point. The ALB forwards traffic to the healthy ECS Fargate tasks running in private subnets across two Availability Zones.

The ECS tasks do not have public IP addresses and accept application traffic only from the Application Load Balancer security group.

Supporting AWS services:

* Amazon ECR – container image storage
* Amazon ECS – container orchestration
* AWS Fargate – serverless container compute
* Application Load Balancer – public access and traffic distribution
* Amazon VPC – networking
* Security Groups – network access control
* AWS Secrets Manager – secure secret storage
* IAM – access control
* Amazon CloudWatch Logs – application logging

The ECS tasks were deployed across two Availability Zones for availability.


## 3. AWS Services Used

* Amazon ECR
* Amazon ECS
* AWS Fargate
* Amazon VPC
* Application Load Balancer
* Target Group
* Security Groups
* AWS IAM
* AWS Secrets Manager
* Amazon CloudWatch Logs

---

## 4. Application Containerization

The web application is a React application located inside the `myapp` directory.

The Dockerfile uses Node.js 20 and runs the React application on port 3000.

Main Docker configuration:

```dockerfile
FROM node:20

WORKDIR /myapp

COPY . .

RUN npm install

EXPOSE 3000

CMD ["npm", "start"]
```

The application was tested locally before deployment.

---

## 5. Amazon ECR

A private Amazon ECR repository named `myweb` was created in the `eu-north-1` (Stockholm) AWS Region.

The Docker image was built and pushed to ECR with the tag:

```text
myweb:latest
```

ECS uses this ECR image when launching the Fargate tasks.

---

## 6. Amazon ECS and Fargate

An ECS cluster named:

```text
myweb-cluster
```

was created.

The application was deployed using AWS Fargate, so no EC2 instances were required.

The ECS service was configured with:

* Service name: `myweb-service`
* Desired tasks: 2
* Launch type: Fargate
* Platform: Linux/X86_64
* CPU: 0.5 vCPU
* Memory: 1 GB

Two tasks were successfully running simultaneously during verification.

---

## 7. Task Definition

The ECS task definition family is:

```text
myweb-task
```

The verified deployment used revision 6.

The container:

* Exposes port 3000
* Uses CloudWatch Logs
* Uses an ECS task execution role
* Retrieves the application secret from AWS Secrets Manager
* Uses `HOST=0.0.0.0`
* Uses `PORT=3000`

A container health check was configured to verify that the application was responding successfully on port 3000.

---

## 8. Networking

A custom VPC networking configuration was used for the ECS deployment.

The ECS tasks were placed in private subnets in two Availability Zones:

* `eu-north-1a`
* `eu-north-1b`

The ECS tasks did not receive public IP addresses.

A NAT Gateway was used to provide required outbound connectivity from the private task subnets.

The Application Load Balancer was placed in public subnets and provided the public entry point to the application.

This design prevents direct Internet access to the ECS tasks.

---

## 9. Security Groups

Two security groups were used.

### ALB Security Group

The Application Load Balancer security group allows HTTP traffic on port 80 from the Internet.

```text
Inbound:
TCP 80 → 0.0.0.0/0
```

### ECS Task Security Group

The ECS task security group allows application traffic on port 3000 only from the ALB security group.

```text
Inbound:
TCP 3000 → ALB Security Group
```

Therefore, ECS tasks are not directly exposed to the Internet.

---

## 10. Application Load Balancer

An internet-facing Application Load Balancer named:

```text
myweb-ALB
```

was configured.

Configuration:

* Listener: HTTP
* Port: 80
* Target Group: `myweb-tg`
* Target protocol: HTTP
* Target port: 3000
* Health check path: `/`

The ALB distributes incoming traffic to the healthy ECS tasks.

The application was successfully accessed through the ALB DNS endpoint during deployment verification.

---

## 11. Health Checks and High Availability

The ECS service was configured with two desired tasks.

The Application Load Balancer target group performed health checks against the application.

Both ECS task targets were verified as healthy during the successful deployment.

If a task becomes unhealthy or stops, ECS can replace it according to the service configuration.

Running tasks across multiple Availability Zones improves availability.

---

## 12. Secrets Management

Application secrets were not stored in the source code or Dockerfile.

AWS Secrets Manager was used to securely store the application secret.

The secret was injected into the ECS task through the task definition.

The secret value itself was never committed to GitHub.

The ECS task execution role was granted permission to retrieve only the required secret.

---

## 13. IAM

The ECS task execution role used for the deployment was:

```text
ecsTaskExecutionRole
```

The role included the standard ECS task execution permissions and a least-privilege permission allowing:

```text
secretsmanager:GetSecretValue
```

for the required application secret.

No AWS credentials or secret values were committed to the GitHub repository.

---

## 14. Logging and Monitoring

Amazon CloudWatch Logs was configured for the ECS container.

Application startup and runtime logs were verified in CloudWatch.

The logs confirmed successful React application startup and compilation.

CloudWatch Logs provide visibility into application behavior and troubleshooting.

---

## 15. Deployment Process

The deployment process was:

1. Build the React application.
2. Create the Docker image.
3. Test the container locally.
4. Create the Amazon ECR repository.
5. Push the Docker image to ECR.
6. Create the ECS cluster.
7. Create the ECS task definition.
8. Configure Secrets Manager integration.
9. Configure VPC networking and private subnets.
10. Configure security groups.
11. Create the Application Load Balancer and target group.
12. Create the ECS Fargate service.
13. Run two ECS tasks.
14. Verify ALB target health.
15. Access the application through the ALB endpoint.
16. Verify CloudWatch Logs.
17. Verify IAM and secret configuration.

---

## 16. Troubleshooting

Detailed troubleshooting issues, causes, and resolutions are documented separately in TROUBLESHOOTING.md.


## 17. Security Considerations

The deployment follows the main security requirements of the assessment:

* ECS tasks do not have public IP addresses.
* ECS tasks are placed in private subnets.
* Application traffic reaches tasks through the ALB.
* The task security group accepts port 3000 traffic only from the ALB security group.
* Secrets are stored in AWS Secrets Manager.
* AWS credentials and secrets are not stored in GitHub.
* IAM permissions are restricted to required functionality.
* The public entry point is the Application Load Balancer.

---

## 18. Public Application Endpoint

The application was successfully verified through the Application Load Balancer during deployment.

ALB endpoint:

```text
http://myweb-ALB-1306108787.eu-north-1.elb.amazonaws.com
```

The endpoint was verified before the billable runtime infrastructure was cleaned up.

---

## 19. Assumptions

* AWS Region used: `eu-north-1` (Stockholm).
* The application is a simple React web application.
* Infrastructure was configured manually through the AWS Management Console.
* Infrastructure as Code was not used.
* CI/CD was not implemented because it was optional for the assessment.
* No production or sensitive application data was used.

---

## 20. Cleanup

After successful deployment verification, billable runtime resources were cleaned up to avoid unnecessary AWS charges.

The following resources were removed:

* ECS service
* ECS tasks
* Application Load Balancer
* Target group
* NAT Gateway
* Associated Elastic IP
* ECS cluster

The ECR repository, Secrets Manager secret, IAM role, and supporting networking/security resources were retained for assessment evidence where appropriate.

---

## 21. Assessment Evidence

Screenshots demonstrating the deployment and configuration are available in the `screenshots` directory.

The evidence includes:

1. Working application through the ALB
2. Two ECS tasks running
3. Healthy ALB target group targets
4. CloudWatch application logs
5. IAM permissions for secret access
6. AWS Secrets Manager configuration
7. Additional deployment/configuration evidence

---

## 22. Conclusion

This project demonstrates the deployment of a Dockerized web application using Amazon ECS Fargate with Amazon ECR, Application Load Balancer, private networking, security groups, IAM, Secrets Manager, CloudWatch Logs, health checks, and multiple ECS tasks.

The application was successfully deployed and externally accessed through the Application Load Balancer, and the deployment was verified before cleanup of billable runtime resources.

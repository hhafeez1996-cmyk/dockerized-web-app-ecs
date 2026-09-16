# Troubleshooting Report

## 1. ECS Tasks Could Not Retrieve the Secret

### Problem

During the initial ECS service deployment, the ECS Fargate tasks failed to start because the task could not retrieve the application secret from AWS Secrets Manager.

The error indicated a connection timeout while attempting to access Secrets Manager.

### Cause

The ECS tasks were running without public IP addresses in private subnets, but the private subnet did not initially have the required outbound connectivity to AWS Secrets Manager.

### Resolution

A NAT Gateway was configured to provide outbound Internet connectivity from the private ECS task subnets.

The private subnets were associated with a route table containing:

```text
0.0.0.0/0 → NAT Gateway
```

After the networking configuration was corrected, the ECS tasks were able to reach the required AWS service.

---

## 2. Secrets Manager Access Denied

### Problem

After fixing the network connectivity, the ECS task still could not retrieve the secret because the ECS task execution role did not have permission to access the required secret.

### Cause

The ECS task execution role was missing the required `secretsmanager:GetSecretValue` permission.

### Resolution

A least-privilege IAM policy was added to the ECS task execution role.

The permission was restricted to the required secret:

```text
secretsmanager:GetSecretValue
```

The policy did not grant unrestricted access to all Secrets Manager secrets.

After the permission was added, ECS was able to retrieve the secret successfully.

---

## 3. Application Load Balancer Health Checks Failed

### Problem

The ECS tasks were running, but the Application Load Balancer target group initially reported the targets as unhealthy.

### Cause

The Application Load Balancer was initially associated with the incorrect security group.

The ECS task security group allowed traffic on port 3000 only from the ALB security group. Because the ALB was using the wrong security group, the health-check traffic could not reach the ECS containers correctly.

### Resolution

The correct ALB security group was attached to the Application Load Balancer.

The configuration was:

**ALB Security Group**

```text
Inbound TCP 80 → 0.0.0.0/0
```

**ECS Task Security Group**

```text
Inbound TCP 3000 → ALB Security Group
```

The target group health check was configured as:

```text
Protocol: HTTP
Port: 3000
Path: /
```

After correcting the security group association, both ECS task targets became healthy.

The application was then successfully accessible through the Application Load Balancer DNS endpoint.

---

## 4. Verification After Troubleshooting

After resolving the above issues, the deployment was successfully verified:

* Two ECS Fargate tasks were running simultaneously.
* Both Application Load Balancer targets were healthy.
* The React application opened successfully through the ALB DNS endpoint.
* CloudWatch Logs showed successful application startup and compilation.
* The ECS task successfully retrieved the secret from AWS Secrets Manager.
* ECS tasks did not have public IP addresses.
* Application traffic was restricted through the Application Load Balancer and security groups.

---

## 5. Final Outcome

The troubleshooting process resulted in a functional ECS Fargate deployment with:

* Amazon ECR for container image storage
* ECS Fargate for container execution
* Private ECS task subnets
* NAT Gateway for required outbound connectivity
* Application Load Balancer for public access
* Security groups controlling traffic
* AWS Secrets Manager for secret storage
* IAM least-privilege access
* CloudWatch Logs for application logging
* Two ECS tasks for availability

The deployment was successfully tested before the billable runtime resources were cleaned up.

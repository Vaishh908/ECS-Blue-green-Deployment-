# Blue-Green Deployment on AWS ECS

A step-by-step guide to implementing Blue-Green deployment using Amazon ECS (EC2 launch type),
Application Load Balancer, ECR, CodeBuild, and S3 — based on a working production setup.

> **Account:** `100303966926` | **Region:** `us-east-1 (N. Virginia)` | **User:** `vaishnavi%20Jagtap`

---

## Architecture Overview

```
Internet
   │
   ▼
Application Load Balancer (ecs-ALB)
   │  DNS: ecs-ALB-489835085.us-east-1.elb.amazonaws.com
   │  VPC: vpc-0bd984acc6d439752
   │
   └──► Target Group (ecs-tg)  [HTTP:80 | Instance type]
           │  6 Total targets: 4 Healthy | 2 Unhealthy
           │
           ├──► ecs-blue-taskdef-service  (2/2 Tasks Running, EC2, Replica)
           └──► ecs-green-taskdef-service (2/2 Tasks Running, EC2, Replica)

ECS Cluster: ecs-cluster
   ARN: arn:aws:ecs:us-east-1:100303966926:cluster/ecs-cluster
   Status: Active | CloudWatch: Default | Container instances: 1 EC2
   Tasks: 4 Running

Auto Scaling Group: ecs-ASG
   Desired: 1 | Min: 1 | Max: 2 | Status: At desired capacity
   Created: Mon May 18 2026 17:06:59 GMT+0530

   └── Launch Template: ecs-launch-template (lt-0e934afe9f38b681b)
       AMI: ami-0f1fa1255d3d32671 | Type: t3.micro | Key: practical
       AZs: us-east-1a, us-east-1b
       Subnets: subnet-06587e34d9fca778f | subnet-017bbae9a0c9abdff
       Security Group: sg-0f4ec2d29460195d3

ECR Repository: ca-container-registry (Private)
   ├── testblue  — 42.00 MB | Created: May 16 2026 11:13:45 | Last pulled: May 18 17:53:39
   └── testgreen — 42.01 MB | Created: May 16 2026 11:21:32 | Last pulled: May 18 18:40:27

Task Definitions (Active):
   ├── ecs-blue-taskdef
   └── ecs-green-taskdef

CodeBuild Projects:
   ├── ecs-blue-project  (Source: S3 → ecs-blue.zip  | Status: Succeeded | 2 days ago)
   └── ecs-green-project (Source: S3 → ecs-green.zip | Status: Succeeded | 2 days ago)

S3 Bucket: ecs-bluegreen-project-bucket
   ├── ecs-blue.zip  — 1.8 KB | May 16 2026 10:53:31 | Standard
   └── ecs-green.zip — 1.7 KB | May 16 2026 10:53:32 | Standard

IAM Roles (5 project-specific + 7 AWS service-linked):
   ├── AWSServiceRoleForAutoScaling
   ├── AWSServiceRoleForECS
   ├── AWSServiceRoleForElasticLoadBalancing
   ├── AWSServiceRoleForRDS
   ├── AWSServiceRoleForResourceExplorer
   ├── AWSServiceRoleForSupport
   ├── AWSServiceRoleForTrustedAdvisor
   ├── codebuild-ecs-blue-project-service-role  (codebuild)
   ├── ecsExternalInstanceRole                  (ssm)
   ├── ecsInstanceRole                          (ec2)
   ├── ecsTaskExecutionRole                     (ecs-tasks)
   └── lambdaec2fullaccessrole                  (lambda)
```

---

## Prerequisites

- AWS account with appropriate permissions
- AWS CLI installed and configured
- Docker installed locally
- EC2 Key Pair named `practical`

---

## Step 1: Set Up IAM Roles

> **Screenshot 12** — IAM Roles console showing all 12 roles (5/12 project-specific highlighted)

Create the following roles before provisioning any other resources.

### ecsTaskExecutionRole
Allows ECS tasks to pull images from ECR and publish logs to CloudWatch.

**Trusted entity:** `ecs-tasks.amazonaws.com`
**Last activity:** 15 hours ago

Policies to attach:
- `AmazonECSTaskExecutionRolePolicy`

### ecsInstanceRole
Allows EC2 instances to register themselves with the ECS cluster.

**Trusted entity:** `ec2.amazonaws.com`
**Last activity:** 8 minutes ago

Policies to attach:
- `AmazonEC2ContainerServiceforEC2Role`

### ecsExternalInstanceRole
**Trusted entity:** `ssm.amazonaws.com`

Policies to attach:
- `AmazonSSMManagedInstanceCore`

### codebuild-ecs-blue-project-service-role
Allows CodeBuild to access ECR, ECS, and S3.

**Trusted entity:** `codebuild.amazonaws.com`
**Last activity:** 2 days ago

Policies to attach:
- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonECS_FullAccess`
- `AmazonS3ReadOnlyAccess`
- CloudWatch Logs write access

---

## Step 2: Create VPC and Networking

Set up a VPC with two public subnets across different Availability Zones (required for the ALB).

> **Screenshot 10** — ASG Network section confirming both subnets and AZ distribution "Balanced best effort"

```
VPC ID: vpc-0bd984acc6d439752
  ├── Subnet: subnet-06587e34d9fca778f  (us-east-1a)
  └── Subnet: subnet-017bbae9a0c9abdff  (us-east-1b)
```

Create a **Security Group** `sg-0f4ec2d29460195d3` with:
- Inbound: HTTP (80) from `0.0.0.0/0`
- Inbound: SSH (22) from your IP
- Outbound: All traffic

---

## Step 3: Create EC2 Launch Template

> **Screenshot 11** — `ecs-launch-template` (lt-0e934afe9f38b681b) details page in EC2 console

| Setting | Value |
|---|---|
| Launch Template ID | `lt-0e934afe9f38b681b` |
| Launch Template Name | `ecs-launch-template` |
| Default Version | `1` |
| AMI ID | `ami-0f1fa1255d3d32671` (Amazon Linux 2 ECS-Optimized) |
| Instance type | `t3.micro` |
| Key pair name | `practical` |
| Security group ID | `sg-0f4ec2d29460195d3` |
| Availability Zone | `us-east-1a` |
| Owner | `arn:aws:iam::100303966926:root` |
| Created | `2026-05-18T11:32:00.000Z` |

Add the following **User Data** so instances auto-register with the ECS cluster on boot:

```bash
#!/bin/bash
echo ECS_CLUSTER=ecs-cluster >> /etc/ecs/ecs.config
```

---

## Step 4: Create Auto Scaling Group

> **Screenshot 10** — `ecs-ASG` capacity overview showing Desired:1, Min:1, Max:2, Status: At desired capacity

| Setting | Value |
|---|---|
| ASG Name | `ecs-ASG` |
| ARN | `arn:aws:autoscaling:us-east-1:100303966926:autoScalingGroup:2e96372f-eb5d-4e84-ba26-f9a55a1fd50c:autoScalingGroupName/ecs-ASG` |
| Launch template | `ecs-launch-template` (Latest version) |
| Desired capacity | `1` |
| Min | `1` |
| Max | `2` |
| Capacity type | Units (number of instances) |
| Status | At desired capacity |
| Availability Zones | `us-east-1a (use1-az1)`, `us-east-1b (use1-az2)` |
| Subnets | `subnet-06587e34d9fca778f`, `subnet-017bbae9a0c9abdff` |
| AZ distribution | Balanced best effort |
| Request Spot Instances | No |
| Created | Mon May 18 2026 17:06:59 GMT+0530 (India Standard Time) |

---

## Step 5: Create ECS Cluster

> **Screenshot 3** — Clusters list: `ecs-cluster` | 2 Services | 0 Pending | 4 Running | 1 EC2 | CloudWatch: Default

```bash
aws ecs create-cluster \
  --cluster-name ecs-cluster \
  --region us-east-1
```

Cluster details after creation:

| Setting | Value |
|---|---|
| Cluster name | `ecs-cluster` |
| ARN | `arn:aws:ecs:us-east-1:100303966926:cluster/ecs-cluster` |
| Status | Active |
| CloudWatch monitoring | Default |
| Registered container instances | 1 EC2 |
| Active services | 2 |
| Running tasks | 4 |
| Pending tasks | - |
| Draining | - |
| Capacity provider strategy | No default found |

Once the ASG launches an EC2, it auto-registers via the User Data script. Confirm **1 EC2** appears under Infrastructure in the cluster console.

---

## Step 6: Configure Nginx on EC2 Instance

> **Screenshot 1** — SSH session to EC2 at `54.84.88.65` from `C:\Users\vj251` using `practical.pem`
> Last login: Sun May 17 12:20:26 2026 from 157.32.112.208

SSH into the EC2 instance:

```bash
ssh -i ./Downloads/practical.pem ec2-user@54.84.88.65
```

The instance runs **Amazon Linux 2 (ECS Optimized)**.

> **Note:** AL2 End of Life is 2026-06-30. Consider migrating to Amazon Linux 2023 (supported until 2028-03-15).
> Docs: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI

Configure Nginx:

```bash
sudo vi /etc/nginx/conf.d/mysite.conf
```

Validate and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Expected output (confirmed in screenshot):
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Verify the health endpoint:

```bash
curl http://localhost/health
```

---

## Step 7: Create ECR Repository and Push Images

> **Screenshot 7** — ECR `ca-container-registry` (Private) showing both images with digest and pull timestamps

```bash
aws ecr create-repository \
  --repository-name ca-container-registry \
  --region us-east-1
```

Authenticate Docker to ECR and push both images:

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  100303966926.dkr.ecr.us-east-1.amazonaws.com
```

**Blue image:**
```bash
docker build -t testblue ./blue-app
docker tag testblue:latest \
  100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testblue
docker push \
  100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testblue
```

**Green image:**
```bash
docker build -t testgreen ./green-app
docker tag testgreen:latest \
  100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testgreen
docker push \
  100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testgreen
```

**Resulting images in `ca-container-registry`:**

| Image tag | Type | Size | Created | Last pulled | Digest |
|---|---|---|---|---|---|
| `testgreen` | Image | 42.01 MB | May 16 2026, 11:21:32 (UTC+05:5) | May 18 2026, 18:40:27 | `sha256:9a65d8256e802e...` |
| `testblue`  | Image | 42.00 MB | May 16 2026, 11:13:45 (UTC+05:5) | May 18 2026, 17:53:39 | `sha256:a2bf1a7d3e6f965...` |

---

## Step 8: Set Up S3 Bucket for CodeBuild Source

> **Screenshot 6** — S3 bucket `ecs-bluegreen-project-bucket` showing 2 objects

```bash
aws s3 mb s3://ecs-bluegreen-project-bucket --region us-east-1
```

Upload application source archives:

```bash
aws s3 cp ecs-blue.zip  s3://ecs-bluegreen-project-bucket/ecs-blue.zip
aws s3 cp ecs-green.zip s3://ecs-bluegreen-project-bucket/ecs-green.zip
```

**Resulting bucket objects:**

| Object | Type | Size | Last Modified | Storage class |
|---|---|---|---|---|
| `ecs-blue.zip`  | zip | 1.8 KB | May 16, 2026 10:53:31 UTC+05:30 | Standard |
| `ecs-green.zip` | zip | 1.7 KB | May 16, 2026 10:53:32 UTC+05:30 | Standard |

Each zip must contain:
- `Dockerfile`
- Application source code
- `buildspec.yml`

### Sample `buildspec.yml` (for blue)

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging into ECR...
      - aws ecr get-login-password --region us-east-1 | docker login
          --username AWS --password-stdin
          100303966926.dkr.ecr.us-east-1.amazonaws.com
  build:
    commands:
      - echo Building Docker image...
      - docker build -t ca-container-registry:testblue .
      - docker tag ca-container-registry:testblue
          100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testblue
  post_build:
    commands:
      - echo Pushing image to ECR...
      - docker push
          100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testblue
      - echo Updating ECS service...
      - aws ecs update-service
          --cluster ecs-cluster
          --service ecs-blue-taskdef-service
          --force-new-deployment
          --region us-east-1
```

---

## Step 9: Create CodeBuild Projects

> **Screenshot 5** — CodeBuild Build projects list: both `ecs-blue-project` and `ecs-green-project` show Succeeded, modified 2 days ago

### ecs-blue-project

| Setting | Value |
|---|---|
| Name | `ecs-blue-project` |
| Source provider | Amazon S3 |
| Repository (bucket/key) | `ecs-bluegreen-project-bucket/ecs-blue.zip` |
| Latest build status | Succeeded |
| Last modified | 2 days ago |
| Service role | `codebuild-ecs-blue-project-service-role` |
| Privileged mode | Enabled (required for Docker builds) |

### ecs-green-project

| Setting | Value |
|---|---|
| Name | `ecs-green-project` |
| Source provider | Amazon S3 |
| Repository (bucket/key) | `ecs-bluegreen-project-bucket/ecs-green.zip` |
| Latest build status | Succeeded |
| Last modified | 2 days ago |
| Service role | `codebuild-ecs-blue-project-service-role` |

Trigger builds:

```bash
aws codebuild start-build --project-name ecs-blue-project  --region us-east-1
aws codebuild start-build --project-name ecs-green-project --region us-east-1
```

---

## Step 10: Create ECS Task Definitions

> **Screenshot 4** — Task Definitions showing `ecs-blue-taskdef` and `ecs-green-taskdef`, both Active

### ecs-blue-taskdef

```json
{
  "family": "ecs-blue-taskdef",
  "executionRoleArn": "arn:aws:iam::100303966926:role/ecsTaskExecutionRole",
  "networkMode": "bridge",
  "containerDefinitions": [
    {
      "name": "blue-container",
      "image": "100303966926.dkr.ecr.us-east-1.amazonaws.com/ca-container-registry:testblue",
      "memory": 256,
      "cpu": 128,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 0,
          "protocol": "tcp"
        }
      ]
    }
  ],
  "requiresCompatibilities": ["EC2"]
}
```

### ecs-green-taskdef

Same structure — replace image tag with `testgreen` and family with `ecs-green-taskdef`.

Register both:

```bash
aws ecs register-task-definition \
  --cli-input-json file://ecs-blue-taskdef.json \
  --region us-east-1

aws ecs register-task-definition \
  --cli-input-json file://ecs-green-taskdef.json \
  --region us-east-1
```

**Result:** Both task definitions appear in the console with status **Active**.

---

## Step 11: Create Target Group

> **Screenshot 8** — Target group `ecs-tg` — 6 Total targets: 4 Healthy, 2 Unhealthy | Attached to `ecs-ALB`

| Setting | Value |
|---|---|
| Name | `ecs-tg` |
| ARN | `arn:aws:elasticloadbalancing:us-east-1:100303966926:targetgroup/ecs-tg/94599b0cd1a6f0bd` |
| Target type | Instance |
| Protocol : Port | HTTP : 80 |
| Protocol version | HTTP1 |
| VPC | `vpc-0bd984acc6d439752` |
| IP address type | IPv4 |
| Load balancer | `ecs-ALB` |
| Total targets | 6 (4 Healthy, 2 Unhealthy, 0 Unused, 0 Initial, 0 Draining) |

Set the health check path to `/health` (served by Nginx on port 80 of each EC2 instance).

---

## Step 12: Create Application Load Balancer

> **Screenshot 9** — Load Balancer `ecs-ALB`: Active, Internet-facing, Application, IPv4 | Created May 18 2026 17:16

| Setting | Value |
|---|---|
| Name | `ecs-ALB` |
| ARN | `arn:aws:elasticloadbalancing:us-east-1:100303966926:loadbalancer/app/ecs-ALB/8d62bc23f0f73955` |
| Load balancer type | Application |
| Status | Active |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `vpc-0bd984acc6d439752` |
| Availability Zones | 2 Availability Zones |
| Security group | `sg-0f4ec2d29460195d3` |
| DNS name | `ecs-ALB-489835085.us-east-1.elb.amazonaws.com` (A Record) |
| Hosted zone | `Z35SXDOTRQ7X7K` |
| Date created | May 18, 2026 17:16 UTC+05:30 |

Add an **HTTP:80** listener forwarding to the `ecs-tg` target group.

---

## Step 13: Create ECS Services

> **Screenshot 2** — Cluster services tab: both services Active with 2/2 tasks running, Deployment Completed
> **Screenshot 3** — Cluster overview: 4 Running tasks, 2 Active services, 1 EC2

### ecs-blue-taskdef-service

```bash
aws ecs create-service \
  --cluster ecs-cluster \
  --service-name ecs-blue-taskdef-service \
  --task-definition ecs-blue-taskdef \
  --desired-count 2 \
  --launch-type EC2 \
  --scheduling-strategy REPLICA \
  --load-balancers \
    "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:100303966926:targetgroup/ecs-tg/94599b0cd1a6f0bd,containerName=blue-container,containerPort=80" \
  --region us-east-1
```

### ecs-green-taskdef-service

```bash
aws ecs create-service \
  --cluster ecs-cluster \
  --service-name ecs-green-taskdef-service \
  --task-definition ecs-green-taskdef \
  --desired-count 2 \
  --launch-type EC2 \
  --scheduling-strategy REPLICA \
  --load-balancers \
    "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:100303966926:targetgroup/ecs-tg/94599b0cd1a6f0bd,containerName=green-container,containerPort=80" \
  --region us-east-1
```

**Expected result in console:**

| Service name | Status | Scheduling | Launch | Task definition | Tasks | Last deployment |
|---|---|---|---|---|---|---|
| `ecs-blue-taskdef-service`  | Active | Replica | EC2 | `ecs-blue-taskd...`  | 2/2 Running | Completed |
| `ecs-green-taskdef-service` | Active | Replica | EC2 | `ecs-green-task...` | 2/2 Running | Completed |

---

## Step 14: Perform the Blue-Green Switch

To shift all traffic, update the ALB listener to forward to the desired environment.

### Switch to Green (Console)

1. Go to **EC2 → Load Balancers → ecs-ALB → Listeners and rules**
2. Edit the **HTTP:80** listener
3. Change **Forward to** target group from blue to green
4. Save changes

### Switch to Green (CLI)

```bash
# Get listener ARN
aws elbv2 describe-listeners \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:100303966926:loadbalancer/app/ecs-ALB/8d62bc23f0f73955 \
  --region us-east-1

# Forward traffic to green
aws elbv2 modify-listener \
  --listener-arn <listener-arn> \
  --default-actions Type=forward,TargetGroupArn=<green-tg-arn> \
  --region us-east-1
```

### Rollback to Blue (instant, no redeployment needed)

```bash
aws elbv2 modify-listener \
  --listener-arn <listener-arn> \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:100303966926:targetgroup/ecs-tg/94599b0cd1a6f0bd \
  --region us-east-1
```

---

## Final State Verification

| Screenshot | Resource | Confirmed State |
|---|---|---|
| Screenshot 1 | EC2 SSH + Nginx | Config OK, `/health` responding |
| Screenshot 2 | ECS Services | Both Active, 2/2 tasks, Deployment Completed |
| Screenshot 3 | ECS Cluster `ecs-cluster` | Active, 1 EC2, 4 Running tasks |
| Screenshot 4 | Task Definitions | `ecs-blue-taskdef` + `ecs-green-taskdef` Active |
| Screenshot 5 | CodeBuild | Both projects Succeeded |
| Screenshot 6 | S3 Bucket | `ecs-blue.zip` (1.8KB) + `ecs-green.zip` (1.7KB) |
| Screenshot 7 | ECR | `testblue` (42MB) + `testgreen` (42MB) |
| Screenshot 8 | Target Group `ecs-tg` | 6 targets: 4 Healthy, 2 Unhealthy |
| Screenshot 9 | ALB `ecs-ALB` | Active, Internet-facing, DNS assigned |
| Screenshot 10 | ASG `ecs-ASG` | At desired capacity (1), Min 1 Max 2 |
| Screenshot 11 | Launch Template | `ecs-launch-template` v1, t3.micro, AL2 ECS AMI |
| Screenshot 12 | IAM Roles | All 5 project roles present and trusted |

---

## Key Concepts

**Blue environment** — The currently live production version serving all user traffic.

**Green environment** — The new version deployed and tested in parallel before any traffic shift.

**Zero-downtime switching** — Traffic is cut over instantly at the ALB listener level. The old environment keeps running alongside for immediate rollback with no redeployment.

**When to use this pattern** — New feature releases, dependency upgrades, or any change where instant rollback is business-critical.

---

## Troubleshooting

| Issue | What to Check |
|---|---|
| Tasks not starting | EC2 registered in cluster? Check cluster → Infrastructure tab |
| Unhealthy targets in `ecs-tg` | Nginx `/health` returning 200? Port 80 open in `sg-0f4ec2d29460195d3`? |
| CodeBuild failing | Privileged mode enabled? `codebuild-ecs-blue-project-service-role` has ECR + ECS permissions? |
| Images not pulling | `ecsTaskExecutionRole` has `AmazonECSTaskExecutionRolePolicy` attached? |
| ALB returning 502 | Container port in task definition matches the port the app listens on? |
| EC2 not joining cluster | `ECS_CLUSTER=ecs-cluster` in `/etc/ecs/ecs.config`? SSH with `ssh -i ./Downloads/practical.pem ec2-user@<ip>` |
| SSH connection refused | Security group allows port 22? Using the correct `practical` key pair? |

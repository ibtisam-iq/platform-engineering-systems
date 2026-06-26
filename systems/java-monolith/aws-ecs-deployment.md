# AWS ECS Fargate Deployment — Java Monolith Application

A production-grade, containerized deployment of the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) on AWS ECS Fargate.

The application container runs on ECS Fargate managed by an ECS Service, backed by Amazon RDS (MySQL 8.4), fronted by an Application Load Balancer, and routed through Cloudflare DNS with a verified TLS certificate from ACM.

> This is **Step 5B** of the overall DevOps implementation journey documented in the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) repository.  
> Step 5A (bare-metal, containerless) is documented in [`aws-bare-metal-deployment.md`](./aws-bare-metal-deployment.md).

---

## Architecture Overview

```
Internet
   │
   ▼
Cloudflare DNS  →  bankapp.ibtisam-iq.com
   │
   ▼
Application Load Balancer          ← sg-alb  (port 80 → redirect 443 | port 443 → forward)
   │
   ├── us-east-1a  (subnet-public-1a   10.0.0.0/26)    ── NAT Gateway
   └── us-east-1b  (subnet-public-1b   10.0.0.64/26)
   │
   ▼
ECS Service (desired 2 | Fargate)
   ├── Task — us-east-1a  (subnet-private-1a  10.0.0.128/26)   ← sg-ecs  (port 8000 from ALB)
   └── Task — us-east-1b  (subnet-private-1b  10.0.0.192/26)
   │
   ▼
Amazon RDS — MySQL 8.4                                          ← sg-rds  (port 3306 from ECS)
(subnet-private-1a | Single-AZ | Multi-AZ ready by design)
```

### AWS Services

| Service | Role |
|---|---|
| VPC + Subnets | Isolated network — public/private separation across 2 AZs |
| Internet Gateway | Inbound internet access for public subnets |
| NAT Gateway | Outbound internet for ECS Tasks in private subnets |
| Bastion Host | Secure SSH jump server for RDS initialization |
| Security Groups | Least-privilege layer-4 traffic control between all tiers |
| Amazon ECR | Private container registry for the application image |
| Amazon RDS (MySQL 8.4) | Managed relational database in private subnet |
| IAM Role (ecsTaskExecutionRole) | Grants ECS permission to pull images and write logs |
| CloudWatch Logs | Container stdout/stderr log storage via awslogs driver |
| ECS Cluster | Logical namespace for the ECS Service and Tasks |
| Task Definition | Container manifest — image, port, env vars, logging |
| ECS Service | Maintains desired Task count, integrates with ALB |
| Application Load Balancer | HTTP/HTTPS traffic distribution across ECS Tasks |
| ACM | TLS certificate for `bankapp.ibtisam-iq.com` |
| Cloudflare DNS | CNAME record pointing to the ALB |

### Bare-Metal vs ECS — Key Differences

| Concern | Bare-Metal (Step 5A) | ECS Fargate (Step 5B) |
|---|---|---|
| Compute unit | EC2 instance | Fargate Task (container) |
| Scaling | Auto Scaling Group | ECS Service desired count |
| Bootstrap | User data + systemd | Task Definition + ECR image |
| Artifact storage | S3 (JAR) | ECR (Docker image) |
| IAM role | EC2 instance profile | ECS Task Execution Role |
| Target type | `instance` | `ip` |
| No. of private IPs | One per EC2 instance | One per Task (awsvpc ENI) |
| SSH access | Via Bastion to EC2 | Not available (serverless) |

---

## Stage 1 — Network & Security Infrastructure

### Phase 1 — VPC, Subnets, and Routing

#### Environment Variables

Capture all resource IDs as shell variables to keep commands clean and reproducible across phases.

```bash
export REGION="us-east-1"
export AZ_A="us-east-1a"
export AZ_B="us-east-1b"
export PROJECT="java-monolith"
export ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
```

#### VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/24 \
  --region $REGION \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$PROJECT-vpc}]" \
  --query 'Vpc.VpcId' --output text)
```

#### Subnets

Divide the `/24` block (`10.0.0.0/24`) into four `/26` ranges — two public and two private, one per AZ.

| Subnet Name | CIDR | AZ | Type |
|---|---|---|---|
| `subnet-public-1a` | `10.0.0.0/26` | us-east-1a | Public |
| `subnet-public-1b` | `10.0.0.64/26` | us-east-1b | Public |
| `subnet-private-1a` | `10.0.0.128/26` | us-east-1a | Private |
| `subnet-private-1b` | `10.0.0.192/26` | us-east-1b | Private |

```bash
PUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.0.0/26 --availability-zone $AZ_A \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PROJECT-public-1a}]" \
  --query 'Subnet.SubnetId' --output text)

PUB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.0.64/26 --availability-zone $AZ_B \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PROJECT-public-1b}]" \
  --query 'Subnet.SubnetId' --output text)

PRIV_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.0.128/26 --availability-zone $AZ_A \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PROJECT-private-1a}]" \
  --query 'Subnet.SubnetId' --output text)

PRIV_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.0.192/26 --availability-zone $AZ_B \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PROJECT-private-1b}]" \
  --query 'Subnet.SubnetId' --output text)
```

#### Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=$PROJECT-igw}]" \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

#### NAT Gateway

Place the NAT Gateway in the public subnet (`us-east-1a`) so that ECS Tasks in private subnets can reach the internet for ECR image pulls — without being directly reachable from outside.

```bash
EIP=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --region $REGION --output text)

NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_1A \
  --allocation-id $EIP \
  --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=$PROJECT-nat}]" \
  --query 'NatGateway.NatGatewayId' --region $REGION --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID --region $REGION
```

#### Route Tables

```bash
# Public Route Table → IGW
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$PROJECT-public-rt}]" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_1A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_1B

# Private Route Table → NAT
PRIV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$PROJECT-private-rt}]" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIV_RT \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID

aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_1A
aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_1B
```

#### Bastion Host

Launch a small EC2 instance in the public subnet as the sole SSH entry point for RDS initialization. ECS Tasks are serverless — SSH is not available to them.

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text)

BASTION_SG=$(aws ec2 create-security-group \
  --group-name "$PROJECT-bastion-sg" \
  --description "Bastion Host SG" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com | tr -d '\n')

aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 \
  --cidr $MY_IP/32 --region $REGION

aws ec2 create-key-pair \
  --key-name "$PROJECT" \
  --region "$REGION" \
  --query 'KeyMaterial' \
  --output text > "${PROJECT}.pem"

chmod 400 "${PROJECT}.pem"

BASTION_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --subnet-id $PUB_1A \
  --security-group-ids $BASTION_SG \
  --associate-public-ip-address \
  --key-name $PROJECT \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PROJECT-bastion}]" \
  --query 'Instances[0].InstanceId' --region $REGION --output text)
```

---

### Phase 2 — Security Groups

Create three security groups with strict least-privilege rules. Each tier only accepts traffic from the tier directly in front of it.

```bash
# SG 1 — Application Load Balancer (open to internet on 80 and 443)
SG_ALB=$(aws ec2 create-security-group \
  --group-name "$PROJECT-alb-sg" \
  --description "ALB Security Group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ALB \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ALB \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# SG 2 — ECS Tasks (port 8000 from ALB only)
SG_ECS=$(aws ec2 create-security-group \
  --group-name "$PROJECT-ecs-sg" \
  --description "ECS Tasks Security Group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ECS \
  --protocol tcp --port 8000 --source-group $SG_ALB

# SG 3 — RDS MySQL (port 3306 from ECS Tasks only)
SG_RDS=$(aws ec2 create-security-group \
  --group-name "$PROJECT-rds-sg" \
  --description "RDS Security Group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_RDS \
  --protocol tcp --port 3306 --source-group $SG_ECS
```

**Traffic flow enforced by security group chaining:**

```
Internet → ALB (80/443) → ECS Task (8000) → RDS (3306)
              ↑                  ↑                ↑
           sg-alb             sg-ecs           sg-rds
```

!!! note "ECS vs Bare-Metal security group difference"
    In the bare-metal deployment, `sg-app` controlled EC2 instances.  
    Here `sg-ecs` controls ECS Tasks.  
    The rule logic is identical — only the name and the attached resource type differ.

---

## Stage 2 — Data & Container Layer

### Phase 3 — Database (Amazon RDS — MySQL 8.4)

#### RDS Subnet Group

Build the subnet group using both private subnets, following Multi-AZ best practice even though Single-AZ is selected for cost control.

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name "$PROJECT-db-subnet-group" \
  --db-subnet-group-description "Private subnets for RDS" \
  --subnet-ids $PRIV_1A $PRIV_1B
```

#### RDS Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier "$PROJECT-db" \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.4.3 \
  --master-username admin \
  --master-user-password your_root_password_here \
  --db-name IbtisamIQbankappdb \
  --db-subnet-group-name "$PROJECT-db-subnet-group" \
  --vpc-security-group-ids $SG_RDS \
  --no-multi-az \
  --allocated-storage 20 \
  --no-publicly-accessible
```

Press `q` to exit the pager. The RDS instance continues creating in the background.

#### Database Initialization via Bastion

The RDS endpoint is not publicly accessible. Use the Bastion host to reach it. ECS Tasks cannot be SSH'd into, so the Bastion is the only initialization path.

Add a temporary inbound rule on `sg-rds` to allow port 3306 from the Bastion SG, then remove it immediately after initialization.

```bash
# On local machine — add port 3306 temporarily to RDS SG, sourced from Bastion SG
aws ec2 authorize-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 3306 \
  --source-group $BASTION_SG

# SSH into Bastion
ssh -i $PROJECT.pem ubuntu@<BASTION_PUBLIC_IP>

# On the Bastion host
sudo apt update -y && sudo apt install -y mysql-client

# Connect to RDS
mysql -h <RDS_ENDPOINT> -u admin -p

# Inside MySQL shell
CREATE DATABASE IbtisamIQbankappdb;
CREATE USER 'your_db_user'@'%' IDENTIFIED BY 'your_db_password';
GRANT ALL PRIVILEGES ON IbtisamIQbankappdb.* TO 'your_db_user'@'%';
FLUSH PRIVILEGES;
EXIT;

# On local machine — remove the temporary rule immediately after
aws ec2 revoke-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 3306 \
  --source-group $BASTION_SG
```

!!! warning "Remove the temporary rule without fail"
    After removal, `sg-rds` returns to its permanent state: port 3306 open from `sg-ecs` only.  
    In the bare-metal deployment, the same pattern applied with `sg-app` as the permanent source.  
    Here `sg-ecs` replaces `sg-app` — the logic is identical.

---

### Phase 4 — Container Image (Amazon ECR)

In the bare-metal deployment, the application JAR was stored in S3 and pulled at EC2 boot time. In the ECS deployment, the application runs as a Docker container. Store the image in ECR instead of S3.

!!! note "S3 is replaced by ECR"
    Bare-metal: EC2 pulls JAR from S3 at boot via IAM instance profile.  
    ECS: Fargate Task pulls container image from ECR at launch via ecsTaskExecutionRole.

#### Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name $PROJECT-app \
  --region $REGION

ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$PROJECT-app"
```

#### Authenticate Docker to ECR

```bash
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin \
    $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

#### Build, Tag, and Push

```bash
# Build from the application repository root
docker build -t $PROJECT-app .

# Tag for ECR
docker tag $PROJECT-app:latest $ECR_URI:latest

# Push
docker push $ECR_URI:latest
```

#### Verify the image

```bash
aws ecr describe-images \
  --repository-name $PROJECT-app \
  --region $REGION \
  --query 'imageDetails[*].[imageTags,imagePushedAt]' \
  --output table
```

---

### Phase 5 — IAM Execution Role, CloudWatch Log Group, and ECS Cluster

These three prerequisites must exist before registering the Task Definition.

#### ecsTaskExecutionRole

ECS uses this role to pull the image from ECR and send container logs to CloudWatch. Without it, the Task goes `STOPPED` immediately.

!!! note "ecsTaskExecutionRole vs EC2 instance profile"
    In bare-metal, an EC2 instance profile with `AmazonS3ReadOnlyAccess` gave EC2 permission to pull the JAR from S3.  
    In ECS, `ecsTaskExecutionRole` with `AmazonECSTaskExecutionRolePolicy` gives ECS permission to pull the image from ECR and write logs to CloudWatch.  
    Both roles exist for the same reason: grant the compute layer access to its artifact source.

```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

sleep 10

EXEC_ROLE_ARN=$(aws iam get-role \
  --role-name ecsTaskExecutionRole \
  --query 'Role.Arn' --output text)
```

#### CloudWatch Log Group

ECS does not auto-create log groups. Create the log group manually before registering the Task Definition — otherwise the Task fails at log driver initialization and stops immediately.

```bash
LOG_GROUP="/ecs/$PROJECT"

aws logs create-log-group \
  --log-group-name "$LOG_GROUP" \
  --region $REGION || true

aws logs describe-log-groups \
  --log-group-name-prefix "$LOG_GROUP" \
  --region $REGION
```

#### ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name $PROJECT \
  --region $REGION
```

!!! note "ECS Cluster vs Auto Scaling Group"
    In bare-metal, the ASG was the compute backbone — it launched EC2 instances and replaced failed ones.  
    In ECS Fargate, the Cluster is just a logical namespace. ECS itself manages compute.  
    The ECS Service (not the Cluster) is the equivalent of the ASG — it maintains desired Task count and replaces failed Tasks.

---

### Phase 6 — Task Definition

The Task Definition is the ECS equivalent of the Launch Template + User Data from the bare-metal deployment.

!!! note "Task Definition vs Launch Template"
    Bare-metal Launch Template defined: AMI, instance type, security group, IAM profile, and user data (Java install, S3 pull, systemd service).  
    ECS Task Definition defines: container image, port, environment variables, IAM execution role, and log configuration.  
    Both describe *what to run* — not *where to run it*. Placement decisions (subnets, AZs) remain outside both.

```bash
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "$PROJECT-db" \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

cat > taskdef.json <<EOF
{
  "family": "$PROJECT-task",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "$EXEC_ROLE_ARN",
  "containerDefinitions": [
    {
      "name": "$PROJECT-app",
      "image": "$ECR_URI:latest",
      "portMappings": [
        { "containerPort": 8000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "SPRING_APPLICATION_NAME",   "value": "IbtisamIQBankApp" },
        { "name": "SPRING_DATASOURCE_USERNAME", "value": "your_db_user" },
        { "name": "SPRING_DATASOURCE_PASSWORD", "value": "your_db_password" },
        { "name": "SPRING_DATASOURCE_URL",      "value": "jdbc:mysql://$RDS_ENDPOINT:3306/IbtisamIQbankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true" },
        { "name": "SERVER_PORT",                "value": "8000" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "$LOG_GROUP",
          "awslogs-region": "$REGION",
          "awslogs-stream-prefix": "$PROJECT"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition \
  --cli-input-json file://taskdef.json \
  --region $REGION
```

!!! note "Environment variables — same values, different mechanism"
    Bare-metal: environment variables were written into the systemd unit file inside the user data script.  
    ECS: the same environment variables are defined in the `environment` array of the container definition.  
    The `SPRING_DATASOURCE_URL` value is identical in both — pointing to the same RDS endpoint.

!!! warning "networkMode is not VPC selection"
    Setting `networkMode: awsvpc` enables each Task to receive its own ENI and private IP.  
    It does not choose which VPC, subnet, or security group to use.  
    Those are runtime decisions made when creating the ECS Service.

---

## Stage 3 — Traffic, Scaling & Verification

### Phase 7 — Target Group

Create the Target Group before the ALB because the listener references it by ARN.

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name "$PROJECT-tg" \
  --protocol HTTP \
  --port 8000 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --ip-address-type ipv4 \
  --protocol-version HTTP1 \
  --health-check-protocol HTTP \
  --health-check-path /actuator/health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

!!! note "Target type: ip, not instance"
    Bare-metal Target Group used `--target-type instance` — the ALB forwarded traffic to EC2 instance IDs.  
    ECS Fargate Target Group must use `--target-type ip` — each Task gets its own ENI with a private IP, and the ALB forwards to that IP directly.  
    Using `instance` with Fargate causes targets to never register, and health checks to fail indefinitely.

!!! note "Health check path"
    `/actuator/health` is exposed by Spring Boot Actuator and returns `{"status":"UP"}` when the application is ready.  
    This path is identical to what was used in the bare-metal deployment — no change required.

---

### Phase 8 — Application Load Balancer, ACM & Cloudflare DNS

#### ALB

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name "$PROJECT-alb" \
  --type application \
  --scheme internet-facing \
  --subnets $PUB_1A $PUB_1B \
  --security-groups $SG_ALB \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

aws elbv2 wait load-balancer-available \
  --load-balancer-arns $ALB_ARN
```

#### ACM Certificate

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "bankapp.ibtisam-iq.com" \
  --validation-method DNS \
  --query 'CertificateArn' --output text)
```

Retrieve the validation CNAME:

```bash
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord' \
  --output table
```

Add the `Name` and `Value` fields as a CNAME record in **Cloudflare → ibtisam-iq.com → DNS**:

| Field | Value |
|-------|-------|
| Type | `CNAME` |
| Name | everything before `.ibtisam-iq.com.` in the `Name` field |
| Target | the full `Value` field |
| Proxy status | **DNS only (grey cloud)** — must not be proxied |

Wait for validation to complete:

```bash
aws acm wait certificate-validated --certificate-arn $CERT_ARN
```

!!! warning "Certificate must be ISSUED before creating the HTTPS listener"
    Attaching a `PENDING_VALIDATION` certificate to a listener causes `UnsupportedCertificate` error.

#### Listeners

```bash
# HTTP → HTTPS redirect (301)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]' \
  --no-cli-pager

# HTTPS → Target Group (forward)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates "CertificateArn=$CERT_ARN" \
  --default-actions "[{
    \"Type\": \"forward\",
    \"TargetGroupArn\": \"$TG_ARN\"
  }]" \
  --no-cli-pager
```

#### Cloudflare — DNS CNAME Record

Add a CNAME record in **Cloudflare → ibtisam-iq.com → DNS** pointing `bankapp.ibtisam-iq.com` to the ALB:

| Field | Value |
|-------|-------|
| Type | `CNAME` |
| Name | `bankapp` |
| Target | value of `$ALB_DNS` |
| Proxy status | **DNS only (grey cloud)** |

#### Session Recovery

Restore all variables if the shell session was interrupted between steps:

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names "$PROJECT-alb" \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

CERT_ARN=$(aws acm list-certificates \
  --query "CertificateSummaryList[?DomainName=='bankapp.ibtisam-iq.com'].CertificateArn" \
  --output text)

TG_ARN=$(aws elbv2 describe-target-groups \
  --names "$PROJECT-tg" \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

EXEC_ROLE_ARN=$(aws iam get-role \
  --role-name ecsTaskExecutionRole \
  --query 'Role.Arn' --output text)

ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$PROJECT-app"
LOG_GROUP="/ecs/$PROJECT"
```

---

### Phase 9 — ECS Service

The ECS Service is the functional equivalent of the Auto Scaling Group in the bare-metal deployment. It uses the Task Definition as a template, maintains the desired count of running Tasks, replaces failed Tasks, and integrates with the ALB Target Group.

!!! note "ECS Service vs Auto Scaling Group"
    Bare-metal ASG: launched EC2 instances using a Launch Template, registered them with the Target Group, replaced failed instances automatically.  
    ECS Service: launches Fargate Tasks using a Task Definition, registers Task IPs with the Target Group (type ip), replaces stopped Tasks automatically.  
    Both use `desired-count / desired-capacity = 2`, minimum 2, across both private subnets.

```bash
aws ecs create-service \
  --cluster $PROJECT \
  --service-name $PROJECT-svc \
  --task-definition $PROJECT-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$PRIV_1A,$PRIV_1B],
    securityGroups=[$SG_ECS],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=$PROJECT-app,containerPort=8000" \
  --health-check-grace-period 120 \
  --region $REGION
```

!!! note "assignPublicIp=DISABLED"
    Tasks run in private subnets. All inbound traffic arrives through the ALB.  
    Tasks reach ECR and CloudWatch outbound via the NAT Gateway.  
    This mirrors the bare-metal design where EC2 instances were also in private subnets.

!!! note "health-check-grace-period"
    Spring Boot takes time to start. Set the grace period to 120 seconds so the ALB does not mark Tasks unhealthy before the application finishes booting.  
    In bare-metal, the same issue was addressed with `--health-check-grace-period 120` on the ASG.

#### Optional: run a one-off Task to validate before creating the Service

Use `run-task` to verify the Task Definition works correctly before committing to a Service.

```bash
aws ecs run-task \
  --cluster $PROJECT \
  --launch-type FARGATE \
  --task-definition $PROJECT-task \
  --network-configuration "awsvpcConfiguration={
    subnets=[$PUB_1A],
    securityGroups=[$SG_ECS],
    assignPublicIp=ENABLED
  }" \
  --count 1 \
  --region $REGION
```

!!! note "run-task for smoke testing"
    This is the ECS equivalent of launching a single EC2 instance outside the ASG to test the user data script.  
    Place the Task in a public subnet with `assignPublicIp=ENABLED` to reach it directly.  
    Once verified, tear down the test Task and proceed with the Service in private subnets.

---

### Phase 10 — Verification

#### ECS Service and Task Status

```bash
# Check Service status
aws ecs describe-services \
  --cluster $PROJECT \
  --services $PROJECT-svc \
  --query 'services[0].[status,runningCount,desiredCount,pendingCount]' \
  --output table

# List running Tasks
aws ecs list-tasks \
  --cluster $PROJECT \
  --service-name $PROJECT-svc \
  --query 'taskArns' \
  --output table
```

Expected: `runningCount = 2`, `desiredCount = 2`, `pendingCount = 0`.

#### Target Group Health

```bash
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table
```

Expected output:

```
----------------------------------
|    DescribeTargetHealth        |
+------------------+-------------+
|  10.0.0.134      |  healthy    |
|  10.0.0.196      |  healthy    |
+------------------+-------------+
```

!!! note "IP addresses instead of instance IDs"
    Bare-metal Target Group showed EC2 instance IDs like `i-0abc123def456`.  
    ECS Fargate Target Group shows Task private IP addresses like `10.0.0.134`.  
    Both confirm the same thing: targets are registered and passing health checks.

#### CloudWatch Logs

```bash
# List log streams
aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --query 'logStreams[*].logStreamName' \
  --output table

# Tail a stream
aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "<STREAM_NAME>" \
  --limit 50 \
  --output text
```

#### Application Endpoints

```bash
# HTTP — should return 301 redirect to HTTPS
curl -I http://bankapp.ibtisam-iq.com

# HTTPS — should return HTTP 200
curl -I https://bankapp.ibtisam-iq.com

# Spring Boot Actuator health check
curl https://bankapp.ibtisam-iq.com/actuator/health
# Expected: {"status":"UP"}
```

#### End-to-End Functional Test

Access the application via browser at `https://bankapp.ibtisam-iq.com`. Register a new account, verify login, and confirm transactional data is written to and read from RDS — confirming all three tiers (ALB → ECS Task → RDS) are operating correctly end to end.

---

## Stage 4 — Troubleshooting

The same two issues that surfaced in the bare-metal deployment also apply here. Both root causes are in the application code, not the infrastructure — so the fixes are identical regardless of whether the compute layer is EC2 or ECS.

### Issue 1 — ALB Health Checks Failing (Tasks Marked Unhealthy)

**Symptom:** ECS Service launches Tasks, but the Target Group marks them `unhealthy`. ECS enters a replacement loop.

**Diagnosis:** Check Task logs in CloudWatch first.

```bash
aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "<STREAM_NAME>" \
  --limit 100 \
  --output text
```

If the application starts cleanly, the health check path itself is the issue. Spring Security intercepts unauthenticated requests to `/actuator/health` and redirects them to `/login`. The ALB receives a `302`, follows it to `/login`, and marks the response as failed.

**Fix:** Update `SecurityConfig.java` to permit `/actuator/health` before the `anyRequest().authenticated()` rule. Rebuild the image, push the new tag to ECR, and force a new Service deployment.

```bash
# Force new deployment after pushing updated image
aws ecs update-service \
  --cluster $PROJECT \
  --service $PROJECT-svc \
  --force-new-deployment \
  --region $REGION
```

!!! note "Bare-metal vs ECS debug path"
    Bare-metal: SSH'd via Bastion into the private EC2 instance and ran `curl localhost:8000/actuator/health` directly.  
    ECS Fargate: Tasks are serverless — no SSH. Reproduce the same test by running a temporary Task with `assignPublicIp=ENABLED` in a public subnet, or read CloudWatch logs directly.

### Issue 2 — HTTPS Login Redirect Loop

**Symptom:** Application is reachable, but the login form produces an infinite redirect loop.

**Root cause:** The ALB terminates TLS and forwards plain HTTP to the Task on port 8000. Spring ignores the `X-Forwarded-Proto: https` header and generates `http://` post-login redirects. The browser upgrades these to `https://`, invalidating the session cookie on every redirect.

**Fix:** Add three properties to `application.properties` and enable ALB sticky sessions.

```properties
server.forward-headers-strategy=native
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.same-site=lax
```

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400
```

Rebuild, push, and force a new Service deployment.

!!! note "Root cause is identical across both deployments"
    This issue exists because the ALB terminates TLS in both bare-metal and ECS deployments.  
    The application fix and the sticky session configuration are the same — only the compute layer differs.

---

## Stage Summary

| Stage | Phases | Covers |
|---|---|---|
| **Stage 1** — Network & Security | Phase 1 → Phase 2 | VPC, Subnets, IGW, NAT, Route Tables, Bastion, Security Groups |
| **Stage 2** — Data & Container | Phase 3 → Phase 6 | RDS MySQL 8.4, ECR, ecsTaskExecutionRole, CloudWatch Log Group, ECS Cluster, Task Definition |
| **Stage 3** — Traffic, Scaling & Verification | Phase 7 → Phase 10 | Target Group, ALB, ACM, Cloudflare DNS, ECS Service, Health Checks, End-to-End Test |
| **Stage 4** — Troubleshooting | — | ALB health check failure, HTTPS login redirect loop |

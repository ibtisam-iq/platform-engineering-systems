# AWS Bare-Metal Deployment — Java Monolith Application

A production-grade, containerless deployment of the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) on AWS.

The application JAR runs directly on EC2 instances managed by an Auto Scaling Group, backed by Amazon RDS (MySQL 8.4), fronted by an Application Load Balancer, and routed through Route 53 with a verified TLS certificate.

> This is **Step 5A** of the overall DevOps implementation journey documented in the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) repository.

---

## Architecture Overview

```
Internet
   │
   ▼
Route 53  →  bankapp.ibtisam-iq.com
   │
   ▼
Application Load Balancer          ← sg-alb  (port 80 → redirect 443 | port 443 → forward)
   │
   ├── us-east-1a  (subnet-public-1a   10.0.0.0/26)    ── NAT Gateway
   └── us-east-1b  (subnet-public-1b   10.0.0.64/26)
   │
   ▼
Auto Scaling Group  (min 2 | desired 2 | max 4)
   ├── EC2 — us-east-1a  (subnet-private-1a  10.0.0.128/26)   ← sg-app  (port 8000 from ALB)
   └── EC2 — us-east-1b  (subnet-private-1b  10.0.0.192/26)
   │
   ▼
Amazon RDS — MySQL 8.4                                         ← sg-rds  (port 3306 from App)
(subnet-private-1a | Single-AZ | Multi-AZ ready by design)
```

### AWS Services

| Service | Role |
|---|---|
| VPC + Subnets | Isolated network — public/private separation across 2 AZs |
| Internet Gateway | Inbound internet access for public subnets |
| NAT Gateway | Outbound internet for private EC2 instances |
| Bastion Host | Secure SSH jump server into the private network |
| Security Groups | Least-privilege layer-4 traffic control between all tiers |
| Amazon RDS (MySQL 8.4) | Managed relational database in private subnet |
| S3 | Artifact storage for the application JAR |
| IAM Role + Instance Profile | Grants EC2 instances read access to S3 |
| Launch Template | EC2 configuration with full bootstrap user data |
| Auto Scaling Group | High availability across two AZs with automatic scaling |
| Application Load Balancer | HTTP/HTTPS traffic distribution across the ASG |
| ACM | TLS certificate for `bankapp.ibtisam-iq.com` |
| Route 53 | DNS alias record pointing to the ALB |

---

## Stage 1 — Network & Security Infrastructure

### Phase 1 — VPC, Subnets, and Routing

#### Environment Variables

All resource IDs were captured as shell variables to keep commands clean and reproducible across phases.

```bash
export REGION="us-east-1"
export AZ_A="us-east-1a"
export AZ_B="us-east-1b"
export PROJECT="java-monolith"
export AMI_ID="ami-091138d0f0d41ff90"
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

The `/24` block (`10.0.0.0/24`) was divided into four `/26` ranges — two public and two private, one per AZ.

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

The NAT Gateway was placed in the public subnet (`us-east-1a`) so that private EC2 instances could reach the internet for package installs and artifact downloads — without being directly reachable from outside.

```bash
EIP=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --region $REGION --output text)

aws ec2 describe-addresses --allocation-ids "$EIP" \
  --region $REGION --output table

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

A small EC2 instance was launched in the public subnet as the sole SSH entry point into the private network. Access was restricted to the operator's local IP only.

```bash
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

Three security groups were created with strict least-privilege rules. Each tier only accepts traffic from the tier directly in front of it — no exceptions.

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

# SG 2 — Application EC2 (port 8000 from ALB only)
SG_APP=$(aws ec2 create-security-group \
  --group-name "$PROJECT-app-sg" \
  --description "App EC2 Security Group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --protocol tcp --port 8000 --source-group $SG_ALB

# SG 3 — RDS MySQL (port 3306 from App EC2 only)
SG_RDS=$(aws ec2 create-security-group \
  --group-name "$PROJECT-rds-sg" \
  --description "RDS Security Group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_RDS \
  --protocol tcp --port 3306 --source-group $SG_APP
```

**Traffic flow enforced by security group chaining:**

```
Internet → ALB (80/443) → EC2 App (8000) → RDS (3306)
              ↑                 ↑                ↑
           sg-alb            sg-app           sg-rds
```

---

## Stage 2 — Data & Compute Layer

### Phase 3 — Database (Amazon RDS — MySQL 8.4)

#### RDS Subnet Group

The subnet group was built using both private subnets, following Multi-AZ best practice even though Single-AZ was selected for this deployment to control cost. The infrastructure is Multi-AZ ready by design.

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

Press `q` to quit the pager and get the prompt back. The RDS instance continues creating in the background regardless.

#### Database Initialization via Bastion

The RDS endpoint is not publicly accessible. The Bastion host was the only path to reach it — `mysql-client` was installed on the Bastion, and the database was initialized with the required user and privileges.

**Port 3306 — temporary inbound rule on `sg-rds`:**

By default, `sg-rds` only accepts port 3306 from `sg-app` (the application EC2 layer). The Bastion's outbound traffic is unrestricted (all traffic allowed), so the Bastion **can send** to port 3306 — but RDS **will not accept** it until the Bastion's security group is explicitly added as an inbound source on `sg-rds`.

A temporary inbound rule was added to `sg-rds` allowing port 3306 from `sg-bastion`, and removed immediately after initialization was complete.

```bash
# On local machine — add port 3306 temporarily to the RDS security group, sourced from Bastion SG
aws ec2 authorize-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 3306 \
  --source-group $BASTION_SG

# SSH into Bastion
ssh -i $PROJECT.pem ubuntu@<BASTION_PUBLIC_IP>

# On the Bastion host
sudo apt update -y && sudo apt install -y mysql-client

# Connect to RDS
mysql -h java-monolith-db.cfekoyoc2t2v.us-east-1.rds.amazonaws.com -u admin -p

# Inside MySQL shell
CREATE DATABASE IbtisamIQbankappdb;
CREATE USER 'your_db_user'@'%' IDENTIFIED BY 'your_db_password';
GRANT ALL PRIVILEGES ON IbtisamIQbankappdb.* TO 'your_db_user'@'%';
FLUSH PRIVILEGES;
EXIT;

# On local machine — remove the temporary rule from the RDS security group immediately after
aws ec2 revoke-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 3306 \
  --source-group $BASTION_SG
```

> After removal, `sg-rds` is back to its permanent state: port 3306 open from `sg-app` only.

---

### Phase 4 — Artifact Storage (S3)

The application JAR — built and published through the CI pipeline — was stored in S3. EC2 instances pull this artifact at boot time via the user data script using the IAM instance profile.

```bash
aws s3 mb s3://$PROJECT-artifacts --region $REGION

aws s3 cp target/bankapp-0.0.1-SNAPSHOT.jar \
  s3://$PROJECT-artifacts/bankapp-0.0.1-SNAPSHOT.jar
```

> The JAR is also available in the Nexus artifact registry (`nexus.ibtisam-iq.com`) and GitHub Packages. S3 was chosen for low-latency access from EC2 at instance launch time, and because IAM-based access requires zero credential management on the instance.

---

### Phase 5 — Launch Template

The Launch Template defined the complete EC2 configuration — OS, instance type, security group, IAM permissions, and the full bootstrap script — used by the ASG for every instance it launched.

#### Base Configuration

| Parameter | Value |
|---|---|
| AMI | Ubuntu LTS (same as Bastion) |
| Instance Type | `t3.medium` |
| Key Pair | Same key pair used across the project |
| Subnet / AZ | **Not configured** — intentionally left to the ASG |
| Security Group | `sg-app` (port 8000 from ALB only) |

> Subnet and AZ were deliberately excluded from the Launch Template. The ASG manages placement across `us-east-1a` and `us-east-1b` automatically — locking the template to a specific subnet would prevent proper multi-AZ distribution.

#### IAM Instance Profile

The EC2 instances needed S3 access at boot time to pull the application JAR. An IAM role with `AmazonS3ReadOnlyAccess` was created, wrapped in an instance profile, and attached to the Launch Template.

```bash
# Create IAM role
aws iam create-role \
  --role-name $PROJECT-ec2-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach S3 read-only policy
aws iam attach-role-policy \
  --role-name $PROJECT-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile and attach role
aws iam create-instance-profile \
  --instance-profile-name $PROJECT-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name $PROJECT-ec2-profile \
  --role-name $PROJECT-ec2-role
```

#### User Data Script

The user data script ran automatically on every instance at first boot. It installed all runtime dependencies, pulled the JAR from S3, set correct file ownership, exported all environment variables, and registered the application as a `systemd` service for process supervision and auto-restart.

```bash
#!/bin/bash
set -euo pipefail

# System update
apt update -y && apt upgrade -y

# Install Java 21
apt install -y openjdk-21-jre-headless

# Install AWS CLI
apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip /tmp/awscliv2.zip -d /tmp
/tmp/aws/install

# Pull artifact from S3
aws s3 cp s3://java-monolith-artifacts/bankapp-0.0.1-SNAPSHOT.jar \
  /home/ubuntu/bankapp.jar

# Set ownership
chown ubuntu:ubuntu /home/ubuntu/bankapp.jar

# Create systemd service unit
cat <<EOF > /etc/systemd/system/bankapp.service
[Unit]
Description=Java Monolith Banking Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="SPRING_APPLICATION_NAME=IbtisamIQBankApp"
Environment="SPRING_DATASOURCE_USERNAME=your_db_user"
Environment="SPRING_DATASOURCE_PASSWORD=your_db_password"
Environment="SPRING_DATASOURCE_URL=jdbc:mysql://java-monolith-db.cfekoyoc2t2v.us-east-1.rds.amazonaws.com:3306/IbtisamIQbankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true"
Environment="SERVER_PORT=8000"
ExecStart=/usr/bin/java -jar /home/ubuntu/bankapp.jar
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Register and start service
systemctl daemon-reload
systemctl enable bankapp
systemctl start bankapp
```

> All environment variables match the schema defined in [`.env.example`](https://github.com/ibtisam-iq/java-monolith-app/blob/main/.env.example). The only change from the local setup is that `SPRING_DATASOURCE_URL` points to the RDS endpoint instead of `localhost`.

#### Launch Template Creation

```bash
cat > /tmp/lt-data.json <<EOF
{
  "ImageId": "$AMI_ID",
  "InstanceType": "t3.medium",
  "KeyName": "$PROJECT",
  "SecurityGroupIds": ["$SG_APP"],
  "IamInstanceProfile": {
    "Name": "${PROJECT}-ec2-profile"
  },
  "UserData": "$(base64 -w 0 userdata.sh)"
}
EOF

LT_ID=$(aws ec2 create-launch-template \
  --launch-template-name "$PROJECT-lt" \
  --version-description "v1" \
  --launch-template-data file:///tmp/lt-data.json \
  --query 'LaunchTemplate.LaunchTemplateId' --output text)
```

---

## Stage 3 — Traffic, Scaling & Verification

### Phase 6 — Target Group

A Target Group was created before the ALB because the ALB listener rule references it by ARN.

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name "$PROJECT-tg" \
  --protocol HTTP \
  --port 8000 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --ip-address-type ipv4 \
  --protocol-version HTTP1 \
  --health-check-protocol HTTP \
  --health-check-path /actuator/health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

> **Port 8000, not 80:** The AWS Console defaults to port 80 for the Target Group. The application listens on port 8000 as defined in the `SERVER_PORT` environment variable. Using the wrong port here causes all health checks to fail and targets to remain permanently unhealthy — even if the application is running correctly.

> **Health check path:** `/actuator/health` is exposed by Spring Boot Actuator, which was added to `pom.xml` during the CI/CD pipeline phase specifically to support container and cloud health probes. It returns `{"status":"UP"}` when the application is ready to serve traffic.

---

### Phase 7 — Auto Scaling Group

The ASG was created using the Launch Template and placed across both private subnets — one per AZ — to achieve high availability. The ASG also registered instances automatically with the Target Group.

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "$PROJECT-asg" \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 2 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$PRIV_1A,$PRIV_1B" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 120
```

**Capacity configuration:**

| Parameter | Value | Reasoning |
|---|---|---|
| Minimum | 2 | One instance per AZ at all times — no single point of failure |
| Desired | 2 | Steady-state baseline |
| Maximum | 4 | Allows scale-out up to 2 additional instances under load |

> **Why private subnets?** EC2 instances are not internet-facing. All inbound traffic arrives through the ALB via `sg-alb → sg-app`. Private subnets mean no public IPs are assigned and the instances cannot be reached directly from the internet.

> **Alternative for testing only:** Instances can be placed in public subnets with auto-assigned public IPs, and `sg-app` temporarily opened to `0.0.0.0/0` on port 8000. This breaks the three-tier architecture but is a valid way to confirm the application starts correctly before introducing the ALB layer.

---

### Phase 8 — Application Load Balancer, ACM & Cloudflare DNS

#### ALB

The ALB was created as internet-facing, placed in both public subnets, with `sg-alb` attached. All three ALB variables were captured immediately after creation — `ALB_ARN` is required by both listeners, `ALB_DNS` is needed for the Cloudflare DNS record.

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
```

#### ACM Certificate

A TLS certificate for `bankapp.ibtisam-iq.com` was requested from AWS Certificate Manager using DNS validation. The domain `ibtisam-iq.com` is managed on Cloudflare — not Route 53 — so the validation CNAME was added manually in the Cloudflare dashboard.

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "bankapp.ibtisam-iq.com" \
  --validation-method DNS \
  --query 'CertificateArn' --output text)
```

The validation CNAME record was retrieved from ACM:

```bash
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord' \
  --output table
```

The `Name` and `Value` fields from the output were added as a CNAME record in **Cloudflare → ibtisam-iq.com → DNS**:

| Field | Value |
|-------|-------|
| Type | `CNAME` |
| Name | everything before `.ibtisam-iq.com.` in the `Name` field |
| Target | the full `Value` field |
| Proxy status | **DNS only (grey cloud)** — must not be proxied |

> Do not include the trailing dots while entering them in Cloudflare, even though AWS shows them in the table output.

Once the record was saved, ACM polled for it automatically. The following command blocked until the certificate status changed from `PENDING_VALIDATION` to `ISSUED`:

```bash
aws acm wait certificate-validated --certificate-arn $CERT_ARN
```

> The HTTPS listener cannot be created before this step completes. Attaching a `PENDING_VALIDATION` certificate to a listener causes `UnsupportedCertificate`.

#### Listeners

Port 80 permanently redirects all HTTP traffic to HTTPS. Port 443 forwards HTTPS traffic to the Target Group — the ALB handles TLS termination using the ACM certificate, so EC2 instances receive plain HTTP on port 8000 internally.

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

A CNAME record was added in **Cloudflare → ibtisam-iq.com → DNS** pointing `bankapp.ibtisam-iq.com` to the ALB:

| Field | Value |
|-------|-------|
| Type | `CNAME` |
| Name | `bankapp` |
| Target | value of `$ALB_DNS` |
| Proxy status | **DNS only (grey cloud)** |

> `ALB_HOSTED_ZONE` and Route 53 were removed entirely — the domain is on Cloudflare, so the Route 53 alias record approach does not apply. A plain CNAME to the ALB DNS name achieves the same result.

#### Session Recovery

If the shell session was interrupted between steps, all variables were restored before continuing:

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
```

---

### Phase 9 — Verification

#### Target Group Health

Once the ASG launched instances and the application started via systemd, the Target Group health checks confirmed both instances registered and passed.

```bash
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table
```

Expected output:
```
---------------------------------
|    DescribeTargetHealth       |
+--------------------+----------+
|  i-0abc123def456  | healthy  |
|  i-0xyz789ghi012  | healthy  |
+--------------------+----------+
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

#### ASG Multi-AZ Distribution

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "$PROJECT-asg" \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,AvailabilityZone]' \
  --output table
```

Expected: two instances in `InService` state — one in `us-east-1a` and one in `us-east-1b` — confirming high availability placement.

#### End-to-End Functional Test

The application was accessed via browser at `https://bankapp.ibtisam-iq.com`. A new user account was registered, login was verified, and transactional data was written to and read from the RDS MySQL database — confirming all three tiers (ALB → EC2 → RDS) were operating correctly end to end.

---

## Stage Summary

| Stage | Phases | Covers |
|---|---|---|
| **Stage 1** — Network & Security | Phase 1 → Phase 2 | VPC, Subnets, IGW, NAT, Route Tables, Bastion, Security Groups |
| **Stage 2** — Data & Compute | Phase 3 → Phase 5 | RDS MySQL 8.4, S3 Artifacts, Launch Template (IAM, User Data, systemd) |
| **Stage 3** — Traffic, Scaling & Verification | Phase 6 → Phase 9 | Target Group, ASG, ALB, ACM, Route 53, Health Checks, End-to-End Test |

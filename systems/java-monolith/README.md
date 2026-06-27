# Java Monolith Infrastructure (Platform Engineering)

This directory contains the complete Infrastructure-as-Code (IaC), Kubernetes manifests, and deployment documentation for the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app). 

Instead of just choosing a single deployment method, I intentionally designed this project as an **evolutionary DevOps journey**. The application is deployed across four increasingly advanced architectural paradigms to demonstrate deep expertise in AWS, Containerization, and Kubernetes.

## The Deployment Evolution

This repository proves the ability to manage infrastructure across the entire spectrum of cloud maturity—from traditional VMs to fully managed Kubernetes.

### 1. [EC2 Auto Scaling Deployment (Containerless)](./ec2-asg/)
The baseline architecture. The Java artifact is dynamically pulled from S3 at boot time via an IAM Instance Profile and executed directly on EC2 virtual machines. It demonstrates foundational expertise in AWS Networking (VPCs, Subnets), Auto Scaling Groups (ASG), and Application Load Balancers (ALB).

### 2. [Amazon ECS Fargate (Serverless Containers)](./ecs/)
The first step into containerization. The application is Dockerized and pushed to Amazon ECR. The underlying EC2 instances are entirely abstracted away using AWS Fargate, allowing the infrastructure to scale purely at the container level.

### 3. [Bare-Metal Kubernetes](./k8s/README.md#bare-metal-overlay-overlaysbare-metal)
The introduction to Kubernetes orchestration. Using Kustomize, the application is deployed to a local/bare-metal cluster. This implementation tackles the complex networking challenges of bare-metal environments, including configuring the modern **Gateway API** (via NGINX Gateway Fabric) and automating TLS certificates behind a NAT using `cert-manager`.

### 4. [Amazon EKS (Production Kubernetes)](./k8s/README.md#eks-overlay-overlayseks)
The final, production-grade cloud-native architecture. 
- **Infrastructure:** The entire EKS cluster and its VPC dependencies are provisioned immutably via **[Terraform](./terraform/)**.
- **Deployment:** The application is deployed via Kustomize. It leverages the AWS Load Balancer Controller for native ALB provisioning via the Gateway API, relies on the EBS CSI driver for stateful database storage (`gp3`), and scales dynamically via the HorizontalPodAutoscaler (HPA).

---

## Directory Structure

```text
java-monolith/
├── app/               # Git Submodule pointing to the application source code
├── docs/              # My Knowledge Base (Deep-dive architectural notes & runbooks)
├── ec2-asg/           # Architecture 1: EC2 + ASG + ALB + RDS (Containerless)
├── ecs/               # Architecture 2: ECS Fargate + ECR
├── k8s/               # Architecture 3 & 4: Kubernetes manifests (Base + Overlays)
└── terraform/         # IaC: Terraform modules for provisioning the EKS cluster
```

## Knowledge Base

While building these diverse environments, I heavily documented the edge-cases, troubleshooting steps, and architectural decisions I made. 

If you want to understand *why* I chose the Gateway API over Ingress, how I solved HTTP-01 challenges behind a NAT without a public IP, or how the Kustomize patches are structured, please refer to the **[Knowledge Base (`docs/`)](./docs/)**.

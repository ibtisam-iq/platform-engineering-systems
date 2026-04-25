# Platform Engineering Systems

> **Multiple applications. Multiple deployment targets. Multiple methods.**
> This repo is the **CD + Infrastructure side** of a two-repo GitOps architecture.
> Each application is fully self-contained — its own infra, its own manifests, its own config.

---

## Tech Stack

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECS%20%7C%20EKS%20%7C%20Lambda-FF9900?logo=amazonaws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Package%20Manager-0F1689?logo=helm&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white)
![CloudFormation](https://img.shields.io/badge/CloudFormation-AWS%20IaC-FF9900?logo=amazonaws&logoColor=white)

---

## What Is This Repository?

This is **not** an application repository.
This is a **platform engineering** repository — it answers the question:

> *"Given a built application artifact, how do you deploy, run, and manage it across real production environments?"*

Each `system` inside this repo represents one real application (linked as a Git submodule). That system folder contains **everything needed to deploy that application** — Kubernetes manifests, Terraform infrastructure code, Ansible configuration, ECS definitions, and CloudFormation stacks — all organized by deployment target.

**Every system is fully independent.** You can clone one system folder, follow its README, and have the application running — without touching any other system.

### Architecture Philosophy: Pattern A — App-Scoped Everything

This repo follows **Pattern A**: each application owns its own infrastructure code, its own Kubernetes manifests, and its own configuration management. There is no shared `infra/` at the root level.

**Why this approach?**

- Each app is an independent portfolio project. Its infra is provisioned, demoed, and torn down independently.
- The EC2 Auto Scaling Group for `java-monolith` has a different name, size, and config than one for `python-monolith`. These are not shared.
- A recruiter or reviewer can navigate to one system folder and understand the complete deployment story for that application — without needing context from other folders.
- This mirrors real-world platform teams where each product team owns their own deployment manifests and infra modules.

### Two-Repo GitOps Architecture

```
┌─────────────────────────────────────┐     ┌──────────────────────────────────────────┐
│         Application Repo            │     │      platform-engineering-systems        │
│     (e.g. java-monolith-app)        │     │                                          │
│                                     │     │  CD + Infrastructure Side                │
│  CI Side                            │     │  ────────────────────────────────────    │
│  ─────────────────────────────────  │     │  • Deploy to EC2, ECS, EKS, and more    │
│  • Build artifact (JAR/wheel/etc.)  │────▶│  • Provision infra (Terraform / CFN)    │
│  • Build Docker image               │     │  • Configure servers (Ansible)           │
│  • Run tests (SonarQube, Trivy)     │     │  • Orchestrate (Helm, ArgoCD)            │
│  • Push image to registry           │     │  • GitOps sync (ArgoCD watches this)     │
│    (ECR / Nexus / Docker Hub)       │     │                                          │
│  • Update image.env in THIS repo    │     │                                          │
└─────────────────────────────────────┘     └──────────────────────────────────────────┘
```

The CI pipeline lives in the application repo. It builds, tests, scans, and pushes the Docker image — then updates `image.env` in this repo. From that point, this repo takes over: provisioning infra, deploying manifests, and running the application.

---

## Repository Structure

```text
platform-engineering-systems/
│
├── README.md                          ← this file
├── LICENSE
├── .gitignore
├── .gitmodules                        ← all git submodule references
│
└── systems/                           ← one folder per application
    │
    ├── java-monolith/                 ← System: Spring Boot + MySQL banking app
    │   │
    │   ├── app                        ← Git submodule → java-monolith-app source
    │   ├── image.env                  ← image tag hand-off (CI writes, CD reads)
    │   │
    │   ├── k8s/                       ← Kubernetes manifests (Kustomize)
    │   │   ├── base/                  ← platform-agnostic core manifests
    │   │   │   ├── deployment.yaml
    │   │   │   ├── service.yaml
    │   │   │   ├── configmap.yaml
    │   │   │   ├── secret.yaml
    │   │   │   ├── mysql-deployment.yaml
    │   │   │   ├── mysql-service.yaml
    │   │   │   ├── mysql-pvc.yaml
    │   │   │   └── kustomization.yaml
    │   │   └── overlays/              ← platform-specific patches
    │   │       ├── local/             ← minikube / kind (NodePort, standard StorageClass)
    │   │       │   ├── ingress.yaml
    │   │       │   ├── storageclass-patch.yaml
    │   │       │   └── kustomization.yaml
    │   │       ├── eks/               ← AWS EKS (LoadBalancer, gp3, aws-load-balancer-controller)
    │   │       │   ├── ingress.yaml
    │   │       │   ├── service-patch.yaml
    │   │       │   ├── storageclass.yaml
    │   │       │   └── kustomization.yaml
    │   │       └── aks/               ← Azure AKS (future)
    │   │           ├── ingress.yaml
    │   │           └── kustomization.yaml
    │   │
    │   ├── ecs/                       ← AWS ECS (Fargate / EC2 launch type)
    │   │   ├── task-definition.json   ← ECS task definition for this app
    │   │   ├── service-definition.json
    │   │   └── README.md
    │   │
    │   ├── infra/                     ← Infrastructure as Code for this app
    │   │   │
    │   │   ├── terraform/             ← Terraform (provisions cloud infra)
    │   │   │   ├── modules/           ← reusable modules (optional — or reference shared registry)
    │   │   │   │   ├── eks-cluster/
    │   │   │   │   │   ├── main.tf
    │   │   │   │   │   ├── variables.tf
    │   │   │   │   │   └── outputs.tf
    │   │   │   │   ├── ec2-asg/
    │   │   │   │   │   ├── main.tf
    │   │   │   │   │   ├── variables.tf
    │   │   │   │   │   └── outputs.tf
    │   │   │   │   └── ecs-cluster/
    │   │   │   │       ├── main.tf
    │   │   │   │       ├── variables.tf
    │   │   │   │       └── outputs.tf
    │   │   │   └── envs/              ← environments that call modules with real values
    │   │   │       ├── eks/           ← provision EKS cluster for this app
    │   │   │       │   ├── main.tf
    │   │   │       │   ├── variables.tf
    │   │   │       │   ├── outputs.tf
    │   │   │       │   └── terraform.tfvars
    │   │   │       ├── ec2-asg/       ← provision EC2 Auto Scaling Group for this app
    │   │   │       │   ├── main.tf
    │   │   │       │   ├── variables.tf
    │   │   │       │   └── terraform.tfvars
    │   │   │       └── ecs/           ← provision ECS cluster for this app
    │   │   │           ├── main.tf
    │   │   │           └── terraform.tfvars
    │   │   │
    │   │   └── cloudformation/        ← AWS-native IaC (alternative to Terraform)
    │   │       ├── ecs-stack.yaml     ← ECS cluster + ALB + service
    │   │       └── ec2-asg-stack.yaml ← Auto Scaling Group + Launch Template
    │   │
    │   ├── config/                    ← Server configuration (Ansible)
    │   │   ├── inventory/
    │   │   │   └── dev.ini            ← EC2 host inventory for this app
    │   │   ├── roles/
    │   │   │   ├── common/            ← baseline packages, users, SSH hardening
    │   │   │   └── docker/            ← installs and configures Docker
    │   │   └── playbooks/
    │   │       ├── setup-ec2.yml      ← provisions a fresh EC2 instance
    │   │       └── deploy.yml         ← deploys this app onto EC2
    │   │
    │   ├── assets/                    ← architecture diagrams, screenshots
    │   └── README.md                  ← full walkthrough for this system
    │
    └── python-monolith/               ← System: next application (same structure)
        ├── app                        ← Git submodule → python-monolith-app source
        ├── image.env
        ├── k8s/
        │   ├── base/
        │   └── overlays/
        │       ├── local/
        │       └── eks/
        ├── ecs/
        ├── infra/
        │   ├── terraform/
        │   └── cloudformation/
        ├── config/
        └── README.md
```

> Every new application added to this repo follows this exact same structure.
> A reviewer navigates to one system folder and finds the complete deployment story — no cross-system dependencies.

---

## Deployment Targets & Methods

The table below shows all supported deployment targets and the tools used per target.
Each cell marked ✅ is independently runnable from its own subfolder.

| Target | Terraform | CloudFormation | Ansible | kubectl (Kustomize) | Helm | ArgoCD (GitOps) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **EC2** (with ASG + ALB) | ✅ | ✅ | ✅ | — | — | — |
| **ECS** (Fargate / EC2) | ✅ | ✅ | — | — | — | — |
| **EKS** — local overlay | ✅ | — | — | ✅ | ✅ | ✅ |
| **EKS** — AWS overlay | ✅ | — | — | ✅ | ✅ | ✅ |
| **AKS** — Azure overlay | ✅ | — | — | ✅ | ✅ | ✅ |

> This matrix grows as new systems and deployment targets are added to `systems/`.

### Target Notes

- **EC2 + Auto Scaling Group** — VM-based deployment. Terraform (or CloudFormation) provisions the ASG, Launch Template, ALB, and Security Groups. Ansible configures the instances (installs Docker, pulls image, sets up systemd). Each app has its own ASG with its own name and sizing.
- **ECS (Fargate / EC2 launch type)** — Fully containerized. Task definitions and service definitions live in `ecs/`. Terraform or CloudFormation provisions the ECS cluster and wires up the ALB.
- **EKS (local)** — Kubernetes on a local cluster (minikube/kind). Kustomize `overlays/local/` patches base manifests with NodePort service type and standard StorageClass.
- **EKS (AWS)** — Kubernetes on AWS EKS. Terraform provisions the cluster. Kustomize `overlays/eks/` patches with LoadBalancer service type, gp3 StorageClass, and AWS Load Balancer Controller ingress annotations.
- **AKS (Azure)** — Kubernetes on Azure AKS. Same Kustomize base; `overlays/aks/` patches for Azure-specific ingress and storage.

---

## How Kustomize Base/Overlays Works

The Kubernetes manifests use Kustomize to avoid duplicating YAML for each platform.

```
k8s/base/           ← ~80% of manifest content — platform-agnostic
    deployment.yaml         same across local, EKS, AKS
    configmap.yaml          same structure; only SPRING_DATASOURCE_URL changes
    mysql-pvc.yaml          same spec; StorageClass is patched per overlay

k8s/overlays/local/         patches for minikube/kind
    service type  → NodePort
    StorageClass  → standard
    ingress class → nginx

k8s/overlays/eks/           patches for AWS EKS
    service type  → LoadBalancer (AWS NLB)
    StorageClass  → gp3 (EBS)
    ingress class → aws-load-balancer-controller
    DATASOURCE URL → RDS endpoint (from Terraform output)

k8s/overlays/aks/           patches for Azure AKS
    service type  → LoadBalancer (Azure LB)
    StorageClass  → managed-premium
    ingress class → azure/application-gateway
```

**Deploy to a target:**

```bash
# local cluster
kubectl apply -k systems/java-monolith/k8s/overlays/local/

# AWS EKS
kubectl apply -k systems/java-monolith/k8s/overlays/eks/

# Azure AKS
kubectl apply -k systems/java-monolith/k8s/overlays/aks/
```

---

## Terraform: Modules vs Environments

Inside each system's `infra/terraform/`, there are two layers:

```
modules/        ← reusable logic (what to build)
    eks-cluster/
    ec2-asg/
    ecs-cluster/

envs/           ← real values (where and how to build it)
    eks/
        terraform.tfvars    ← cluster name, region, node size for THIS app
    ec2-asg/
        terraform.tfvars    ← ASG name, instance type, min/max for THIS app
```

The `modules/` code is structurally identical across apps. Only `terraform.tfvars` differs — this is where `java-monolith` gets its own cluster name, tags, and sizing, separate from `python-monolith`.

**Provision infrastructure:**

```bash
cd systems/java-monolith/infra/terraform/envs/eks
terraform init
terraform plan
terraform apply
```

---

## The CI → CD Hand-Off

```
CI (Application Repo — Jenkins / GitHub Actions)
    1. Code pushed to java-monolith-app
    2. Build JAR → Docker image
    3. Run tests (JUnit, SonarQube)
    4. Trivy image scan
    5. Push image to registry (DockerHub / ECR / Nexus)
    6. Update systems/java-monolith/image.env in THIS repo
              │
              │  git commit: "ci: update java-monolith image to sha-abc123"
              ▼
CD (This Repo — platform-engineering-systems)
    EKS  → ArgoCD detects image.env change → syncs overlays/eks/ → rolling update
    EC2  → Ansible pull new image tag → restart container via systemd
    ECS  → Terraform updates task definition with new image tag → ECS rolls
```

`image.env` is the single source of truth for which image version is running where.

---

## GitOps Flow

```
Developer pushes code to app repo
              │
              ▼
   ┌─────────────────────────┐
   │    Application Repo CI  │
   │  Jenkins / GitHub Actions│
   │  ─────────────────────── │
   │  1. Build artifact       │
   │  2. Docker build & push  │
   │  3. Tests + Trivy scan   │
   │  4. Update image.env     │
   └────────┬────────────────┘
            │  git commit → platform-engineering-systems
            ▼
   ┌──────────────────────────────────┐
   │   platform-engineering-systems   │
   │   ──────────────────────────────  │
   │   EKS  → ArgoCD syncs manifests  │
   │   EC2  → Ansible re-deploys      │
   │   ECS  → Terraform updates task  │
   └────────┬─────────────────────── ┘
            │
            ▼
   ┌────────────────────────┐
   │   Target Environment   │
   │   App running live     │
   └────────────────────────┘
```

---

## Current Systems

### [java-monolith](./systems/java-monolith/)

| Property | Detail |
|---|---|
| **Application** | Spring Boot REST API — banking/hospital management system |
| **Language** | Java 17 + Maven |
| **Database** | MySQL 8 |
| **Source Repo** | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) (submodule) |
| **CI Pipeline** | Jenkins (DevSecOps) + GitHub Actions (mirrored) |
| **Deployment Targets** | Local K8s · EKS · EC2 ASG · ECS |
| **IaC Tools** | Terraform · CloudFormation |
| **K8s Tooling** | Kustomize · Helm · ArgoCD |
| **Config Management** | Ansible |

> More systems will be added as new applications are onboarded.

---

## Getting Started

### Prerequisites

```bash
git --version          # Git
docker --version       # Docker
terraform --version    # Terraform >= 1.5
kubectl version        # kubectl
helm version           # Helm >= 3
aws --version          # AWS CLI v2
ansible --version      # Ansible (for EC2 targets)
```

### Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/ibtisam-iq/platform-engineering-systems.git
cd platform-engineering-systems
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Navigate to a Deployment

Each target folder is self-contained with its own README. Example workflows:

```bash
# Deploy java-monolith to local Kubernetes cluster
kubectl apply -k systems/java-monolith/k8s/overlays/local/

# Deploy java-monolith to AWS EKS (after Terraform provisions the cluster)
cd systems/java-monolith/infra/terraform/envs/eks && terraform apply
kubectl apply -k systems/java-monolith/k8s/overlays/eks/

# Deploy java-monolith to EC2 via Ansible
cd systems/java-monolith/config
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml
```

---

## Engineering Principles

- **App-scoped everything** — each system folder owns its own infra, manifests, and config; no cross-system dependencies
- **GitOps-ready** — ArgoCD watches this repo; `image.env` is the CI→CD hand-off contract
- **Kustomize base/overlays** — manifests written once, patched per platform; no YAML duplication
- **Terraform modules + envs** — reusable module logic, app-specific values in `terraform.tfvars`
- **Method isolation** — each deployment method (Terraform, Ansible, kubectl) lives in its own subfolder and is independently runnable
- **Platform-agnostic by design** — currently AWS; new platforms (Azure, GCP, bare metal) added via new overlays without touching existing ones
- **Reproducible** — every deployment can be torn down and rebuilt from scratch using the code in this repo

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | This repo — CD + Infrastructure |
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Spring Boot application source + CI pipelines |

---

## Author

**Muhammad Ibtisam Iqbal**
DevOps Engineer · Platform Engineering · Kubernetes (CKA + CKAD)
[GitHub](https://github.com/ibtisam-iq) · [Website](https://ibtisam-iq.com)

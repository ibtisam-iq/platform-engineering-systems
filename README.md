# Platform Engineering Systems

> **One application. Multiple AWS targets. Multiple deployment methods.**  
> This repo is the **CD + Infrastructure side** of a two-repo GitOps architecture.  
> It takes a built artifact and deploys it to EC2, ECS, and EKS — using Bash, Ansible, Terraform, kubectl, Helm, and ArgoCD.

---

## Tech Stack

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECS%20%7C%20EKS-FF9900?logo=amazonaws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Package%20Manager-0F1689?logo=helm&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white)

---

## What Is This Repository?

This is **not** an application repository.  
This is a **platform engineering** repository — it answers the question:

> *"Given a built application artifact, how do you deploy, run, and manage it in production-like AWS environments?"*

Each system inside this repo takes a real application (linked as a Git submodule) and demonstrates deploying it across multiple AWS targets using multiple tools and methods — the way real platform teams operate.

### Two-Repo Architecture

```
┌─────────────────────────────────────┐     ┌──────────────────────────────────────────┐
│       devsecops-pipelines           │     │      platform-engineering-systems        │
│                                     │     │                                          │
│  CI Side                            │     │  CD + Infrastructure Side                │
│  ─────────────────────────────────  │     │  ────────────────────────────────────    │
│  • Build JAR / Docker image         │────▶│  • Deploy to EC2, ECS, EKS               │
│  • Run tests (SonarQube, Trivy)     │     │  • Provision infra (Terraform)           │
│  • Push image to ECR                │     │  • Orchestrate (Helm, ArgoCD)            │
│  • Update image tag here            │     │  • GitOps sync (ArgoCD watches this)     │
└─────────────────────────────────────┘     └──────────────────────────────────────────┘
```

👉 CI/CD Pipelines repo: [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines)

---

## Deployment Matrix

The table below shows which target × method combinations are implemented per system.

| Target | Bash / Scripts | Ansible | Terraform | kubectl | Helm | ArgoCD (GitOps) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **EC2** (with ASG + ALB) | ✅ | ✅ | ✅ | — | — | — |
| **ECS** (Fargate / EC2) | — | — | ✅ | — | — | — |
| **EKS** (Kubernetes on AWS) | — | — | ✅ | ✅ | ✅ | ✅ |

> Each method lives in its own subfolder under the target folder.  
> No method bleeds into another — each is independently runnable.

---

## Repository Structure

```text
platform-engineering-systems/
│
├── systems/
│   │
│   ├── java-monolith/                  ← Spring Boot monolith (Java)
│   │   ├── app/                        ← Git submodule: application source code
│   │   │
│   │   ├── ec2/                        ← Target 1: AWS EC2 + Auto Scaling Group
│   │   │   ├── bash/                   ← userdata scripts, bootstrap automation
│   │   │   ├── ansible/                ← Ansible playbook for provisioning
│   │   │   ├── terraform/              ← IaC: EC2, ASG, ALB, Security Groups
│   │   │   └── README.md
│   │   │
│   │   ├── ecs/                        ← Target 2: AWS ECS (containerized)
│   │   │   ├── aws-cli/                ← task-definition.json + service.json
│   │   │   ├── terraform/              ← IaC: ECS cluster, ECR, task, service
│   │   │   └── README.md
│   │   │
│   │   ├── eks/                        ← Target 3: AWS EKS (Kubernetes)
│   │   │   ├── manifests/              ← raw Kubernetes YAML (kubectl apply)
│   │   │   ├── helm/                   ← Helm chart for the application
│   │   │   ├── argocd/                 ← GitOps: ArgoCD Application manifest
│   │   │   ├── terraform/              ← IaC: EKS cluster + node groups
│   │   │   └── README.md
│   │   │
│   │   ├── assets/                     ← architecture diagrams, run screenshots
│   │   ├── docs/                       ← written walkthroughs per deployment target
│   │   └── README.md                   ← project-level overview
│   │
│   ├── node-monolith/                  ← Node.js monolith (coming soon)
│   └── python-monolith/               ← Python Flask monolith (coming soon)
│
├── shared/                             ← shared configs, base Helm values, reusable modules
├── docs/                               ← cross-system documentation
└── README.md
```

---

## GitOps Flow

How a code change travels from developer commit to running pod on EKS:

```
Developer pushes code to app repo
              │
              ▼
   ┌─────────────────────┐
   │  devsecops-pipelines │  (CI)
   │  Jenkins / GH Actions│
   │  ─────────────────── │
   │  1. Build JAR        │
   │  2. Docker build     │
   │  3. Trivy scan       │
   │  4. Push to ECR      │
   │  5. Update image tag │
   │     in THIS repo     │
   └────────┬────────────┘
            │  git commit → platform-engineering-systems
            ▼
   ┌──────────────────────────────┐
   │  platform-engineering-systems│  (CD + Infra)
   │  ────────────────────────── │
   │  ArgoCD watches this repo   │
   │  Detects image tag change   │
   │  Syncs manifests to EKS     │
   └────────┬─────────────────── ┘
            │
            ▼
   ┌────────────────────┐
   │  AWS EKS Cluster   │
   │  App pod updated   │
   │  Zero downtime     │
   └────────────────────┘
```

---

## Systems

### Java Monolith

| Property | Detail |
|---|---|
| **Application** | Spring Boot REST API with MySQL |
| **Source** | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) (submodule) |
| **Deployment Targets** | EC2, ECS, EKS |
| **IaC Tool** | Terraform |
| **Orchestration** | Kubernetes (kubectl, Helm, ArgoCD) |
| **CI Pipeline** | Jenkins + GitHub Actions |

### Node.js Monolith *(coming soon)*
### Python Monolith *(coming soon)*

---

## Getting Started

### Prerequisites

Make sure the following tools are installed:

```bash
git --version          # Git
docker --version       # Docker
terraform --version    # Terraform >= 1.5
kubectl version        # kubectl
helm version           # Helm >= 3
aws --version          # AWS CLI v2
ansible --version      # Ansible (for EC2 method)
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

Each target folder is self-contained. Example — deploy to EKS using Helm:

```bash
cd systems/java-monolith/eks/helm
# follow the README.md inside
```

---

## Engineering Principles

- **Target-first structure** — folders organised by *where* the app runs, not by tool
- **Method isolation** — each deployment method is independently runnable
- **Submodule separation** — app source code lives in its own repo; this repo owns operations
- **GitOps-ready** — ArgoCD watches this repo for changes; infrastructure is code, not console
- **Reproducible** — every deployment can be torn down and rebuilt from scratch

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | This repo — CD + Infrastructure |
| [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines) | CI pipelines (Jenkins, GitHub Actions) |
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Spring Boot application source |

---

## Author

**Muhammad Ibtisam Iqbal**  
DevOps Engineer · Cloud Infrastructure · Kubernetes  
[GitHub](https://github.com/ibtisam-iq) · [LinkedIn](https://linkedin.com/in/ibtisam-iq)

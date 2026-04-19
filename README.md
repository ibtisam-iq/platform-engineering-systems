# Platform Engineering Systems

> **One application. Multiple deployment targets. Multiple methods.**
> This repo is the **CD + Infrastructure side** of a two-repo GitOps architecture.
> It takes a built artifact and deploys it across EC2, ECS, EKS, Lambda, Bare Metal, and more — using Bash, Ansible, Terraform, kubectl, Helm, and ArgoCD.

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

---

## What Is This Repository?

This is **not** an application repository.
This is a **platform engineering** repository — it answers the question:

> *"Given a built application artifact, how do you deploy, run, and manage it across real production environments?"*

Currently implemented on **AWS** — because that is what is known right now.
The structure is intentionally platform-agnostic: new deployment targets (Azure, GCP, bare metal, on-prem) can be added under any system as they are learned.

Each system inside this repo takes a real application (linked as a Git submodule) and demonstrates deploying it across multiple targets using multiple tools and methods — the way real platform teams operate.

### Two-Repo Architecture

```
┌─────────────────────────────────────┐     ┌──────────────────────────────────────────┐
│       devsecops-pipelines           │     │      platform-engineering-systems        │
│                                     │     │                                          │
│  CI Side                            │     │  CD + Infrastructure Side                │
│  ─────────────────────────────────  │     │  ────────────────────────────────────    │
│  • Build artifact (JAR/wheel/etc.)  │────▶│  • Deploy to EC2, ECS, EKS, Lambda, … │
│  • Build Docker image               │     │  • Provision infra (Terraform)           │
│  • Run tests (SonarQube, Trivy)     │     │  • Orchestrate (Helm, ArgoCD)            │
│  • Push image to registry           │     │  • GitOps sync (ArgoCD watches this)     │
│    (ECR / Nexus / Docker Hub)       │     │                                          │
│  • Update image tag here            │     │                                          │
└─────────────────────────────────────┘     └──────────────────────────────────────────┘
```

👉 CI/CD Pipelines repo: [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines)

---

## Deployment Targets & Methods

The table below shows all supported deployment targets and the methods available for each.
Each cell marked ✅ is independently runnable from its own subfolder.

| Target | Bash / Scripts | Ansible | Terraform | kubectl | Helm | ArgoCD (GitOps) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **EC2** (with ASG + ALB) | ✅ | ✅ | ✅ | — | — | — |
| **ECS** (Fargate / EC2) | — | — | ✅ | — | — | — |
| **EKS** (Kubernetes on AWS) | — | — | ✅ | ✅ | ✅ | ✅ |
| **Bare Metal** (self-hosted) | ✅ | ✅ | — | — | — | — |
| **Lambda** (serverless) | ✅ | — | ✅ | — | — | — |
| **Elastic Beanstalk** | ✅ | — | ✅ | — | — | — |
| **App Runner** | ✅ | — | ✅ | — | — | — |

> This matrix grows as new systems and deployment targets are added.
> Each method lives in its own subfolder — no method bleeds into another.

### Target Notes

- **EC2 + Auto Scaling Group** — VM-based deployment with ALB for load balancing, ASG for horizontal scaling, and a custom domain. Provisioned via Bash userdata, Ansible playbooks, or Terraform.
- **ECS (Fargate / EC2 launch type)** — Fully containerized. Task definitions and services managed via Terraform or AWS CLI.
- **EKS** — Kubernetes on AWS. Supports raw manifests (kubectl), packaged deployments (Helm), and fully automated GitOps sync (ArgoCD).
- **Bare Metal** — Direct deployment onto self-hosted Linux servers. No cloud provider needed. SSH-based provisioning via Bash or Ansible, running as systemd services or Docker containers.
- **Lambda** — Serverless execution. Artifact deployed as a ZIP package or container image. Triggered by events (API Gateway, S3, SQS, etc.). Provisioned via AWS CLI scripts or Terraform.
- **Elastic Beanstalk** — AWS-managed PaaS. Upload a WAR, JAR, or Docker image — Beanstalk handles the underlying EC2, ASG, and ALB automatically.
- **App Runner** — Fully managed container runtime. Point at an ECR image; App Runner handles scaling, load balancing, and TLS with zero infrastructure management.

---

## Repository Structure

```text
platform-engineering-systems/
│
├── systems/
│   │
│   ├── <system-name>/              ← any application, any language
│   │   ├── app/                    ← Git submodule: application source code
│   │   │
│   │   ├── ec2/                    ← Target: EC2 + Auto Scaling Group + ALB
│   │   │   ├── bash/               ← userdata scripts, bootstrap automation
│   │   │   ├── ansible/            ← Ansible playbook for provisioning
│   │   │   ├── terraform/          ← IaC: EC2, ASG, ALB, Security Groups
│   │   │   └── README.md
│   │   │
│   │   ├── ecs/                    ← Target: AWS ECS (containerized)
│   │   │   ├── aws-cli/            ← task-definition.json + service.json
│   │   │   ├── terraform/          ← IaC: ECS cluster, task, service
│   │   │   └── README.md
│   │   │
│   │   ├── eks/                    ← Target: AWS EKS (Kubernetes)
│   │   │   ├── manifests/          ← raw Kubernetes YAML (kubectl apply)
│   │   │   ├── helm/               ← Helm chart for the application
│   │   │   ├── argocd/             ← GitOps: ArgoCD Application manifest
│   │   │   ├── terraform/          ← IaC: EKS cluster + node groups
│   │   │   └── README.md
│   │   │
│   │   ├── bare-metal/             ← Target: self-hosted Linux servers
│   │   │   ├── bash/               ← SSH-based deploy scripts
│   │   │   ├── ansible/            ← Ansible for provisioning + deploy
│   │   │   └── README.md
│   │   │
│   │   ├── lambda/                 ← Target: AWS Lambda (serverless)
│   │   │   ├── scripts/            ← package + deploy scripts
│   │   │   ├── terraform/          ← IaC: Lambda function, IAM, triggers
│   │   │   └── README.md
│   │   │
│   │   ├── elastic-beanstalk/      ← Target: AWS Elastic Beanstalk (PaaS)
│   │   │   ├── scripts/            ← eb cli or deploy scripts
│   │   │   ├── terraform/          ← IaC: Beanstalk environment
│   │   │   └── README.md
│   │   │
│   │   ├── app-runner/             ← Target: AWS App Runner
│   │   │   ├── scripts/            ← deploy scripts
│   │   │   ├── terraform/          ← IaC: App Runner service
│   │   │   └── README.md
│   │   │
│   │   ├── assets/                 ← architecture diagrams, screenshots
│   │   ├── docs/                   ← written walkthroughs per target
│   │   └── README.md               ← system-level overview
│   │
│   └── <more-systems>/             ← added progressively as new apps are onboarded
│
├── shared/                         ← shared configs, base Helm values, reusable Terraform modules
├── docs/                           ← cross-system documentation
└── README.md
```

> The `systems/` folder is open-ended. Any application in any language can be onboarded here.
> Each system is fully self-contained with its own submodule, deployment targets, and docs.

---

## GitOps Flow

A code change travels from developer commit to running deployment:

```
Developer pushes code to app repo
              │
              ▼
   ┌─────────────────────────┐
   │    devsecops-pipelines  │  (CI)
   │  Jenkins / GitHub Actions│
   │  ─────────────────────── │
   │  1. Build artifact       │
   │     (JAR / wheel / etc.) │
   │  2. Docker build         │
   │  3. Run tests            │
   │  4. Trivy scan           │
   │  5. Push to registry     │
   │     (ECR / Nexus)        │
   │  6. Update image tag     │
   │     in THIS repo         │
   └────────┬────────────────┘
            │  git commit → platform-engineering-systems
            ▼
   ┌──────────────────────────────────┐
   │   platform-engineering-systems   │  (CD + Infra)
   │   ──────────────────────────────  │
   │   Each target has its own flow:  │
   │                                  │
   │   EKS  → ArgoCD detects change  │
   │          syncs manifests         │
   │                                  │
   │   EC2  → Ansible / Terraform    │
   │          provisions + deploys    │
   │                                  │
   │   ECS  → Terraform updates      │
   │          task + service          │
   │                                  │
   │   Lambda → scripts / Terraform  │
   │            update function code  │
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

### Java Monolith

| Property | Detail |
|---|---|
| **Application** | Spring Boot REST API with MySQL |
| **Source** | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) (submodule) |
| **Deployment Targets** | EC2, ECS, EKS |
| **IaC Tool** | Terraform |
| **Orchestration** | Kubernetes (kubectl, Helm, ArgoCD) |
| **CI Pipeline** | Jenkins + GitHub Actions |

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
ansible --version      # Ansible (for EC2 + bare-metal methods)
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

- **Target-first structure** — folders organized by *where* the app runs, not by tool
- **Method isolation** — each deployment method is independently runnable
- **Platform-agnostic by design** — currently AWS; new platforms added as learned
- **Submodule separation** — app source lives in its own repo; this repo owns operations
- **GitOps-ready** — ArgoCD watches this repo; each target's CD flow is explicit
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

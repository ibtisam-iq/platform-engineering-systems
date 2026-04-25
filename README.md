# Platform Engineering Systems

> **Multiple applications. Multiple deployment targets. Multiple methods.**
> This repo is the **CD + Infrastructure side** of a two-repo GitOps architecture.
> Each application lives in its own `systems/` subfolder — fully self-contained, independently deployable.

---

## Tech Stack

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECS%20%7C%20EKS-FF9900?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Package%20Manager-0F1689?logo=helm&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Kustomize](https://img.shields.io/badge/Kustomize-Manifests-326CE5?logo=kubernetes&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Configuration-EE0000?logo=ansible&logoColor=white)
![CloudFormation](https://img.shields.io/badge/CloudFormation-AWS%20IaC-FF9900?logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)

---

## What Is This Repository?

This is **not** an application repository. It is a **platform engineering** repository — it answers the question:

> *"Given a built Docker image, how do you deploy, run, and manage it across real production environments?"*

The CI pipeline lives in the **application repository** (build, test, scan, push image). This repository takes over from that point: it provisions infrastructure, applies Kubernetes manifests, configures servers, and manages the full lifecycle of running applications in production.

---

## Architecture: Two-Repo GitOps

```
┌──────────────────────────────────────┐       ┌───────────────────────────────────────────┐
│         Application Repo             │       │       platform-engineering-systems        │
│      (e.g. java-monolith-app)        │       │                                           │
│                                      │       │   CD + Infrastructure Side                │
│   CI Side                            │       │   ─────────────────────────────────────   │
│   ──────────────────────────────     │       │   • Deploy to EC2 ASG, ECS, EKS, AKS     │
│   • Build artifact (JAR)             │──────▶│   • Provision infra (Terraform / CFN)    │
│   • Build & tag Docker image         │       │   • Configure servers (Ansible)           │
│   • Run tests (JUnit, SonarQube)     │       │   • Orchestrate (Kustomize, Helm)         │
│   • Scan image (Trivy)               │       │   • GitOps sync (ArgoCD watches this)     │
│   • Push image (DockerHub/ECR/Nexus) │       │   • image.env = CI→CD hand-off contract   │
│   • Update image.env in THIS repo ───┼──────▶│                                           │
└──────────────────────────────────────┘       └───────────────────────────────────────────┘
```

The `image.env` file inside each system folder is the **hand-off contract**: the CI pipeline writes the new image tag there, and the CD side (ArgoCD, Ansible, or Terraform) picks it up.

---

## Design Philosophy: Pattern A — App-Scoped Everything

This repo follows **Pattern A**: each application owns its own infrastructure code, its own Kubernetes manifests, and its own configuration. There is no shared `infra/` at the root level.

**Why this approach?**

- Each system is an independent portfolio project — provisioned, demoed, and torn down independently.
- A recruiter or reviewer navigates to one `systems/` folder and finds the complete deployment story for that one application, without needing context from anything else.
- The EC2 Auto Scaling Group for `java-monolith` has its own name, size, and configuration — completely separate from `python-monolith`. Sharing infra modules would create coupling where none is needed.
- This mirrors real-world platform teams where each product team owns their deployment manifests and infra modules.

---

## Repository Structure

```text
platform-engineering-systems/
│
├── README.md                          ← this file
├── LICENSE
├── .gitignore
├── .gitmodules                        ← all git submodule references live here
│
└── systems/                           ← one folder per application
    │
    ├── java-monolith/                 ← System: Spring Boot + MySQL application
    │   ├── app/                       ← Git submodule → java-monolith-app (source + CI)
    │   ├── image.env                  ← CI writes image tag here; CD reads it
    │   │
    │   ├── k8s/                       ← Kubernetes manifests (Kustomize base/overlays)
    │   │   ├── base/                  ← platform-agnostic core manifests (~80% shared)
    │   │   │   ├── deployment.yaml
    │   │   │   ├── service.yaml       ← ClusterIP (no cloud assumptions)
    │   │   │   ├── configmap.yaml     ← env vars (SPRING_DATASOURCE_URL, SERVER_PORT, etc.)
    │   │   │   ├── secret.yaml        ← template only; real values via Secrets Manager / Vault
    │   │   │   ├── mysql-deployment.yaml
    │   │   │   ├── mysql-service.yaml
    │   │   │   ├── mysql-pvc.yaml
    │   │   │   └── kustomization.yaml
    │   │   └── overlays/              ← platform-specific patches (only what changes)
    │   │       ├── local/             ← minikube / kind
    │   │       │   ├── ingress.yaml             ← nginx ingress
    │   │       │   ├── storageclass-patch.yaml  ← standard StorageClass
    │   │       │   └── kustomization.yaml
    │   │       ├── eks/               ← AWS EKS
    │   │       │   ├── ingress.yaml             ← aws-load-balancer-controller
    │   │       │   ├── service-patch.yaml       ← LoadBalancer type + AWS annotations
    │   │       │   ├── storageclass.yaml        ← gp3 (EBS)
    │   │       │   └── kustomization.yaml
    │   │       └── aks/               ← Azure AKS (planned)
    │   │           ├── ingress.yaml
    │   │           └── kustomization.yaml
    │   │
    │   ├── infra/                     ← Infrastructure as Code for this application
    │   │   ├── terraform/
    │   │   │   ├── modules/           ← reusable Terraform modules
    │   │   │   │   ├── eks-cluster/
    │   │   │   │   ├── ec2-asg/
    │   │   │   │   └── ecs-cluster/
    │   │   │   └── envs/              ← environment configs that call modules with real values
    │   │   │       ├── eks/           ← terraform.tfvars for this app's EKS cluster
    │   │   │       ├── ec2-asg/       ← terraform.tfvars for this app's ASG
    │   │   │       └── ecs/           ← terraform.tfvars for this app's ECS cluster
    │   │   └── cloudformation/        ← AWS-native IaC alternative
    │   │       ├── ecs-stack.yaml
    │   │       └── ec2-asg-stack.yaml
    │   │
    │   ├── ecs/                       ← AWS ECS definitions (Fargate / EC2 launch type)
    │   │   ├── task-definition.json
    │   │   ├── service-definition.json
    │   │   └── README.md
    │   │
    │   ├── config/                    ← Server configuration (Ansible)
    │   │   ├── inventory/dev.ini      ← EC2 host inventory for this application
    │   │   ├── roles/
    │   │   │   ├── common/            ← baseline packages, users, SSH hardening
    │   │   │   └── docker/            ← installs and configures Docker
    │   │   └── playbooks/
    │   │       ├── setup-ec2.yml      ← provisions a fresh EC2 instance
    │   │       └── deploy.yml         ← deploys the application onto EC2
    │   │
    │   ├── assets/                    ← architecture diagrams, screenshots
    │   └── README.md                  ← full deployment walkthrough for this system
    │
    └── python-monolith/               ← System: next application (same structure, coming soon)
        ├── app/
        ├── image.env
        ├── k8s/
        │   ├── base/
        │   └── overlays/
        │       ├── local/
        │       └── eks/
        ├── infra/
        │   ├── terraform/
        │   └── cloudformation/
        ├── ecs/
        ├── config/
        └── README.md
```

> Every new application added to this repository follows this exact same structure.
> Navigate to one `systems/` folder — everything needed to deploy that application is there.

---

## Kustomize: Base/Overlays Pattern

Kubernetes manifests are written **once** in `base/` and **patched per platform** in `overlays/`. No YAML is duplicated.

```
k8s/base/                     ← platform-agnostic (~80% of all manifest content)
    deployment.yaml               identical across local, EKS, AKS
    configmap.yaml                same structure; only datasource URL changes per overlay
    mysql-pvc.yaml                same spec; StorageClass is patched in each overlay

k8s/overlays/local/           ← patches for minikube / kind
    service type   →  NodePort
    StorageClass   →  standard
    ingress class  →  nginx

k8s/overlays/eks/             ← patches for AWS EKS
    service type   →  LoadBalancer (AWS NLB)
    StorageClass   →  gp3 (EBS CSI driver)
    ingress class  →  aws-load-balancer-controller
    datasource URL →  RDS endpoint (from Terraform output)

k8s/overlays/aks/             ← patches for Azure AKS (planned)
    service type   →  LoadBalancer (Azure LB)
    StorageClass   →  managed-premium (Azure Disk)
    ingress class  →  azure/application-gateway
```

**Why manifests change per platform:**

The `SPRING_DATASOURCE_URL` alone illustrates why overlays are necessary. On a local cluster it points to `mysql-service:3306` (Kubernetes internal DNS). On EKS it points to an RDS endpoint like `your-db.us-east-1.rds.amazonaws.com:3306`. The `Service` type, `StorageClass`, and `Ingress` annotations also differ per cloud provider. `base/` holds what never changes; `overlays/` holds only what does.

**Deploy to a target:**

```bash
# Local cluster (minikube / kind)
kubectl apply -k systems/java-monolith/k8s/overlays/local/

# AWS EKS
kubectl apply -k systems/java-monolith/k8s/overlays/eks/

# Azure AKS
kubectl apply -k systems/java-monolith/k8s/overlays/aks/
```

---

## Terraform: Modules vs Environments

Inside each system's `infra/terraform/` there are two layers:

```
modules/        ← reusable logic (what to build and how)
    eks-cluster/    main.tf, variables.tf, outputs.tf
    ec2-asg/        main.tf, variables.tf, outputs.tf
    ecs-cluster/    main.tf, variables.tf, outputs.tf

envs/           ← real values per deployment target (where and with what config)
    eks/
        terraform.tfvars    ← cluster name, region, node type for THIS application
    ec2-asg/
        terraform.tfvars    ← ASG name, instance type, min/max for THIS application
    ecs/
        terraform.tfvars    ← ECS cluster name and task sizing for THIS application
```

Module code is structurally identical across systems. Only `terraform.tfvars` differs — this is where `java-monolith` gets its own cluster name, tags, and node sizing, entirely separate from `python-monolith`.

```bash
# Provision EKS cluster for java-monolith
cd systems/java-monolith/infra/terraform/envs/eks
terraform init
terraform plan
terraform apply
```

---

## Deployment Targets & Methods

| Target | Terraform | CloudFormation | Ansible | kubectl (Kustomize) | Helm | ArgoCD |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **EC2 + Auto Scaling Group** | ✅ | ✅ | ✅ | — | — | — |
| **ECS (Fargate / EC2)** | ✅ | ✅ | — | — | — | — |
| **Local Kubernetes** (minikube/kind) | — | — | — | ✅ | ✅ | — |
| **AWS EKS** | ✅ | — | — | ✅ | ✅ | ✅ |
| **Azure AKS** (planned) | ✅ | — | — | ✅ | ✅ | ✅ |

---

## CI → CD Hand-Off

```
CI  (Application Repo — Jenkins / GitHub Actions)
────────────────────────────────────────────────────
  1. Code pushed to java-monolith-app
  2. Build JAR → Docker image
  3. Run tests (JUnit, SonarQube)
  4. Trivy image scan
  5. Push image → DockerHub / ECR / Nexus
  6. Write new image tag to systems/java-monolith/image.env
                │
                │  git commit: "ci: update java-monolith image to sha-abc123"
                ▼
CD  (This Repo — platform-engineering-systems)
────────────────────────────────────────────────────
  EKS  →  ArgoCD detects image.env change → syncs overlays/eks/ → rolling update
  EC2  →  Ansible reads new image tag → pulls image → restarts via systemd
  ECS  →  Terraform updates task definition with new image tag → ECS rolling deploy
```

`image.env` is the single source of truth for which image version is deployed where. The CI pipeline never touches deployment logic — it only writes a tag and commits.

---

## GitOps Flow

```
Developer pushes code to application repo
              │
              ▼
   ┌──────────────────────────────┐
   │     CI — Application Repo   │
   │  Jenkins / GitHub Actions   │
   │  ────────────────────────── │
   │  1. Build artifact + image  │
   │  2. Test + scan             │
   │  3. Push image to registry  │
   │  4. Update image.env ───────┼──────────────────────────┐
   └─────────────────────────────┘                          │
                                                            ▼
                                          ┌─────────────────────────────────────┐
                                          │   platform-engineering-systems       │
                                          │   ─────────────────────────────────  │
                                          │   EKS  → ArgoCD syncs manifests     │
                                          │   EC2  → Ansible re-deploys         │
                                          │   ECS  → Terraform updates task     │
                                          └─────────────────┬───────────────────┘
                                                            │
                                                            ▼
                                                ┌───────────────────────┐
                                                │   Target Environment  │
                                                │   Application live    │
                                                └───────────────────────┘
```

---

## Current Systems

### [java-monolith](./systems/java-monolith/) — Active

| Property | Detail |
|---|---|
| **Application** | Spring Boot REST API — hospital / banking management system |
| **Language** | Java 17 + Maven |
| **Database** | MySQL 8 |
| **Source Repo** | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) (Git submodule) |
| **CI Pipeline** | Jenkins (DevSecOps) + GitHub Actions (mirrored) |
| **Deployment Targets** | Local K8s · AWS EKS · EC2 ASG · AWS ECS |
| **K8s Manifests** | Kustomize base/overlays (local, eks, aks) |
| **IaC** | Terraform · CloudFormation |
| **Config Management** | Ansible |
| **GitOps** | ArgoCD (EKS target) |

### python-monolith — Planned

Same structure as `java-monolith`. Will be added when the Python application source repo is ready.

---

## Getting Started

### Prerequisites

```bash
git --version         # Git (submodule support)
docker --version      # Docker
kubectl version       # kubectl
kustomize version     # Kustomize (or use kubectl -k)
helm version          # Helm >= 3
terraform --version   # Terraform >= 1.5
aws --version         # AWS CLI v2 (for EKS / ECS targets)
ansible --version     # Ansible (for EC2 targets)
```

### Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/ibtisam-iq/platform-engineering-systems.git
cd platform-engineering-systems
```

If already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Deploy java-monolith

```bash
# Local Kubernetes (minikube / kind)
kubectl apply -k systems/java-monolith/k8s/overlays/local/

# AWS EKS (after Terraform provisions the cluster)
cd systems/java-monolith/infra/terraform/envs/eks
terraform init && terraform apply
kubectl apply -k systems/java-monolith/k8s/overlays/eks/

# EC2 via Ansible
cd systems/java-monolith/config
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml
```

---

## Engineering Principles

- **App-scoped everything** — each system folder owns its own infra, manifests, and config; zero cross-system dependencies
- **GitOps-first** — ArgoCD watches this repo; `image.env` is the immutable CI→CD hand-off contract
- **Kustomize base/overlays** — Kubernetes manifests written once, patched per platform; no YAML duplication
- **Terraform modules + envs** — reusable module logic, app-specific values isolated in `terraform.tfvars`
- **Method isolation** — each deployment method (Terraform, Ansible, kubectl, Helm) lives in its own subfolder and is independently executable
- **Platform-agnostic by design** — new platforms (Azure, GCP, bare metal) added via new overlays without modifying existing ones
- **Reproducible** — every deployment can be torn down and rebuilt from scratch using only the code in this repository

---

## Related Repositories

| Repository | Role |
|---|---|
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | This repo — CD + Infrastructure |
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Spring Boot source + CI pipelines (submodule) |

---

## Author

**Muhammad Ibtisam Iqbal**  
DevOps Engineer · Platform Engineering · Cloud Infrastructure  
[GitHub](https://github.com/ibtisam-iq) · [Website](https://ibtisam-iq.com)

# Platform Engineering Systems

> **Transforming CI artifacts into highly reliable, scalable, and observable production systems.**

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECS%20%7C%20EKS-FF9900?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Configuration-EE0000?logo=ansible&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Package%20Manager-0F1689?logo=helm&logoColor=white)
![Kustomize](https://img.shields.io/badge/Kustomize-Manifests-326CE5?logo=kubernetes&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?logo=prometheus&logoColor=white)
![Elastic](https://img.shields.io/badge/Elastic-Logging-005571?logo=elastic&logoColor=white)

This repository is **not** an application codebase. It is a hands-on portfolio showcasing my journey into **Platform Engineering and Continuous Deployment (CD)**. 

The Continuous Integration (CI) process—building code, running tests, and compiling artifacts—is intentionally isolated in separate application repositories. Once those external CI pipelines produce a deployable artifact (a Docker image, a JAR file, etc.), *this* repository takes over to provision the infrastructure, configure the orchestration, and execute the deployment across real-world environments.

## The Two-Repo Architecture (CI vs. CD)

```text
┌──────────────────────────────────────┐       ┌───────────────────────────────────────────┐
│       Application Repositories       │       │       platform-engineering-systems        │
│       (e.g., java-monolith-app)      │       │                (THIS REPO)                │
│                                      │       │                                           │
│   Continuous Integration (CI)        │       │   Continuous Deployment (CD) / Platform   │
│   ──────────────────────────────     │       │   ─────────────────────────────────────   │
│   • Compile source code              │──────▶│   • Provision infrastructure (Terraform)  │
│   • Run automated tests              │       │   • Orchestrate (Kustomize, Helm, Ansible)│
│   • Build & scan container images    │       │   • Deploy to EC2, ECS, or Kubernetes     │
│   • Push Artifact to Registry        │       │   • GitOps sync (ArgoCD watches this repo)│
└──────────────────────────────────────┘       └───────────────────────────────────────────┘
```

---

## 🚀 The Architectural Progression

This portfolio documents a deliberate progression in application architecture. I curated 10-12 different repositories to document my practical learning journey through software architecture evolution:

1. **2-Tier Architecture:** I started by deploying a foundational frontend + database architecture (documented inside `node-monolith`).
2. **3-Tier Architecture:** I manually upgraded that codebase and infrastructure to a secure 3-tier architecture (adding a dedicated backend API layer).
3. **Microservices:** With a firm grasp on decoupled tiers, I progressed to orchestrating complex, polyglot microservices on EKS (`microservices-demo` and `retail-store-sample-app`).

---

## 🏗 The Portfolio: Curated Projects ("Systems")

A "System" in this repository represents an independent, real-world project. I have isolated each project into its own folder. 

| System (Project) | Architecture & Focus | Highlights |
|------------------|----------------------|------------|
| **[`microservices-demo`](./systems/microservices-demo/)** | **Platform Engineering & GitOps**<br>A massively scalable polyglot microservices stack deployed onto Amazon EKS. | <ul><li>**GitOps:** ArgoCD + Image Updater</li><li>**Networking:** Gateway API + External-DNS</li><li>**Observability:** EFK Stack + Kube-Prometheus</li></ul> |
| **[`java-monolith`](./systems/java-monolith/)** | **Deployment Evolution**<br>Deploys a single Spring Boot artifact across four increasingly complex environments, building practical cloud experience. | <ul><li>**Containerless:** EC2 ASG + ALB</li><li>**Serverless:** ECS Fargate</li><li>**Bare-Metal:** Kubeadm + NAT networking</li><li>**Production:** EKS + Terraform</li></ul> |
| **`python-monolith` & `node-monolith`** | **Language-Agnostic Platforming**<br>Exact structural mirrors of the `java-monolith` system, demonstrating that the identical 4-stage deployment architecture applies seamlessly across Python and Node.js. *The Node project specifically highlights the 2-tier ➔ 3-tier upgrade.* | <ul><li>**IaC:** Reusable Terraform Modules</li><li>**Manifests:** Kustomize Base/Overlays</li></ul> |
| **[`static-website`](./systems/static-website/)** | **Global Distribution & Security**<br>A highly secure static asset deployment on AWS without underlying compute resources. | <ul><li>**Delivery:** S3 + CloudFront (OAC)</li><li>**Security:** KMS Encryption + IAM</li><li>**DR:** Cross-Region Replication</li></ul> |

---

## 🧠 Engineering Disciplines (The Portfolio Philosophy)

Because this repository serves as a professional portfolio, I intentionally avoid tool redundancy. I designed these systems to build and showcase my competency across the core **pillars of DevOps** (IaC, Orchestration, CI/CD, and Observability) by implementing a diverse, rotating technology stack. For a detailed breakdown, read the global **[Architectural Principles Document](./docs/architecture.md)**.

1. **Intentional Technology Diversity:** Instead of using the exact same stack everywhere, I rotate tools to show adaptability. If I use **ArgoCD** for GitOps in one system, I might use **Helm** or **Kustomize** natively in another. If I use the **ELK Stack** for logging in one, I will use **CloudWatch** or **Prometheus/Grafana** in another.
2. **Core Concepts Over Tool Lock-in:** The goal is to demonstrate an understanding of the underlying architecture—whether provisioning a cluster via **Terraform**, **eksctl**, or bare-metal **kubeadm**.
3. **Strict System Isolation:** Each application owns its own infrastructure code and manifests. Because the systems are completely decoupled, I can freely experiment with different orchestration and monitoring paradigms without cross-system contamination.
4. **Production-Grade Rigor:** Regardless of the tool chosen for a specific system, every implementation strives to adhere to strict production standards: immutable infrastructure, automated deployments, and deep observability.

---

## 🔗 External Application Repositories (The CI Side)

The Continuous Integration pipelines and source code for the artifacts deployed in this repository can be found here:
- **Microservices:** [microservices-demo](https://github.com/ibtisam-iq/microservices-demo) \| [retail-store-sample-app](https://github.com/ibtisam-iq/retail-store-sample-app)
- **Monoliths:** [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) \| [python-monolith-app](https://github.com/ibtisam-iq/python-monolith-app)
- **Architecture Progression:** [node-monolith-2tier-app](https://github.com/ibtisam-iq/node-monolith-2tier-app) \| [node-monolith-3tier-app](https://github.com/ibtisam-iq/node-monolith-3tier-app)

---

## 📚 The Knowledge Bases

While building these environments, I extensively documented the edge-cases, networking hurdles, and architectural pivots I encountered. 

**Note:** Because each system is strictly isolated, their deep-dive runbooks are stored locally within their own directories. For example, if you want to understand how I solved HTTP-01 ACME challenges for a cluster trapped behind a NAT without a public IP, explore the Java Monolith's specific Knowledge Base:

👉 **[View the `java-monolith` Knowledge Base](./systems/java-monolith/docs/)**

---

## 👨‍💻 Author

**Muhammad Ibtisam Iqbal**  
*DevOps Engineer · Platform Engineering · Cloud Infrastructure*  
[GitHub](https://github.com/ibtisam-iq) · [Website](https://ibtisam-iq.com)

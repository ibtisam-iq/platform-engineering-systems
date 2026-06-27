# Platform Engineering Systems — Architectural Principles

## The Core Problem

How do I take a raw, compiled application artifact and transform it into a reliable, scalable, and observable system running in a production-grade environment? 

This repository is my answer to that question. It demonstrates how to build the crucial bridge between the end of a CI pipeline and the reality of a production runtime.

## The Architectural Evolution: 2-Tier ➔ 3-Tier ➔ Microservices

I believe an effective DevOps engineer must understand the software they are hosting. I didn't just deploy random code—I curated over 10 different repositories to explore a deliberate evolution in application architecture:

1. **2-Tier Architecture:** I began with a foundational 2-tier setup (a Node.js frontend interacting directly with a database).
2. **3-Tier Architecture:** To practice secure tiering, I manually upgraded the application code and the infrastructure to a 3-tier architecture, isolating the database behind a dedicated backend API layer (documented within `node-monolith`).
3. **Polyglot Microservices:** After establishing the core paradigms of decoupled monoliths, I progressed to orchestrating highly distributed, polyglot microservices (`microservices-demo`). 

This progression proves that I understand how to host an application not just as a black box, but as an integrated architectural component.

---

## Separation of Concerns: CI vs. CD (The Artifact Handoff)

To build a truly resilient ecosystem, I intentionally divided my architecture into two strictly decoupled layers:

### 1. The Continuous Integration (CI) Layer (External)
The CI pipelines live entirely outside of this repository in dedicated application repositories (e.g., `node-monolith-3tier-app`, `java-monolith-app`). They are strictly responsible for **building and testing**. They pull source code, run tests, compile the application, and produce an immutable **Artifact** (a Docker image, a JAR file, or a static bundle) which is pushed to a registry.

### 2. The Continuous Deployment (CD) Layer (This Repository)
This repository represents the **Platform Layer**. It takes over the moment the CI pipeline finishes. It never compiles source code. Instead, it provisions the infrastructure (Terraform), configures the orchestrator (Kubernetes/ECS), and establishes the observability stack to deploy the artifact.

---

## System Anatomy

In this repository, a "system" represents an independent **Project**, acting as a complete runtime unit. I design every system to encompass:

- **Container/Artifact Layer:** Standardized runtime packaging (e.g., pulling a specific Docker image tag from a registry).
- **Orchestration Layer:** Kubernetes manifests, Ingress/Gateway API configurations, and Horizontal Pod Autoscaling.
- **Infrastructure Layer:** Immutable, Terraform-provisioned resources (VPCs, EKS clusters, RDS instances).
- **Observability Layer:** Metrics (Prometheus), logs (ElasticSearch/Loki), and dashboards (Grafana) injected directly into the deployment.

---

## Key Design Decisions

### 1. Intentional Technology Diversity
This repository serves as a portfolio. Therefore, I intentionally avoid tool lock-in. I rotate technologies (ArgoCD vs. Helm, Prometheus vs. CloudWatch, Terraform vs. kubeadm) across different systems to showcase my adaptability and broad competency across all pillars of DevOps.

### 2. Strict System Isolation
Every system (e.g., `java-monolith`, `microservices-demo`) is completely independent. There is no shared, centralized deployment configuration. Because the systems are completely decoupled, I can freely experiment with different orchestration and monitoring paradigms without cross-system contamination.

### 3. Artifact-Driven Model
These platform systems consume pre-built artifacts. By preventing the platform layer from building source code, I ensure that the exact same artifact tested in the CI pipeline is the one running in production, eliminating "it works on my machine" anomalies.

### 4. Language-Agnostic Platforming
To prove that my architectural designs are robust, I explicitly mirror them across different language stacks. The exact same 4-phase deployment evolution applied to the Java application is replicated for Python and Node.js, proving the platform layer is entirely language-agnostic.

---

## Implemented Systems

To prove this architecture, I have implemented the following systems, each demonstrating a different level of infrastructure complexity:

### 1. `static-website`
**Focus:** Global Distribution & Security  
**Architecture:** S3 Origin + CloudFront (OAC) + KMS Encryption + Cross-Region Replication + CloudTrail. Demonstrates highly secure, distributed static asset hosting without the overhead of compute resources.

### 2. `java-monolith`
**Focus:** Deployment Evolution & Kubernetes  
**Architecture:** I deployed a single Java artifact across four increasingly complex environments: EC2 Auto Scaling (Containerless) ➔ ECS Fargate ➔ Bare-Metal Kubernetes (tackling NAT/Gateway API challenges) ➔ Amazon EKS (provisioned via Terraform). 

### 3. `python-monolith` & `node-monolith`
**Focus:** Proving Language-Agnostic Deployments & App Evolution  
**Architecture:** These systems are exact structural mirrors of the `java-monolith`. They implement the identical four environments to prove that the platform architecture handles Python and Node.js artifacts with the same production-grade rigor. *The `node-monolith` specifically houses the 2-tier ➔ 3-tier architectural upgrade.*

### 4. `microservices-demo`
**Focus:** Platform Engineering & GitOps  
**Architecture:** A massive polyglot microservices stack deployed onto EKS. Focuses heavily on the control plane: ArgoCD (for true GitOps), Gateway API, External-DNS, EFK (for centralized logging), and kube-prometheus (for deep metrics).

---

## Conclusion

This repository demonstrates my hands-on experience and foundational understanding of platform engineering, cloud-native architecture, and production-grade operations. It focuses on one overarching goal: **Taking CI artifacts and turning them into reliable, observable, production-ready systems.**

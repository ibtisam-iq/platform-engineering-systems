# Polyglot Microservices Platform — End-to-End GitOps on EKS

This directory contains the **Platform Engineering** and **Infrastructure** layer for a highly scalable polyglot microservices application. 

While the application source code (and its CI pipelines) are isolated in their own repository (linked here as the `app/` git submodule), *this* directory contains the complete Platform Engineering implementation, focusing heavily on **GitOps**, **Observability**, and **Cluster Add-ons** (Phases 3 through 6 of my deployment journey).

For in-depth runbooks and architectural decisions regarding this build, refer to my official documentation: **[End-to-End DevOps: CI/CD, GitOps, and Observability on Amazon EKS](https://runbook.ibtisam-iq.com/projects/deployments/microservices-demo/)**.

---

## Directory Structure & Platform Capabilities

This repository defines the infrastructure and the platform stack deployed onto Amazon EKS.

### 1. [Infrastructure as Code (Terraform)](./terraform/)
Contains the Terraform modules used to provision the underlying EKS clusters and VPC networking. Includes configurations for both real AWS accounts (`eks-aws`) and KodeKloud sandboxes (`eks-kodekloud`).

### 2. [Platform Add-ons (The Control Plane)](./addons/)
This is the core of the platform engineering implementation. Instead of manually applying manifests, these add-ons establish a fully automated, observable, and GitOps-driven cluster:

- **[`argocd/`](./addons/argocd/)**: The GitOps engine. Continuously reconciles the cluster state against this repository.
- **[`image-updater/`](./addons/image-updater/)**: ArgoCD Image Updater. Automates continuous deployment by monitoring ECR for new application image tags and updating the cluster without human intervention.
- **[`gateway-api/`](./addons/gateway-api/)**: Modern Kubernetes networking. Replaces legacy Ingress with the Gateway API for advanced traffic routing.
- **[`external-dns/`](./addons/external-dns/)**: Automatically synchronizes Kubernetes services and gateways with AWS Route53.
- **[`kube-prometheus/`](./addons/kube-prometheus/)**: The full kube-prometheus-stack (Prometheus, Grafana, Alertmanager) for deep infrastructure and application metrics.
- **[`elastic-logging/`](./addons/elastic-logging/)**: The EFK (Elasticsearch, Fluentd/Filebeat, Kibana) stack for centralized log aggregation across all microservices.

### 3. Application Deployment via GitOps
- **[`app/`](./app/)**: A Git Submodule pointing to the application source code and CI pipelines.
- **[`application.yaml`](./application.yaml) & [`kustomization.yaml`](./kustomization.yaml)**: The ArgoCD root Application manifests that bootstrap the cluster and deploy the microservices workloads (via the `chart/` and `manifests/` directories) entirely through GitOps.

---

## The Platform Engineering Workflow

By separating the application code from the platform code, we achieve true Platform Engineering GitOps:

1. **Continuous Integration (App Repo):** Developers push code to the `app/` repo. GitHub Actions / Cloud Build runs tests and pushes a new container image to Amazon ECR.
2. **Continuous Deployment (This Repo):** The ArgoCD Image Updater detects the new ECR image tag and automatically patches the cluster state to deploy the new version.
3. **Observability:** As the new microservices roll out, metrics are instantly scraped by Prometheus and logs are aggregated into ElasticSearch, allowing for immediate feedback on the health of the deployment.

> *Note: For details on the Phase 1 CI pipelines, please navigate into the `app/` submodule. This repository focuses strictly on the Phase 2+ Platform Infrastructure deployment.*

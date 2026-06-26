# BankApp Kubernetes Deployment

This directory contains the Infrastructure-as-Code (IaC) for deploying the Java Monolith (BankApp) to Kubernetes. I chose to use **Kustomize** to manage configurations across multiple environments, ensuring a DRY (Don't Repeat Yourself) approach without duplicating YAML manifests.

## Key Architectural Decisions

Throughout this implementation, I took several architectural decisions to ensure the deployment is scalable, maintainable, and production-ready:

1. **Platform-Agnostic Design:** To demonstrate flexibility and expertise across diverse environments, I intentionally designed this deployment to be platform-agnostic. The project can be seamlessly deployed across different infrastructure providers, specifically Bare-Metal/Local Clusters (Minikube, Kind) and Managed Cloud Kubernetes (AWS EKS).
2. **Gateway API vs. Ingress:** This repository includes networking configurations for both traditional Ingress and the modern Gateway API. While both methods are available for reference, the active routing configuration defaults to the Gateway API, as it is the officially promoted standard for the future of Kubernetes networking.
3. **Storage Class Decoupling:** I intentionally omitted the `storageClassName` from the base MySQL PVC (`mysql-pvc.yaml`). Rather than polluting the base configuration with environment-specific storage drivers, I chose to patch the PVC in the respective overlays (`local-path` for bare-metal, `gp3` for EKS).
4. **Centralized Kustomize Patching:** Instead of editing individual resource files for environment-specific dynamic values (such as RDS endpoints or ACM Certificate ARNs), I moved all patches to `kustomization.yaml`. This ensures all modifications are handled centrally and individual manifest files remain untouched.
5. **Secret Management:** The `secret.yaml` in the base directory currently contains placeholder base64 values. For real production deployments, I use **HashiCorp Vault** (often paired with External Secrets Operator) rather than committing actual credentials to Git.
6. **Strategic HPA Placement:** I intentionally kept the `HorizontalPodAutoscaler` (HPA) out of the `base` and `bare-metal` overlay. It is exclusively deployed in the `eks` overlay because autoscaling relies on a Metrics Server, which is expected in production EKS clusters but often unnecessary in local development environments. This prevents local errors and keeps the bare-metal environment lightweight.
7. **Decoupled HTTPRoutes for Gateway Features:** I excluded `httproute.yaml` from the `base` and wrote separate routes for each overlay. The routing logic inherently differs between platforms: in bare-metal (using Nginx), explicit HTTP-to-HTTPS redirection must be handled within the HTTPRoute itself. In EKS, the AWS Application Load Balancer natively handles TLS termination and redirection at the Gateway level. Maintaining separate HTTPRoutes allows me to leverage these platform-specific features without conflict.

## Directory Structure

```text
k8s/
├── base/                   # Core application manifests (environment-agnostic)
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── mysql-pvc.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   └── kustomization.yaml
└── overlays/               # Environment-specific patches and additions
    ├── bare-metal/         # For local development (minikube, kind, local bare-metal)
    └── eks/                # For AWS Elastic Kubernetes Service deployments
```

## Environments

### Base
The `base` directory contains the foundational resources that are common to all environments. This includes the `bankapp` Namespace, the Spring Boot application Deployment/Service, and the MySQL database Stateful workloads.

### Bare-Metal Overlay (`overlays/bare-metal`)
Tailored for local clusters (e.g., Minikube, Kind) or bare-metal servers.
- **Storage:** Patches the MySQL PVC to use the `local-path` provisioner.
- **Routing:** Uses the `nginx` Gateway API (`nginx-gateway-fabric`) for ingress traffic routing.
- **Certificates:** Integrates `cert-manager` to automatically provision Let's Encrypt TLS certificates via the HTTP-01 challenge.

### EKS Overlay (`overlays/eks`)
Tailored for AWS Elastic Kubernetes Service (EKS) infrastructure.
- **Storage:** Configures the AWS EBS `gp3` StorageClass for high-performance, cost-effective database persistence.
- **Routing:** Utilizes the **AWS Load Balancer Controller** to provision native Application Load Balancers (ALB) via the Gateway API.
- **Scaling:** Includes a `HorizontalPodAutoscaler` (HPA) to automatically scale the application between 2 and 5 replicas based on CPU and Memory utilization.
- **Certificates:** Relies on AWS Certificate Manager (ACM) for TLS termination at the ALB level.

## Deployment Instructions

Apply the specific Kustomize overlay for the target environment.

### Deploying to Bare-Metal (Local)
1. Ensure the local cluster has a `local-path` provisioner and the `nginx-gateway-fabric` installed. Reference the [cluster bootstrap runbook](https://runbook.ibtisam-iq.com/bootstrap/) to install all necessary cluster addons.
2. Run:
   ```bash
   kubectl apply -k k8s/overlays/bare-metal
   ```

### Deploying to EKS (Production)
1. Ensure the EKS cluster has the **AWS Load Balancer Controller**, **Metrics Server** (for the HPA), and **EBS CSI Driver** installed. Reference the [cluster bootstrap runbook](https://runbook.ibtisam-iq.com/bootstrap/) for provisioning these EKS-specific addons.
2. Open `k8s/overlays/eks/kustomization.yaml` and update the database connection string and `<ACM_CERTIFICATE_ARN>` placeholders. Thanks to the centralized Kustomize architecture, individual resource files never need editing—all dynamic values and storage classes are injected via patches directly in the `kustomization.yaml`.
3. Run:
   ```bash
   kubectl apply -k k8s/overlays/eks
   ```

## Checking Deployment Status
Verify that all components are properly running in the `bankapp` namespace using the following commands:
```bash
kubectl get all -n bankapp
kubectl get gateways -n bankapp
kubectl get httproute -n bankapp
```

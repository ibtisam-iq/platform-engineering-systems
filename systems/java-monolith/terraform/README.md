# Infrastructure Provisioning for Java Monolith

This directory contains the Terraform scripts I used to provision the underlying Kubernetes infrastructure for the Java Monolith project.

I chose to deploy this project on the **AWS KodeKloud Playground** to test my setup in a constrained environment. Because the KodeKloud lab enforces strict AWS Organizations SCP (Service Control Policies), standard deployment methods (like the official EKS Terraform module or managed node groups) silently fail. Therefore, I wrote this custom Terraform configuration specifically to work around those restrictions and successfully build the EKS cluster.

## 📖 Deployment Instructions & Runbook

This Terraform code is part of a broader deployment process. To successfully provision the EKS cluster within the KodeKloud constraints, attach worker nodes, and configure the necessary addons, I strictly followed my companion runbook.

To avoid duplicating complex steps and troubleshooting notes, **all deployment commands and workarounds are fully documented in the runbook:**

👉 **[EKS on KodeKloud (Terraform) Runbook](https://runbook.ibtisam-iq.com/iac/terraform/provisioning/eks-on-kodekloud-terraform/)**

---

## What This Provisions

| Resource | Detail |
|---|---|
| VPC | 3 AZs, public + private subnets, single NAT gateway |
| Bastion host | Ubuntu 26.04, public subnet, SSH-locked to operator IP |
| EKS cluster | v1.36, private API endpoint, `API_AND_CONFIG_MAP` auth |
| EKS addons | `vpc-cni`, `kube-proxy`, `eks-pod-identity-agent` (no CoreDNS) |
| OIDC provider | For IRSA (IAM Roles for Service Accounts) |
| IAM roles | `eksClusterRole`, `eksNodeRole` with exact SCP-whitelisted names |
| Worker nodes | **Not provisioned by Terraform** — deployed via CloudFormation (see runbook) |

## Related Documentation

- [EKS challenges and fixes log](https://runbook.ibtisam-iq.com/iac/terraform/provisioning/eks-on-kodekloud-terraform-challenges/)
- [EKS on KodeKloud via eksctl (alternative manual approach)](https://runbook.ibtisam-iq.com/iac/terraform/provisioning/eks-on-kodekloud-eksctl/)
- [Deploy AWS Load Balancer Controller](https://runbook.ibtisam-iq.com/bootstrap/kubernetes/addons-eks/deploy-aws-load-balancer-controller/)
- [Install EBS CSI Driver](https://runbook.ibtisam-iq.com/bootstrap/kubernetes/addons-eks/install-ebs-csi-driver/)
# BankApp Knowledge Base

This directory contains technical deep-dives, architectural notes, and runbooks I authored while implementing the infrastructure for this project. They document the various environments, edge cases, and technologies I explored (such as transitioning from Ingress to the Gateway API, and handling TLS in NAT environments vs public IPs).

## [Networking](./networking/)

- **[`kubernetes-networking-guide.md`](./networking/kubernetes-networking-guide.md)**: A comprehensive guide covering the transition from the Kubernetes Ingress API to the modern Gateway API. It details HTTP host-based routing, ClusterIP vs NodePort vs LoadBalancer, and specific implementations for Bare-Metal vs Cloud environments.
- **[`kubernetes-ingress-guide.md`](./networking/kubernetes-ingress-guide.md)**: A practical deployment guide specifically focused on the legacy `Ingress` object, detailing how to expose HTTP/HTTPS routes and test them with local `/etc/hosts` DNS overrides.

## [Cert-Manager](./cert-manager/)

- **[`cert-manager-overview.md`](./cert-manager/cert-manager-overview.md)**: An overview of the cert-manager architecture, its internal components (webhook, cainjector, controller), and the ACME Certificate Request lifecycle.
- **[`cert-manager-ingress-http01.md`](./cert-manager/cert-manager-ingress-http01.md)**: A specific implementation guide for automating TLS certificates using the HTTP-01 challenge via Ingress on a Bare-Metal EC2 cluster that possesses a public IP address.
- **[`cert-manager-gateway-troubleshooting.md`](./cert-manager/cert-manager-gateway-troubleshooting.md)**: A troubleshooting runbook dedicated to resolving stuck HTTP-01 challenges when using cert-manager with the NGINX Gateway Fabric (Gateway API).

## [TLS](./tls/)

- **[`tls-complete-guide.md`](./tls/tls-complete-guide.md)**: A massive 900+ line guide documenting the unique challenge of acquiring real TLS certificates on a Bare-Metal kubeadm cluster running behind a NAT without a public IP. It covers DNS-01 challenges, Cloudflare Tunnels, and Self-Signed configurations.

## [Git](./git/)

- **[`git-submodule.md`](./git/git-submodule.md)**: A quick reference for managing the application source code Git Submodule inside this infrastructure monorepo.

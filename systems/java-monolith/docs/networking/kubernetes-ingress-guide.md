# Kubernetes Ingress — Practical Deployment Guide

The Kubernetes `Ingress` object exposes HTTP/HTTPS routes from outside the cluster to services inside it.
It provides host-based and path-based routing, TLS termination, and header rewriting — all defined
declaratively. Unlike a `Service` of type `NodePort` or `LoadBalancer`, Ingress consolidates multiple
routing rules under a single entry point.

> **Deprecation notice**
> The Ingress API (`networking.k8s.io/v1`) is now in maintenance mode. The Kubernetes community has
> moved to the [Gateway API](https://gateway-api.sigs.k8s.io/) as the long-term replacement. Ingress
> remains fully functional and widely used, but new projects should evaluate Gateway API first.

Official reference: https://kubernetes.io/docs/concepts/services-networking/ingress/

---

## Prerequisites

### 1. Install the Ingress Controller

An Ingress object does nothing on its own. A controller must be running in the cluster to satisfy it.
Multiple controllers are available (Nginx, Traefik, HAProxy, Contour, etc.). This runbook uses **ingress-nginx**.

> **Important**
> Use the `baremetal` provider variant for kubeadm clusters. The `cloud` and `aws` variants provision
> a `LoadBalancer` service, which requires a cloud provider or MetalLB on bare-metal.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
```

Full installation guide: https://runbook.ibtisam-iq.com/bootstrap/kubernetes/deploy-ingress-nginx-controller/

Verify the controller is running:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

> **Note**
> On bare-metal, the `ingress-nginx-controller` service is of type `NodePort`. Note the assigned port —
> it is the external entry point for all Ingress traffic on this cluster.
>
> ```bash
> kubectl get svc ingress-nginx-controller -n ingress-nginx \
>   -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'
> ```

---

## The Ingress Object

### Multiple IngressClass options

Multiple Ingress controllers can coexist in the same cluster, each identified by an `IngressClass`.
Specify which controller handles a given Ingress via `spec.ingressClassName`. The `kubernetes.io/ingress.class`
annotation is kept for backwards compatibility with older controllers.

This deployment uses `nginx` as the IngressClass.

---

## Variant 1 — HTTP (No TLS)

Deploy this variant in local or bare-metal environments without a public domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bankapp-ingress
  namespace: bankapp
  labels:
    app.kubernetes.io/name: bankapp
    app.kubernetes.io/part-of: bankapp
    app.kubernetes.io/managed-by: kustomize
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
    - host: java-monolith.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bankapp-service
                port:
                  number: 80
```

---

## Variant 2 — HTTPS with Automated TLS (cert-manager + Let's Encrypt)

Deploy this variant on a bare-metal server with a public IP and a real DNS record.

> **Prerequisites for TLS**
> - DNS A record: `<hostname>` → `<node-public-ip>` (Cloudflare proxy **must be disabled** — grey cloud — for HTTP-01 challenge)
> - Ports 80 and 443 open on the host firewall
> - cert-manager installed:
>   ```bash
>   helm upgrade --install cert-manager jetstack/cert-manager \
>     --namespace cert-manager \
>     --set crds.enabled=true \
>     --set config.enableGatewayAPI=true
>   kubectl get pods -n cert-manager
>   ```

```yaml
***
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: contact@ibtisam-iq.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx

***
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bankapp-ingress-cert
  namespace: bankapp
  labels:
    app.kubernetes.io/name: bankapp
    app.kubernetes.io/part-of: bankapp
    app.kubernetes.io/managed-by: kustomize
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
    - host: java-monolith.ibtisam-iq.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bankapp-service
                port:
                  number: 80
  tls:
    - hosts:
        - java-monolith.ibtisam-iq.com
      secretName: java-monolith-tls
```

> **Certificate lifecycle**
> cert-manager watches any Ingress annotated with `cert-manager.io/cluster-issuer`. It creates a
> `Certificate` resource, initiates an ACME HTTP-01 challenge with Let's Encrypt, and stores the
> issued certificate in the `secretName` Secret. Renewal happens automatically 30 days before expiry
> (certificates are valid for 90 days). Do not create the TLS Secret manually.

Watch certificate issuance:

```bash
kubectl apply -f ingress-cert.yaml
kubectl get certificate -n bankapp -w
kubectl describe certificate java-monolith-tls -n bankapp
kubectl get secret java-monolith-tls -n bankapp
```

---

## Accessing the Application

### Step 1 — Identify the NodePort

On bare-metal, ingress-nginx exposes a `NodePort`. All external traffic enters through this port.

```bash
# HTTP NodePort
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'

# HTTPS NodePort
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
```

### Step 2 — Verify the Ingress object

```bash
kubectl get ingress -n bankapp
kubectl describe ingress bankapp-ingress -n bankapp
```

> **Expected result**
> The `ADDRESS` field should show the node IP. If it is empty, the controller is not processing
> the Ingress — verify `ingressClassName` matches the installed controller.

---

## Access Methods

### Method 1 — Direct curl with explicit Host header (no DNS required)

```bash
# HTTP
curl -I -H "Host: <ingress-host>" http://<node-ip>:<http-nodeport>/

# Example (values from this deployment)
curl -I -H "Host: java-monolith.local" http://172.16.0.2:31844/
```

> **Why the Host header is required**
> ingress-nginx routes based on the `Host` header in the HTTP request. Without it, the controller
> returns 404 because no rule matches the empty host.

### Method 2 — /etc/hosts entry (local machine only)

Add a static hostname mapping on the machine running the curl command:

```bash
echo "<node-ip>  <ingress-host>" | sudo tee -a /etc/hosts

# Example
echo "172.16.0.2  java-monolith.local" | sudo tee -a /etc/hosts
```

Access without a Host header flag:

```bash
# HTTP variant
curl -I http://java-monolith.local:<http-nodeport>/

# Example
curl -I http://java-monolith.local:31844/
```

> **Note**
> This works only on the machine where the `/etc/hosts` entry is added. It does not affect DNS
> resolution for any other host or device.

### Method 3 — Real DNS A record (public domain)

Point a DNS A record at the node's public IP. The NodePort is still required in the URL unless a
reverse proxy or Cloudflare Tunnel fronts the cluster.

```
A  java-monolith.ibtisam-iq.com  →  <node-public-ip>
```

> **Which IP to use in the A record**
> Use the node's **public IP**, not the internal cluster IP. On bare-metal with NodePort, the node
> IP is the ingress entry point. The NodePort itself (e.g. `32262`) is the port on that IP where
> ingress-nginx is listening.
>
> Retrieve the node IP:
> ```bash
> hostname -I
> kubectl get nodes -o wide
> ```

Access with real domain:

```bash
curl -I http://java-monolith.ibtisam-iq.com:<http-nodeport>/
```

### Method 4 — Cloudflare Tunnel (no public port exposure required)

Configure the tunnel origin to point at the NodePort service. The tunnel handles external TLS and
forwards traffic to the cluster.

```yaml
# Cloudflare Tunnel config
ingress:
  - hostname: java-monolith.ibtisam-iq.com
    service: http://<node-ip>:<http-nodeport>
    originRequest:
      httpHostHeader: java-monolith.ibtisam-iq.com
```

> **Important**
> The `httpHostHeader` field is mandatory. Without it, ingress-nginx receives a request with the
> Cloudflare internal hostname and returns 404 because no rule matches.

### Method 5 — curl from inside the cluster (pod-to-ingress)

```bash
# Exec into a pod in the same namespace
kubectl exec -it <pod-name> -n bankapp -- sh

# Hit the ingress-nginx service directly (cluster DNS)
curl -I -H "Host: java-monolith.local" \
  http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/
```

> **Note**
> After an Ingress is deployed, Kubernetes assigns it a cluster-internal IP visible in
> `kubectl get ingress -n bankapp`. This IP is reachable only from within the cluster.

---

## TLS Verification

```bash
# Check certificate status
kubectl get certificate -n bankapp
kubectl get secret java-monolith-tls -n bankapp

# Test HTTPS with real DNS
curl -I https://java-monolith.ibtisam-iq.com

# Inspect the issued certificate
openssl s_client \
  -connect java-monolith.ibtisam-iq.com:443 \
  -servername java-monolith.ibtisam-iq.com

# Test HTTPS with node IP (skip DNS, ignore self-signed warning)
curl -I -H "Host: java-monolith.ibtisam-iq.com" \
  https://<node-ip>:<https-nodeport>/ -k
```

---

## Troubleshooting

```bash
# Certificate not issued
kubectl describe certificate java-monolith-tls -n bankapp
kubectl describe certificaterequest -n bankapp
kubectl get challenges -n bankapp
kubectl logs -n cert-manager deploy/cert-manager

# Ingress not routing
kubectl describe ingress -n bankapp
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

> **Common failure: Cloudflare proxy + HTTP-01 challenge**
> Cloudflare intercepts port 80 before Let's Encrypt can complete the HTTP-01 challenge. Disable
> the proxy (grey cloud icon) on the DNS record during initial certificate issuance. Re-enable after
> the certificate is issued. Alternatively, switch to DNS-01 challenge for wildcard certificates
> (`*.ibtisam-iq.com`).

> **Rate limits**
> Let's Encrypt production: 50 certificates per registered domain per week, 5 failed validations
> per account per hour. Use the staging endpoint during testing:
> `https://acme-staging-v02.api.letsencrypt.org/directory`

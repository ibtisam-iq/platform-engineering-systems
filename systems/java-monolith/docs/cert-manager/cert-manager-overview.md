# cert-manager: TLS Certificate Automation on Bare-Metal Kubernetes

## 1. What cert-manager Does

cert-manager is a Kubernetes controller that automates the full TLS certificate lifecycle:

```
┌──────────────────────────────────────────────────────────────────────┐
│                  cert-manager Responsibilities                       │
│                                                                      │
│  1. Watch  → Ingress/Gateway resources with cert-manager annotation  │
│  2. Create → Certificate, CertificateRequest, Order, Challenge       │
│  3. Solve  → HTTP-01 or DNS-01 ACME challenge with Let's Encrypt     │
│  4. Store  → Issued certificate + private key in a Kubernetes Secret │
│  5. Renew  → Automatically re-issues 30 days before expiry           │
└──────────────────────────────────────────────────────────────────────┘
```

cert-manager does **not** serve traffic. It only manages certificate objects. The actual TLS termination happens at the ingress controller or Gateway controller level.

---

## 2. cert-manager Internal Components

Three pods run in the `cert-manager` namespace. Each has a distinct role:

```bash
kubectl get pods -n cert-manager
# NAME                                       READY   STATUS
# cert-manager-<hash>                        1/1     Running   ← Core controller
# cert-manager-cainjector-<hash>             1/1     Running   ← Injects CA bundles into webhooks
# cert-manager-webhook-<hash>                1/1     Running   ← Validates cert-manager resources
```

| Pod | Role |
|---|---|
| `cert-manager` | Core controller — watches resources, drives ACME workflow, creates/renews certs |
| `cert-manager-cainjector` | Injects CA data into `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` resources |
| `cert-manager-webhook` | Admission webhook — validates and mutates cert-manager CRD manifests at apply time |

> **Note:** If the webhook pod is not running, `kubectl apply` of any cert-manager resource (`ClusterIssuer`, `Certificate`, etc.) will be rejected at the API server level — even before the core controller sees it.

---

## 3. Installation Methods and the Gateway API Flag

### Method A — Raw Manifest (no Gateway API support by default)

Raw manifest install:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
```

- Installs cert-manager and all CRDs.
- **Does not** enable Gateway API support automatically.
- The `gatewayHTTPRoute` solver stays disabled until the controller is explicitly reconfigured.

To enable Gateway API support on a manifest‑based install, patch the `cert-manager` deployment to add the flag:

```bash
kubectl patch deployment cert-manager -n cert-manager \
  --type=json \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--enable-gateway-api"
  }]'

kubectl rollout status deployment/cert-manager -n cert-manager
```

Verification:

```bash
# 1) Check the deployment args directly
kubectl get deployment cert-manager -n cert-manager \
  -o jsonpath='{.spec.template.spec.containers[*].args}' \
  | tr ',' '\n'

# Expected to contain:
# --enable-gateway-api

# 2) Check logs for Gateway API controllers
kubectl logs deploy/cert-manager -n cert-manager | grep -E "gateway-shim|Gateway API|HTTPRoute"
# Expected:
# "enabling the sig-network Gateway API certificate-shim and HTTP-01 solver"
# "starting controller" ... controller="gateway-shim"
# "Caches populated" type="*v1.Gateway"
# "Caches populated" type="*v1.HTTPRoute"
```

> **Important:** This verification via `grep -i gateway` in `args` is correct only for the manifest + `kubectl patch` approach, where `--enable-gateway-api` is explicitly added as a container argument.
>

> **Summary:** Raw manifest install works, but Gateway API integration is **opt‑in** and requires a manual patch to add `--enable-gateway-api`.

***

### Method B — Helm (recommended, configurable, including Gateway API)

Helm install (OCI chart):

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.20.2 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

Key points:

- `crds.enabled=true` ensures cert-manager CRDs are installed by Helm. [cert-manager](https://cert-manager.io/docs/installation/helm/)
- `config.apiVersion` and `config.kind` define the controller configuration object.
- `config.enableGatewayAPI=true` enables:
  - Gateway API “shim” controller (watches Gateways and annotates Certificates).
  - HTTP‑01 solver for `gatewayHTTPRoute`.

With these values:

- No manual patch is required.
- Logs show:

  ```text
  "enabling the sig-network Gateway API certificate-shim and HTTP-01 solver"
  "starting controller" ... controller="gateway-shim"
  ```

> **Important:** The OCI chart **does not** automatically enable Gateway API; the `config.*` values are still required. The only difference between OCI and legacy repo is *where* the chart is pulled from, not the feature set. [artifacthub](https://artifacthub.io/packages/helm/cert-manager/cert-manager)

---

## 4. ClusterIssuer — Every Field Explained

```yaml
apiVersion: cert-manager.io/v1          # cert-manager API group, version v1 (stable since v1.0)
kind: ClusterIssuer                     # Cluster-scoped (no namespace) — reusable across all namespaces
                                        # Alternative: Issuer (namespace-scoped, only works within one namespace)
metadata:
  name: letsencrypt-prod                # Referenced by: Ingress/Gateway annotation OR Certificate spec.issuerRef.name
                                        # Any resource that references this name will use this issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    # ↑ Let's Encrypt production ACME directory endpoint
    # Rate limits apply: 50 certificates per registered domain per week
    # For testing use staging: https://acme-staging-v02.api.letsencrypt.org/directory
    # Staging certs are NOT trusted by browsers but have no rate limits

    email: contact@ibtisam-iq.com
    # ↑ Let's Encrypt sends expiry warnings here (60 days, 30 days, 7 days before expiry)
    # Not used for authentication — purely for notifications

    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    # ↑ cert-manager auto-creates this Secret in the cert-manager namespace
    # Contains the ACME account private key — NOT the TLS certificate
    # This key is used to sign ACME protocol messages (prove identity to Let's Encrypt)
    # Never create or modify this Secret manually

    solvers:
      - http01:                         # HTTP-01 challenge type (port 80 must be reachable)
                                        # Alternative: dns01 (for wildcard certs, no port 80 needed)
```

The `name` in `metadata` is the single most critical field. Every downstream resource that needs a certificate references it:

```yaml
# In Ingress
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod   # ← must match ClusterIssuer name exactly

# In Gateway
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod   # ← same

# In standalone Certificate resource
spec:
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    group: cert-manager.io
```

---

## 5. The Two Solver Types

The solver defines **how cert-manager proves domain ownership** to Let's Encrypt. The choice is determined entirely by what routing layer is in use.

### Solver A — `ingress` (for Ingress + ingress-nginx)

```yaml
solvers:
  - http01:
      ingress:
        ingressClassName: nginx    # must match ingressClassName in the Ingress spec
```

**What cert-manager creates behind the scenes:**

```
ClusterIssuer (solver: ingress)
    │
    └─► Creates temporary Ingress resource:
        metadata.name: cm-acme-http-solver-<hash>
        spec.ingressClassName: nginx
        spec.rules:
          - host: java-monolith.ibtisam-iq.com
            http.paths:
              - path: /.well-known/acme-challenge/<token>
                backend: cm-acme-http-solver-<hash>:8089
        │
        └─► Creates temporary Pod + Service:
            Pod: cm-acme-http-solver-<hash>  ← tiny Go HTTP server
            Service: cm-acme-http-solver-<hash> → port 8089
            │
            └─► ingress-nginx routes:
                http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>
                → solver Pod:8089
                → returns the key authorization token
```

> **Note:** The `class: nginx` field (without the `ingressClassName` prefix) is deprecated in cert-manager v1.5+. Always use `ingressClassName: nginx`.

### Solver B — `gatewayHTTPRoute` (for Gateway API + NGF/Envoy)

```yaml
solvers:
  - http01:
      gatewayHTTPRoute:
        parentRefs:
          - name: bankapp-gateway
            namespace: bankapp
            kind: Gateway
            group: gateway.networking.k8s.io   # required — defaults to core group if omitted
```

**What cert-manager creates behind the scenes:**

```
ClusterIssuer (solver: gatewayHTTPRoute)
    │
    └─► Creates temporary HTTPRoute resource:
        metadata.name: cm-acme-http-solver-<hash>
        spec.parentRefs:
          - name: bankapp-gateway (the referenced Gateway)
        spec.rules:
          - matches: path /.well-known/acme-challenge/<token>
            backendRefs: cm-acme-http-solver-<hash>:8089
        │
        └─► Creates temporary Pod + Service (same as ingress solver):
            Pod: cm-acme-http-solver-<hash>
            Service: cm-acme-http-solver-<hash> → port 8089
            │
            └─► Gateway controller (NGF/Envoy) routes:
                http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>
                → solver Pod:8089
                → returns the key authorization token
```

**Solver comparison:**

| | `ingress` solver | `gatewayHTTPRoute` solver |
|---|---|---|
| Routing layer | `Ingress` resource | `HTTPRoute` resource |
| Temporary resource created | `Ingress` | `HTTPRoute` |
| Requires `--enable-gateway-api` flag | ❌ No | ✅ Yes |
| Gateway API CRDs required | ❌ No | ✅ Yes |
| Works with ingress-nginx | ✅ Yes | ❌ No |
| Works with NGF / Envoy | ❌ No | ✅ Yes |

---

## 6. Certificate Lifecycle — Full Object Chain

When cert-manager detects a resource with the `cert-manager.io/cluster-issuer` annotation, it drives a chain of object creation. Every object in this chain is inspectable with `kubectl`:

```
Annotation on Ingress/Gateway
        │
        ▼
┌───────────────┐
│  Certificate  │  cert-manager auto-creates this
│  READY=False  │  spec.secretName: java-monolith-tls
│  (pending)    │  spec.issuerRef: letsencrypt-prod
└──────┬────────┘
       │
       ▼
┌──────────────────────┐
│  CertificateRequest  │  represents a single issuance attempt
│  Approved/Denied     │  contains the CSR (Certificate Signing Request)
└──────┬───────────────┘
       │
       ▼
┌───────────┐
│   Order   │  ACME order placed with Let's Encrypt
│  pending  │  contains authorization URLs
└──────┬────┘
       │
       ▼
┌───────────────┐
│   Challenge   │  one per domain in the certificate
│   pending     │  type: HTTP-01
│   presented   │  token: <random string>
└──────┬────────┘
       │  Let's Encrypt fetches:
       │  GET http://<domain>/.well-known/acme-challenge/<token>
       │
       ▼
┌──────────────────────┐
│  Challenge → valid   │  Let's Encrypt confirmed token
│  Order → valid       │
│  CertificateRequest  │
│    → Issued          │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────────────────┐
│  Secret: java-monolith-tls       │  auto-created by cert-manager
│  type: kubernetes.io/tls         │
│  data:                           │
│    tls.crt: <full cert chain>    │
│    tls.key: <private key>        │
└──────────────────────────────────┘
       │
       ▼
┌──────────────────────────┐
│  Certificate READY=True  │  ingress-nginx / NGF reads this Secret
│  90 days validity         │  and serves it on port 443
│  auto-renews at day 60   │
└──────────────────────────┘
```

---

## 7. The Bare-Metal Root Cause — Port 80 Connection Refused

This is the failure that blocks the entire chain above on bare-metal/EC2 with NodePort.

### What happens

```
cert-manager self-check:
  GET http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>
      │
      ├─► DNS resolves to 54.211.245.7
      ├─► TCP connect to 54.211.245.7:80
      └─► connection refused  ← NOTHING listening on port 80 on the host OS
```

### Why

ingress-nginx and NGF run as `NodePort` services. The OS port map looks like this:

```
Port 80  → NOTHING (no listener)      ← external traffic hits here, gets refused
Port 443 → NOTHING (no listener)      ← same

Port 30622 → ingress-nginx (HTTP)     ← actual nginx listener
Port 31571 → ingress-nginx (HTTPS)    ← actual nginx listener
```

The EC2 security group being open is irrelevant — the OS has no socket on 80/443.

### The failure cascade

```
Port 80 not reachable
    → cert-manager self-check fails
        → Challenge stays PENDING
            → Order stays PENDING
                → CertificateRequest not issued
                    → Certificate READY=False
                        → Secret java-monolith-tls never created
                            → ingress-nginx/NGF serves self-signed cert on 443
                                → browser shows "Not Secure" / curl SSL error
```

---

## 8. Fix Method A — iptables Port Forwarding

Intercepts packets in the `PREROUTING` chain before they reach any socket. Traffic arriving on port 80 is `REDIRECT`ed to the NodePort. The proxy pod does not need to be modified.

```bash
# Get the actual NodePorts first
kubectl get svc -n ingress-nginx   # for ingress-nginx
kubectl get svc -n nginx-gateway   # for NGF

# Apply redirect rules (replace port numbers with actual NodePorts)
HTTP_NODEPORT=30622
HTTPS_NODEPORT=31571

sudo iptables -t nat -A PREROUTING -p tcp --dport 80  -j REDIRECT --to-port $HTTP_NODEPORT
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port $HTTPS_NODEPORT

# Persist across reboots
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

Verify rules are active:

```bash
sudo iptables -t nat -L PREROUTING -n --line-numbers
# Look for lines: tcp dpt:80 redir ports 30622
#                 tcp dpt:443 redir ports 31571

# Confirm port 80 now reaches ingress
curl -I http://java-monolith.ibtisam-iq.com
# Expected: HTTP/1.1 308 or 301 (redirect to HTTPS)
```

---

## 9. Fix Method B — `hostNetwork: true` Patch

Makes the proxy pod bind directly to ports 80 and 443 on the host network interface. Eliminates the NodePort translation layer entirely.

> **Warning:** Before applying this patch, confirm nothing else is bound to ports 80 or 443 on the host:
> ```bash
> sudo ss -tlnp | grep -E ':80|:443'
> ```
> If another process is listed (e.g., a previous nginx, Apache), the pod will crash with `bind: address already in use`.

### Option 1 — kubectl patch (imperative)

```bash
# For ingress-nginx
kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
  --type=json \
  -p='[
    {"op": "add", "path": "/spec/template/spec/hostNetwork", "value": true},
    {"op": "add", "path": "/spec/template/spec/dnsPolicy",  "value": "ClusterFirstWithHostNet"}
  ]'

# For NGF
kubectl patch deployment ngf-nginx-gateway-fabric -n nginx-gateway \
  --type=json \
  -p='[
    {"op": "add", "path": "/spec/template/spec/hostNetwork", "value": true},
    {"op": "add", "path": "/spec/template/spec/dnsPolicy",  "value": "ClusterFirstWithHostNet"}
  ]'

# For Envoy Gateway
kubectl patch deployment envoy-gateway -n envoy-gateway-system \
  --type=json \
  -p='[
    {"op": "add", "path": "/spec/template/spec/hostNetwork", "value": true},
    {"op": "add", "path": "/spec/template/spec/dnsPolicy",  "value": "ClusterFirstWithHostNet"}
  ]'
```

### Option 2 — Kustomize patch (declarative)

```yaml
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: ingress-nginx-controller   # change per controller
      namespace: ingress-nginx         # change per controller
    patch: |-
      - op: add
        path: /spec/template/spec/hostNetwork
        value: true
      - op: add
        path: /spec/template/spec/dnsPolicy
        value: ClusterFirstWithHostNet
```

### Verify the patch

```bash
# Confirm pod restarted
kubectl rollout status deployment/<name> -n <namespace>

# Confirm hostNetwork is set to true
kubectl get deployment <name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.hostNetwork}'
# Expected: true

# Confirm the process is now bound to port 80 on the host
sudo ss -tlnp | grep -E ':80|:443'
# Expected: nginx or envoy process listed on *:80 and *:443
```

> **Note — Why `dnsPolicy: ClusterFirstWithHostNet`?** When `hostNetwork: true` is set, the pod inherits the node's `/etc/resolv.conf` by default. This breaks in-cluster DNS (e.g., `bankapp-service.bankapp.svc.cluster.local` stops resolving). `ClusterFirstWithHostNet` restores cluster DNS while keeping host networking active.

---

## 10. Verification at Every Stage

Run these commands in sequence after deployment to confirm each stage of the pipeline is healthy:

```bash
# Stage 1 — ClusterIssuer is registered and ready
kubectl get clusterissuer letsencrypt-prod
kubectl describe clusterissuer letsencrypt-prod
# Look for: Status.Conditions: Ready=True

# Stage 2 — Certificate object was created by cert-manager
kubectl get certificate -n bankapp
# NAME                READY   SECRET              AGE
# java-monolith-tls   False   java-monolith-tls   30s  ← False is expected at this point

# Stage 3 — Order was placed with Let's Encrypt
kubectl get order -n bankapp
# NAME                             STATE     AGE
# java-monolith-tls-1-<hash>       pending   30s

# Stage 4 — HTTP-01 Challenge was created and presented
kubectl get challenges -n bankapp
# NAME                                    STATE     DOMAIN                          AGE
# java-monolith-tls-1-<hash>-<hash>       pending   java-monolith.ibtisam-iq.com    30s

# Stage 5 — Inspect the challenge for errors
kubectl describe challenge <challenge-name> -n bankapp
# Look at Status.Reason — this is where port 80 errors appear

# Stage 6 — Manually verify the challenge token is reachable
# Get the token from the challenge description above, then:
curl http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>
# Expected: the key authorization string (not connection refused, not 404)

# Stage 7 — Watch certificate become ready
kubectl get certificate -n bankapp -w
# java-monolith-tls   True   java-monolith-tls   2m  ← True means cert issued

# Stage 8 — Confirm Secret was created with the cert
kubectl get secret java-monolith-tls -n bankapp
# NAME                TYPE                DATA   AGE
# java-monolith-tls   kubernetes.io/tls   2      2m

kubectl describe secret java-monolith-tls -n bankapp
# Data: tls.crt, tls.key  ← both must be present

# Stage 9 — Confirm HTTPS is working end-to-end
curl -I https://java-monolith.ibtisam-iq.com
# Expected: HTTP/2 200 or 302 (no SSL error)

# Stage 10 — Inspect the actual certificate served
openssl s_client -connect java-monolith.ibtisam-iq.com:443 \
  -servername java-monolith.ibtisam-iq.com 2>/dev/null \
  | openssl x509 -noout -dates -issuer
# issuer=C=US, O=Let's Encrypt  ← confirms real cert, not self-signed
```

---

## 11. Troubleshooting Reference

```bash
# cert-manager controller logs — primary source of truth
kubectl logs -n cert-manager deploy/cert-manager --tail=100 -f

# Webhook logs — check if resource validation is failing
kubectl logs -n cert-manager deploy/cert-manager-webhook --tail=50

# Full CertificateRequest details
kubectl describe certificaterequest -n bankapp

# Check all cert-manager owned resources in one namespace
kubectl get certificate,certificaterequest,order,challenge -n bankapp

# Check cert-manager RBAC — missing permissions show up here
kubectl auth can-i create httproutes --as=system:serviceaccount:cert-manager:cert-manager -n bankapp
# Expected: yes  (only relevant when using gatewayHTTPRoute solver)
```

**Common failure patterns:**

| Symptom | Cause | Fix |
|---|---|---|
| Challenge `pending`, reason: `connection refused on :80` | Nothing listening on port 80 | Apply iptables or hostNetwork patch |
| Certificate stuck `READY=False`, no challenges created | `--enable-gateway-api` missing on cert-manager | Patch cert-manager deployment |
| Challenge `pending`, reason: `404 on /.well-known/` | Solver created the route but controller not routing it | Verify `ingressClassName` or `parentRefs` matches actual controller |
| `no matches for kind "HTTPRoute"` | Gateway API CRDs not installed | Install CRDs before deploying `ClusterIssuer` with gateway solver |
| `ClusterIssuer not ready` | ACME account registration failed | Check cert-manager logs, verify email, check network egress to `acme-v02.api.letsencrypt.org` |
| Secret `java-monolith-tls` has 1 data key instead of 2 | Temporary secret (pre-issuance placeholder) | Wait for challenge to complete, secret will be replaced |

# cert-manager Gateway HTTP-01 Troubleshooting on Bare-Metal EC2

This runbook documents troubleshooting and resolution of `HTTP-01` ACME challenges getting stuck in `pending` with `connection refused` when using `cert-manager`’s **Gateway HTTPRoute solver** with **NGINX Gateway Fabric (NGF)** on a **single-node bare‑metal EC2** cluster.

---

## 1. Scenario Overview

**Context**

- Kubernetes: single-node control‑plane on EC2.
- CNI: Calico.
- Ingress layer: NGINX Gateway Fabric (Gateway API), not ingress‑nginx.
- ACME: cert-manager `ClusterIssuer` with HTTP‑01, `gatewayHTTPRoute` solver.
- Domain: `java-monolith.ibtisam-iq.com`.
- Symptom: `Challenge` objects remain `pending` with `connection refused` on port 80.

The traffic path for HTTP‑01 in this setup:

```text
Let's Encrypt / cert-manager self-check
    ↓
java-monolith.ibtisam-iq.com:80  (public → EC2)
    ↓
EC2 node iptables (port 80/443) → Gateway Service NodePorts
    ↓
NGF per-Gateway nginx Service (NodePort)
    ↓
Gateway + temporary HTTPRoute + solver Service/Pod
    ↓
ACME token served
```

The failure mode:

- `Challenge.Status.Reason` shows `dial tcp <public-ip>:80: connect: connection refused`.
- `Challenge.State` remains `pending`.

---

## 2. Baseline Checks

**Checklist**

Confirm that core control plane, cert-manager, NGF, and Gateway API CRDs are healthy before deeper debugging.

```bash
# 
curl ifconfig.me
dig +short java-monolith.ibtisam-iq.com

# Kubernetes control-plane and addons
kubectl get nodes
kubectl get pods -A

# cert-manager components
kubectl get pods -n cert-manager
kubectl logs deploy/cert-manager -n cert-manager --tail=80

# NGINX Gateway Fabric (controller)
kubectl get pods -n nginx-gateway
kubectl get svc -n nginx-gateway

# Gateway API CRDs
kubectl get crd | grep gateway.networking.k8s.io

# GatewayClass from NGF
kubectl get gatewayclass
kubectl describe gatewayclass nginx
```

Expected:

- `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook` in `Running`.
- NGF controller Deployment `ngf-nginx-gateway-fabric` in `Running`.
- Gateway API CRDs installed (`gateways.gateway.networking.k8s.io`, `httproutes.gateway.networking.k8s.io`, etc.).
- `GatewayClass nginx` accepted and supported (`Status.Conditions: Accepted=True`).

---

## 3. Gateway + Certificate Wiring

**Goal**

Ensure that the Gateway, HTTPRoutes, ClusterIssuer, and Certificate are aligned for the Gateway HTTP‑01 solver.

### 3.1 Gateway and HTTPRoutes

```bash
kubectl get gateway -n bankapp
kubectl describe gateway bankapp-gateway -n bankapp

kubectl get httproute -n bankapp
kubectl describe httproute bankapp-route -n bankapp
kubectl describe httproute bankapp-http-redirect -n bankapp
```

Key expectations:

- `Gateway.spec.gatewayClassName: nginx`.
- Listeners:

  ```text
  - name: http
    port: 80
    protocol: HTTP
    hostname: java-monolith.ibtisam-iq.com

  - name: https
    port: 443
    protocol: HTTPS
    hostname: java-monolith.ibtisam-iq.com
    tls.certificateRefs: Secret java-monolith-tls
  ```

- `Gateway.status.conditions` contains `Programmed=True` and `Accepted=True`.
- `Gateway.status.listeners[*].attachedRoutes` shows HTTPRoutes attached.

### 3.2 ClusterIssuer and Certificate

```bash
kubectl get clusterissuer letsencrypt-prod
kubectl describe clusterissuer letsencrypt-prod

kubectl get certificate -n bankapp
kubectl describe certificate java-monolith-tls -n bankapp
```

Key fields:

- `ClusterIssuer.spec.acme.solvers[0].http01.gatewayHTTPRoute.parentRefs` points at `bankapp-gateway`.
- `Certificate.spec`:

  ```text
  dnsNames: ["java-monolith.ibtisam-iq.com"]
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  secretName: java-monolith-tls
  ```

- `Certificate.Status.Conditions` show `Issuing=True`, `Ready=False` with `Reason=DoesNotExist` initially.

---

## 4. Understanding the Failure: Challenge Pending with Connection Refused

**Observation**

Old Challenge objects show:

```text
Reason: Waiting for HTTP-01 challenge propagation:
failed to perform self check GET request 'http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>':
Get "http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>":
dial tcp <public-ec2-ip>:80: connect: connection refused
State: pending
```

Interpretation:

- cert-manager **did** create the Gateway HTTPRoute solver (Challenge `Presented=true`).
- Temporary solver Pod/Service and HTTPRoute exist.
- However, TCP `SYN` to `<public-ec2-ip>:80` reaches the node and receives `RST` (no listener), producing `connection refused`.

Root cause at this stage:

- No process is bound to host port 80/443, and no iptables DNAT/REDIRECT exists.
- `ngf-nginx-gateway-fabric` Service is `ClusterIP` only; it does not expose 80/443 on the node.

---

## 5. Per-Gateway NGINX Service and NodePorts

**Key concept**

NGF creates a **per‑Gateway nginx Deployment and Service** in the application namespace. This per‑Gateway Service is the correct NodePort entrypoint, not the controller Service.

### 5.1 Identify per-Gateway nginx resources

```bash
kubectl get deploy -n bankapp
kubectl get svc -n bankapp
```

Example output:

```text
# Deployments
bankapp                  1/1   Running
bankapp-gateway-nginx    1/1   Running
mysql                    1/1   Running

# Services
bankapp-gateway-nginx    NodePort  10.105.42.246  80:32030/TCP,443:32315/TCP
bankapp-service          NodePort  10.98.85.236   80:30082/TCP
cm-acme-http-solver-…    NodePort  10.104.173.56  8089:32563/TCP
mysql-service            ClusterIP 10.108.102.184 3306/TCP
```

Important distinctions:

- `bankapp-gateway-nginx` (NodePort) is the nginx data plane for Gateway `bankapp-gateway`.
- `ngf-nginx-gateway-fabric` (in `nginx-gateway` namespace) remains `ClusterIP` and is not the external entrypoint.

### 5.2 Verify HTTP path through Gateway NodePort

```bash
HTTP_NODEPORT=32030   # from bankapp-gateway-nginx service

curl -I -H "Host: java-monolith.ibtisam-iq.com"   http://172.31.86.199:$HTTP_NODEPORT
```

Expected result:

```text
HTTP/1.1 301 Moved Permanently
Location: https://java-monolith.ibtisam-iq.com/
```

Consequences:

- Gateway HTTP listener on port 80 is functional.
- HTTPRoute handling HTTP→HTTPS redirect is active.

---

## 6. iptables Hairpin for Ports 80 and 443

**Goal**

Wire host ports 80 and 443 to the Gateway NodePorts on a single EC2 node, for both external traffic and cert-manager self‑checks.

### 6.1 Configure iptables redirects

```bash
HTTP_NODEPORT=32030    # bankapp-gateway-nginx 80 NodePort
HTTPS_NODEPORT=32315   # bankapp-gateway-nginx 443 NodePort

# Optional: remove legacy redirect rule if present (adjust index as needed)
sudo iptables -t nat -D PREROUTING 3 2>/dev/null || true

# Redirect incoming port 80 → Gateway HTTP NodePort
sudo iptables -t nat -A PREROUTING   -p tcp --dport 80   -j REDIRECT --to-port $HTTP_NODEPORT

# Redirect incoming port 443 → Gateway HTTPS NodePort
sudo iptables -t nat -A PREROUTING   -p tcp --dport 443   -j REDIRECT --to-port $HTTPS_NODEPORT

# Redirect locally generated traffic to NodePorts as well
sudo iptables -t nat -A OUTPUT   -p tcp --dport 80   -j REDIRECT --to-port $HTTP_NODEPORT
sudo iptables -t nat -A OUTPUT   -p tcp --dport 443   -j REDIRECT --to-port $HTTPS_NODEPORT

# Inspect rules
sudo iptables -t nat -L PREROUTING -n --line-numbers | grep -E 'dpt:80|dpt:443'
sudo iptables -t nat -L OUTPUT -n --line-numbers | grep -E 'dpt:80|dpt:443'
```

Expected pattern:

```text
PREROUTING ... tcp dpt:80  redir ports 32030
PREROUTING ... tcp dpt:443 redir ports 32315
OUTPUT     ... tcp dpt:80  redir ports 32030
OUTPUT     ... tcp dpt:443 redir ports 32315
```

Result:

- Any TCP to node:80/443 (from external clients) is redirected to the Gateway NodePorts.
- Any TCP to node:80/443 (generated from the node itself) is also redirected, enabling `curl` tests from the host.

---

## 7. CoreDNS Override to Node Private IP

**Problem**

cert-manager self‑check uses the domain’s DNS A record. On EC2, that A record is the **public** IP. Traffic from pods to the public IP may fail due to AWS hairpin routing or because the iptables rules are only effective against traffic to the private IP.

**Goal**

Make `java-monolith.ibtisam-iq.com` resolve to the node’s **private** IP inside the cluster.

### 7.1 Modify CoreDNS ConfigMap

```bash
kubectl edit configmap coredns -n kube-system
```

Inside the `Corefile` for `.:53 { ... }`, insert:

```txt
    hosts {
      172.31.86.199 java-monolith.ibtisam-iq.com
      fallthrough
    }
```

Ensure this block appears **before** the `forward . /etc/resolv.conf` stanza.

### 7.2 Restart CoreDNS and verify resolution

```bash
kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system

# From an existing pod, e.g., bankapp
kubectl exec -it bankapp-<pod> -n bankapp --   getent hosts java-monolith.ibtisam-iq.com
```

Expected result:

```text
172.31.86.199   java-monolith.ibtisam-iq.com
```

Effect:

- Inside the cluster, cert-manager and other workloads treat `java-monolith.ibtisam-iq.com` as the node private IP.
- Combined with iptables, self‑check traffic reaches the Gateway NodePorts instead of leaking out to the public address.

---

## 8. Forcing a Fresh ACME Flow

**Purpose**

Clear old Orders/Challenges created before the DNS and iptables fixes, then trigger a new issuance.

### 8.1 Delete existing Orders and Challenges

```bash
kubectl delete challenge -n bankapp --all
kubectl delete order -n bankapp --all
```

### 8.2 Force Certificate renewal

```bash
kubectl annotate certificate java-monolith-tls -n bankapp   cert-manager.io/force-renewal="true" --overwrite
```

### 8.3 Observe new Order and Challenge

```bash
kubectl get order -n bankapp
kubectl get challenges -n bankapp -w
```

Expected progression:

- New `Order` `java-monolith-tls-1-<hash>` appears.
- New `Challenge` `java-monolith-tls-1-<hash>-<hash>` is created.
- `Challenge.Status.Presented` becomes `true`.
- `Challenge.Status.State` transitions from `pending` → `valid`.

`Challenge.Status.Reason` should no longer contain `dial tcp <public-ip>:80: connect: connection refused`.

---

## 9. Final Verification

**Goal**

Confirm issuance, Secret, and Gateway TLS behavior.

### 9.1 Certificate and Secret

```bash
kubectl get certificate -n bankapp
kubectl describe certificate java-monolith-tls -n bankapp
```

Expected:

- `READY=True`.
- Condition `Reason: Issued`.

```bash
kubectl get secret java-monolith-tls -n bankapp
kubectl describe secret java-monolith-tls -n bankapp
```

Expected:

- `Type: kubernetes.io/tls`.
- Data keys: `tls.crt` and `tls.key`.

### 9.2 Gateway HTTPS listener

```bash
kubectl describe gateway bankapp-gateway -n bankapp
```

Expected for `https` listener:

- `Conditions: Accepted=True, ResolvedRefs=True, Programmed=True`.
- No more `Secret bankapp/java-monolith-tls does not exist` messages.

### 9.3 External HTTPS test

```bash
curl -I https://java-monolith.ibtisam-iq.com -k

openssl s_client -connect java-monolith.ibtisam-iq.com:443   -servername java-monolith.ibtisam-iq.com 2>/dev/null   | openssl x509 -noout -issuer -dates
```

Expected:

- HTTP status `200/302` (no `connection refused`).
- Certificate issuer: Let’s Encrypt.

---

## 10. Notes on Ingress vs Gateway

**Clarification**

This runbook is **specific** to Gateway API with NGF and the `gatewayHTTPRoute` solver on bare‑metal EC2.

- For ingress‑nginx with `ingress` HTTP‑01 solver:
    - NodePort or LoadBalancer for ingress controller is usually sufficient.
    - iptables hairpin rules and CoreDNS host overrides are typically unnecessary.

- For Gateway + NGF:
    - The relevant data‑plane Service is the **per‑Gateway** `*-gateway-nginx` Service (NodePort), not the NGF controller Service.
    - On single-node EC2, DNS override and iptables hairpin are required to satisfy cert-manager’s self‑check semantics reliably.

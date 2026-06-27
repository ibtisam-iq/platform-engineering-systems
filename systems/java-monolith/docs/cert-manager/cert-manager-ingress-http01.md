# cert-manager TLS Certificate Automation on Bare-Metal Kubernetes – HTTP-01 with Ingress on EC2 Public IP

Public IP as DNS target on a single-node EC2 cluster is fully workable as long as port 80 on that public IP reliably reaches the ingress NodePort and cert-manager can perform both self-check and external validation.

## Overview and goals

This runbook describes end-to-end setup of cert-manager on a fresh EC2-based Kubernetes cluster, using:

- Public DNS (A record) pointing to the EC2 public IP.
- Ingress-nginx as the HTTP routing layer.
- cert-manager with **HTTP‑01 Ingress solver** for Let’s Encrypt (staging and production). [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

Primary goal: automatic TLS for `java-monolith.ibtisam-iq.com` with a ClusterIssuer and a single Ingress, plus a clear troubleshooting path when Challenges stay pending with `connection refused` on port 80. [community.letsencrypt](https://community.letsencrypt.org/t/waiting-for-http-01-challenge-propagation-failed-to-perform-self-check-get-request/181020)

---

## Phase 1 – EC2, Kubernetes, ingress

1. EC2 and networking

- Launch Ubuntu EC2, attach an Elastic IP (stable public IP). [letsencrypt](https://letsencrypt.org/docs/allow-port-80/)
- Security group inbound rules:
  - 22 (SSH) from admin source.
  - 80 (HTTP) from 0.0.0.0/0.
  - 443 (HTTPS) from 0.0.0.0/0. [letsencrypt](https://letsencrypt.org/docs/allow-port-80/)

2. Kubernetes install (single node)

- Install a simple single-node cluster, e.g. k3s with Traefik disabled:

  ```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
  kubectl get nodes
  kubectl get pods -A
  ```

- Ensure all system pods are running before proceeding.

3. Ingress-nginx controller

- Install ingress-nginx:

  ```bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update

  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx --create-namespace
  ```

- Verify:

  ```bash
  kubectl get pods -n ingress-nginx
  kubectl get svc -n ingress-nginx
  ```

- For bare‑metal / EC2 without cloud LoadBalancer, the controller exposes HTTP/HTTPS via NodePorts (for example 30622 for HTTP, 31571 for HTTPS).

---

## Phase 2 – DNS and port 80 hairpin to NodePort

4. DNS: public IP

- Create A record:

  - Name: `java-monolith.ibtisam-iq.com`
  - Value: EC2 **public** IP (for example 3.85.x.x).

- Validate on the node:

  ```bash
  dig +short java-monolith.ibtisam-iq.com
  # Expected: EC2 public IP
  ```

5. Bare-metal root cause: nothing on 80/443

On a fresh EC2 with ingress-nginx as NodePort-only, OS-level ports look like:

- Port 80: nothing listening.
- Port 443: nothing listening.
- Port 30622: ingress-nginx HTTP listener.
- Port 31571: ingress-nginx HTTPS listener.

External traffic hits EC2:80/443, finds no listener, and receives `connection refused`; security group openness is irrelevant if no process binds those ports.

6. Fix A: iptables redirect 80/443 → NodePort

- Retrieve NodePorts:

  ```bash
  kubectl get svc -n ingress-nginx
  ```

- Apply NAT rules:

  ```bash
  HTTP_NODEPORT=30622      # replace with actual
  HTTPS_NODEPORT=31571     # replace with actual

  sudo iptables -t nat -A PREROUTING -p tcp --dport 80  -j REDIRECT --to-port $HTTP_NODEPORT
  sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port $HTTPS_NODEPORT
  ```

- Verify:

  ```bash
  sudo iptables -t nat -L PREROUTING -n --line-numbers | grep dpt:
  # Expected dpt:80 redir ports HTTP_NODEPORT, dpt:443 redir ports HTTPS_NODEPORT
  ```

- Persist with `iptables-persistent` if desired:

  ```bash
  sudo apt-get install -y iptables-persistent
  sudo netfilter-persistent save
  ```

7. Basic HTTP test through public DNS

- Deploy a simple app and ingress:

  ```bash
  kubectl create namespace bankapp

  kubectl -n bankapp create deployment whoami \
    --image=traefik/whoami --port=80

  kubectl -n bankapp expose deployment whoami --port=80

  cat <<EOF | kubectl apply -f -
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: whoami-ingress
    namespace: bankapp
    annotations:
      kubernetes.io/ingress.class: "nginx"
  spec:
    rules:
    - host: java-monolith.ibtisam-iq.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: whoami
              port:
                number: 80
  EOF
  ```

- From an external client and from the node:

  ```bash
  curl -v -H "Host: java-monolith.ibtisam-iq.com" \
    http://java-monolith.ibtisam-iq.com/
  ```

Expected: 200 from whoami. If `connection refused` appears here, port 80 redirect is still not working and cert-manager HTTP‑01 will fail in the same way. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

---

## Phase 3 – cert-manager with Ingress HTTP‑01 solver

8. Install cert-manager

- Using Helm (supported method):

  ```bash
  helm repo add jetstack https://charts.jetstack.io
  helm repo update

  kubectl create namespace cert-manager

  helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --set installCRDs=true
  ```

- Confirm controller pods:

  ```bash
  kubectl get pods -n cert-manager
  # cert-manager, cert-manager-cainjector, cert-manager-webhook should all be Running
  ```



9. Internet egress validation

- On node:

  ```bash
  curl -v https://acme-staging-v02.api.letsencrypt.org/directory
  ```

If this fails, ACME account registration and challenges cannot succeed; outbound network must be fixed first. [community.letsencrypt](https://community.letsencrypt.org/t/letsencrypt-doesnt-work-for-different-ports/17519)

10. ClusterIssuer (staging)

- Example manifest:

  ```yaml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-staging
  spec:
    acme:
      email: contact@ibtisam-iq.com
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      privateKeySecretRef:
        name: letsencrypt-staging-account-key
      solvers:
      - http01:
          ingress:
            ingressClassName: nginx
  ```

- Apply:

  ```bash
  kubectl apply -f clusterissuer-letsencrypt-staging.yaml
  kubectl describe clusterissuer letsencrypt-staging
  ```

Status should show `Ready=True`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

---

## Phase 4 – Application ingress and certificate request

11. Application deployment and ingress with TLS

- Deploy application `java-monolith` in `bankapp` namespace.

- Ingress manifest using staging issuer:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: java-monolith-ingress
    namespace: bankapp
    annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/cluster-issuer: "letsencrypt-staging"
      acme.cert-manager.io/http01-edit-in-place: "true"
  spec:
    tls:
    - hosts:
      - java-monolith.ibtisam-iq.com
      secretName: java-monolith-tls
    rules:
    - host: java-monolith.ibtisam-iq.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: java-monolith
              port:
                number: 80
  ```

- Apply and check:

  ```bash
  kubectl apply -f java-monolith-ingress.yaml
  kubectl get ingress -n bankapp
  ```

12. cert-manager object chain

Once the annotated Ingress exists, cert-manager creates:

- Certificate: `java-monolith-tls` (READY False initially).
- CertificateRequest: one per issuance attempt.
- Order: ACME Order at Let’s Encrypt.
- Challenge: HTTP‑01 challenge for `java-monolith.ibtisam-iq.com`.

Quick inspection:

```bash
kubectl get certificate,certificaterequest,order,challenge -n bankapp
kubectl describe certificate java-monolith-tls -n bankapp
```

---

## Phase 5 – HTTP‑01 challenge flow with public IP

13. Understanding the HTTP‑01 path with public IP

For DNS pointing at the EC2 public IP:

- Let’s Encrypt:
  `ACME → java-monolith.ibtisam-iq.com:80 → EC2 public 80 → iptables redirect → ingress NodePort → cm-acme-http-solver pod`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)
- cert-manager self-check:
  `cert-manager pod → http://java-monolith.ibtisam-iq.com/.well-known/... → resolves to EC2 public IP → node networking stack → same path as above`. [community.letsencrypt](https://community.letsencrypt.org/t/waiting-for-http-01-challenge-propagation-failed-to-perform-self-check-get-request/181020)

Any `connection refused` on `3.x.x.x:80` breaks **both** external validation and self-check, leaving the Challenge stuck in `pending`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

14. Inspect Challenge resource

- Get and describe:

  ```bash
  kubectl get challenge -n bankapp
  kubectl describe challenge -n bankapp <challenge-name>
  ```

Typical fields:

- `Spec.DnsName: java-monolith.ibtisam-iq.com`
- Solver: `http01.ingress.ingressClassName: nginx`
- `Status.Reason` example:

  - `Waiting for HTTP-01 challenge propagation: failed to perform self check GET request 'http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<TOKEN>': Get ... dial tcp 3.85.x.x:80: connect: connection refused`. [community.letsencrypt](https://community.letsencrypt.org/t/waiting-for-http-01-challenge-propagation-failed-to-perform-self-check-get-request/181020)

Reason: public IP 80 not reaching ingress NodePort.

15. Verify temporary solver Ingress

- cert-manager creates or patches an ingress in `bankapp` with host `java-monolith.ibtisam-iq.com` and path `/.well-known/acme-challenge/<TOKEN>`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

Check:

```bash
kubectl get ingress -n bankapp
kubectl describe ingress -n bankapp <acme-http-solver-ingress>
```

Host and path must match the Challenge.

16. Manual challenge URL test (critical)

- From external machine:

  ```bash
  curl -v -H "Host: java-monolith.ibtisam-iq.com" \
    http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<TOKEN>
  ```

- From EC2 node:

  ```bash
  curl -v -H "Host: java-monolith.ibtisam-iq.com" \
    http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<TOKEN>
  ```

Expected: 200 and key authorization body. `connection refused` indicates port 80 handling is still wrong on the host. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

---

## Phase 6 – Alternative fix: hostNetwork for ingress

If iptables redirect proves fragile (especially with public IP hairpin), ingress controller can bind directly to host ports 80 and 443 using `hostNetwork: true`.

17. hostNetwork patch for ingress-nginx

- Patch deployment:

  ```bash
  kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
    --type=json \
    -p='[
      {"op": "add", "path": "/spec/template/spec/hostNetwork", "value": true},
      {"op": "add", "path": "/spec/template/spec/dnsPolicy", "value": "ClusterFirstWithHostNet"}
    ]'
  ```

- Wait for rollout:

  ```bash
  kubectl rollout status deployment ingress-nginx-controller -n ingress-nginx
  ```

- Validate host binding:

  ```bash
  sudo ss -tlnp | grep -E ':80|:443'
  # Expected: nginx process bound to 0.0.0.0:80 and 0.0.0.0:443
  ```

No iptables NAT is required in this mode; EC2 security group + process binding handle 80/443.

18. Retest challenge URL

- Repeat HTTP‑01 tests:

  ```bash
  curl -v -H "Host: java-monolith.ibtisam-iq.com" \
    http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<TOKEN>
  ```

Successful 200 here implies:

- Let’s Encrypt can reach the solver.
- cert-manager self-check should start passing; Challenge transitions from `pending` to `valid`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

---

## Phase 7 – Certificate readiness and production switch

19. Confirm success chain

Run staged verification sequence:

```bash
# Stage 1: ClusterIssuer ready
kubectl get clusterissuer letsencrypt-staging
kubectl describe clusterissuer letsencrypt-staging

# Stage 2: Certificate created
kubectl get certificate -n bankapp

# Stage 3: Order
kubectl get order -n bankapp

# Stage 4: Challenge
kubectl get challenge -n bankapp
kubectl describe challenge -n bankapp <challenge-name>

# Stage 5: Manual token curl (already done)

# Stage 6: Wait for Certificate READY
kubectl get certificate -n bankapp -w
```

Expected end state: `java-monolith-tls` is `READY=True`, secret exists with `tls.crt` and `tls.key`. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

20. Test HTTPS

- From external client:

  ```bash
  curl -vk https://java-monolith.ibtisam-iq.com/
  ```

- Optional deep inspection:

  ```bash
  openssl s_client -connect java-monolith.ibtisam-iq.com:443 \
    -servername java-monolith.ibtisam-iq.com 2>/dev/null \
  | openssl x509 -noout -dates -issuer
  ```

Issuer should be Let’s Encrypt, not self-signed. [community.letsencrypt](https://community.letsencrypt.org/t/letsencrypt-doesnt-work-for-different-ports/17519)

21. Move from staging to production

- Create production ClusterIssuer with `server: https://acme-v02.api.letsencrypt.org/directory`. [community.letsencrypt](https://community.letsencrypt.org/t/letsencrypt-doesnt-work-for-different-ports/17519)
- Update Ingress annotation:

  ```yaml
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
  ```

- Re-apply ingress; repeat challenge verification and HTTP‑01 tests. [cert-manager](https://cert-manager.io/v1.5-docs/faq/acme/)

---

## Quick reference: common failures with public IP DNS

| Symptom (public IP DNS)                                                                 | Root cause                                               | Fix                                                                 |
|-----------------------------------------------------------------------------------------|----------------------------------------------------------|----------------------------------------------------------------------|
| `connection refused` on `http://java-monolith.../.well-known/...`                      | OS not listening on 80                                   | Apply iptables redirect OR hostNetwork for ingress-nginx            |
| Challenge `pending`, reason self-check `dial tcp <public-ip>:80: connect: refused`     | Same as above, affects cert-manager self-check           | Fix port 80 path, then re-run curl and monitor Challenge            |
| 404 on challenge path                                                                   | Solver ingress route not matching host/path              | Inspect solver Ingress, ensure `host` and path `.well-known/...`    |
| ClusterIssuer not ready                                                                 | ACME directory unreachable, or misconfigured server URL  | Check network to ACME, server URL, cert-manager controller logs     |

 [community.letsencrypt](https://community.letsencrypt.org/t/waiting-for-http-01-challenge-propagation-failed-to-perform-self-check-get-request/181020)

This structure is already close to the existing `cert-manager.md` content; sections can be merged directly into the “Bare-Metal Root Cause” and “Fix Method A/B” chapters so future EC2 labs with public IP DNS follow the same deterministic path.

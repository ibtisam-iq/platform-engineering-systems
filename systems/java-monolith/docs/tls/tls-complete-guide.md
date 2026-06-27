# Complete TLS Guide: Three Approaches for Bare-Metal Kubernetes Behind NAT
# Author: Muhammad Ibtisam Iqbal
# Date: 2026-04-27
# Environment: bare-metal kubeadm cluster (172.16.0.2, behind NAT, no public IP)

================================================================================
OVERVIEW: WHY HTTP-01 FAILS IN YOUR ENVIRONMENT
================================================================================

Your cluster is behind NAT with private IP 172.16.0.2.
Let's Encrypt HTTP-01 challenge requires:
  1. Public DNS record pointing to your cluster
  2. Let's Encrypt servers reaching http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/<token>
  3. Port 80 accessible from the internet

Since 172.16.0.2 is private (RFC 1918), Let's Encrypt CANNOT reach it.
HTTP-01 will always fail with timeout/connection refused.

Solution Matrix:

┌──────────────────┬─────────────┬──────────────┬─────────────────┐
│ Approach         │ Real Cert?  │ Public DNS?  │ Complexity      │
├──────────────────┼─────────────┼──────────────┼─────────────────┤
│ DNS-01 Challenge │ Yes (LE)    │ Required     │ Medium          │
│ Cloudflare Tunnel│ Yes (CF)    │ Auto-managed │ Low             │
│ Self-Signed Cert │ No (test)   │ Not required │ Very Low        │
└──────────────────┴─────────────┴──────────────┴─────────────────┘

All three approaches are documented below with complete step-by-step instructions.


================================================================================
APPROACH 1: DNS-01 CHALLENGE WITH CERT-MANAGER + CLOUDFLARE
================================================================================

DNS-01 Challenge Overview:
  - Proves domain ownership by creating a DNS TXT record
  - Let's Encrypt queries DNS, not your cluster IP
  - Works behind NAT/firewall (no inbound connectivity needed)
  - Requires API access to your DNS provider (Cloudflare, Route53, etc.)

Prerequisites:
  1. Domain registered and managed by a supported DNS provider
  2. DNS provider API token with Zone:Edit permissions
  3. cert-manager installed with ExperimentalGatewayAPISupport enabled
  4. Gateway API CRDs installed

Supported DNS Providers:
  - Cloudflare (most common, documented here)
  - AWS Route53
  - Google Cloud DNS
  - DigitalOcean DNS
  - Azure DNS
  - 20+ others: https://cert-manager.io/docs/configuration/acme/dns01/

We'll use Cloudflare as the example (same pattern for other providers).


Step 1: Get Cloudflare API Token
---------------------------------

1. Log in to Cloudflare Dashboard: https://dash.cloudflare.com

2. Navigate to: My Profile → API Tokens → Create Token

3. Use template: "Edit zone DNS" or create custom token with:

   Permissions:
     Zone | DNS | Edit
     Zone | Zone | Read

   Zone Resources:
     Include | Specific zone | ibtisam-iq.com

   TTL: No expiry (or set to 1 year)

4. Create Token → Copy the token (shows only once)

   Example token: XYZ123ABC456DEF789GHI012JKL345MNO678PQR901STU234VWX567YZ


Step 2: Store API Token as Kubernetes Secret
---------------------------------------------

Create secret in cert-manager namespace (not bankapp namespace):

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: XYZ123ABC456DEF789GHI012JKL345MNO678PQR901STU234VWX567YZ
EOF

Verify secret created:

kubectl get secret cloudflare-api-token -n cert-manager


Step 3: Update ClusterIssuer to Use DNS-01
-------------------------------------------

Edit k8s/overlays/bare-metal/gateway-cert.yaml:

# k8s/overlays/bare-metal/gateway-cert.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: contact@ibtisam-iq.com
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
      # DNS-01 solver for Cloudflare
      - dns01:
          cloudflare:
            email: contact@ibtisam-iq.com  # Cloudflare account email
            apiTokenSecretRef:
              name: cloudflare-api-token   # Secret created in Step 2
              key: api-token               # Key within the secret
        selector:
          dnsZones:
            - ibtisam-iq.com               # Only for this domain

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      hostname: java-monolith.ibtisam-iq.com
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      hostname: java-monolith.ibtisam-iq.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: java-monolith-tls
            kind: Secret
            group: ""
      allowedRoutes:
        namespaces:
          from: Same


Step 4: Apply Configuration
----------------------------

kubectl apply -k k8s/overlays/bare-metal/


Step 5: Watch Certificate Issuance
-----------------------------------

# Monitor certificate status (takes 1-3 minutes)
kubectl get certificate -n bankapp -w

# Check detailed status
kubectl describe certificate java-monolith-tls -n bankapp

# Check DNS-01 challenge
kubectl get challenges -n bankapp
kubectl describe challenge -n bankapp

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager -f


Step 6: Verify DNS TXT Record Created
--------------------------------------

During challenge, cert-manager creates a TXT record:

dig TXT _acme-challenge.java-monolith.ibtisam-iq.com

# Or check Cloudflare DNS dashboard
# You should see: _acme-challenge.java-monolith subdomain with TXT record

This record is automatically created and deleted by cert-manager.


Step 7: Verify Certificate Issued
----------------------------------

kubectl get certificate -n bankapp

Expected output:
NAME                READY   SECRET              AGE
java-monolith-tls   True    java-monolith-tls   2m

kubectl get secret java-monolith-tls -n bankapp -o yaml

# Should show type: kubernetes.io/tls with tls.crt and tls.key


Step 8: Test HTTPS
------------------

# Local test (with /etc/hosts or DNS)
curl -I https://java-monolith.ibtisam-iq.com:31872/ -k

# Should now work (port 31872 is HTTPS NodePort)

# Check certificate details
openssl s_client -connect java-monolith.ibtisam-iq.com:31872   -servername java-monolith.ibtisam-iq.com < /dev/null 2>/dev/null |   openssl x509 -noout -issuer -subject -dates

Expected:
  issuer=C = US, O = Let's Encrypt, CN = R10
  subject=CN = java-monolith.ibtisam-iq.com
  notBefore=Apr 26 22:00:00 2026 GMT
  notAfter=Jul 25 22:00:00 2026 GMT


Troubleshooting DNS-01:
------------------------

Issue: Challenge stuck in "pending"

  Check:
  kubectl describe challenge -n bankapp

  Common causes:
  - Wrong API token (check secret)
  - Token lacks Zone:Edit permission
  - Wrong DNS zone in selector.dnsZones
  - DNS provider rate limiting

Issue: "dns: lookup _acme-challenge.java-monolith.ibtisam-iq.com: no such host"

  Cause: DNS propagation delay (usually < 60s)
  Action: Wait 1-2 minutes, cert-manager retries automatically

Issue: "acme: error code 400: DNS problem: NXDOMAIN looking up TXT"

  Cause: TXT record not visible to Let's Encrypt's DNS resolvers
  Check: dig @8.8.8.8 TXT _acme-challenge.java-monolith.ibtisam-iq.com
  Action: Verify DNS propagation globally


Alternative DNS Providers:
---------------------------

AWS Route53:
  solvers:
    - dns01:
        route53:
          region: us-east-1
          accessKeyIDSecretRef:
            name: route53-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: route53-credentials
            key: secret-access-key

Google Cloud DNS:
  solvers:
    - dns01:
        cloudDNS:
          project: my-gcp-project
          serviceAccountSecretRef:
            name: clouddns-credentials
            key: key.json

DigitalOcean:
  solvers:
    - dns01:
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns-token
            key: access-token


================================================================================
APPROACH 2: CLOUDFLARE TUNNEL (RECOMMENDED FOR NAT BYPASS)
================================================================================

Cloudflare Tunnel Overview:
  - Cloudflare handles TLS termination at the edge
  - Tunnel forwards decrypted HTTP to your cluster
  - No cert-manager needed for public-facing HTTPS
  - No inbound ports needed (works behind NAT)
  - Free tier: unlimited bandwidth, no certificate management

Architecture:

  Browser (Internet)
       ↓ HTTPS
  Cloudflare Edge (TLS termination)
       ↓ HTTP (encrypted tunnel)
  cloudflared connector (in cluster or on node)
       ↓ HTTP
  Gateway/Ingress → Service → Pod

Public users see HTTPS (Cloudflare's cert), but inside the tunnel it's HTTP.


Step 1: Install cloudflared
----------------------------

Option A: On Control Plane Node (Recommended for testing)

# Download cloudflared binary
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64

# Move to system binary path
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared

# Verify installation
cloudflared --version


Option B: In Kubernetes (Production approach)

# We'll cover this in Step 8 (after manual setup works)


Step 2: Authenticate with Cloudflare
-------------------------------------

cloudflared tunnel login

This opens a browser window:
  1. Log in to Cloudflare account
  2. Select domain: ibtisam-iq.com
  3. Authorize cloudflared

Result: Creates credentials file at ~/.cloudflared/cert.pem


Step 3: Create Tunnel
----------------------

cloudflared tunnel create bankapp

Output:
  Tunnel credentials written to /root/.cloudflared/<tunnel-id>.json
  Created tunnel bankapp with id: f6d8a9b2-1234-5678-90ab-cdef12345678

Note the tunnel ID: f6d8a9b2-1234-5678-90ab-cdef12345678


Step 4: Configure Tunnel Routing
---------------------------------

Create config file: ~/.cloudflared/config.yml

tunnel: f6d8a9b2-1234-5678-90ab-cdef12345678
credentials-file: /root/.cloudflared/f6d8a9b2-1234-5678-90ab-cdef12345678.json

ingress:
  # Route bankapp.ibtisam-iq.com to Gateway HTTP listener
  - hostname: bankapp.ibtisam-iq.com
    service: http://172.16.0.2:32262
    originRequest:
      httpHostHeader: java-monolith.ibtisam-iq.com
      noTLSVerify: true

  # Catch-all rule (required)
  - service: http_status:404

Important fields:
  - hostname: Public domain users access
  - service: Internal endpoint (Gateway NodePort on HTTP)
  - httpHostHeader: Preserves Host header for Gateway routing
  - noTLSVerify: Accepts self-signed certs (if upstream is HTTPS)


Step 5: Create DNS CNAME
-------------------------

Cloudflare Dashboard:
  1. Go to DNS → Records
  2. Add record:

     Type: CNAME
     Name: bankapp
     Target: f6d8a9b2-1234-5678-90ab-cdef12345678.cfargotunnel.com
     Proxy: Yes (Orange Cloud - proxied through Cloudflare)
     TTL: Auto

Or via cloudflared CLI:

cloudflared tunnel route dns bankapp bankapp.ibtisam-iq.com


Step 6: Run Tunnel
-------------------

# Foreground (testing)
cloudflared tunnel run bankapp

# Background (daemon)
cloudflared tunnel run bankapp &

# As systemd service (production)
sudo cloudflared service install
sudo systemctl enable --now cloudflared


Step 7: Test Public HTTPS Access
---------------------------------

# From anywhere on the internet
curl -I https://bankapp.ibtisam-iq.com

Expected:
  HTTP/2 302
  location: https://bankapp.ibtisam-iq.com/login
  server: cloudflare

Traffic flow:
  Browser → Cloudflare Edge (HTTPS with Cloudflare's cert) →
  → Tunnel → 172.16.0.2:32262 (HTTP) →
  → Gateway → Service → Pod


Step 8: Deploy cloudflared in Kubernetes (Production)
------------------------------------------------------

Create Kubernetes Secret with tunnel credentials:

kubectl create secret generic cloudflared-credentials   --from-file=credentials.json=/root/.cloudflared/f6d8a9b2-1234-5678-90ab-cdef12345678.json   -n bankapp

Create ConfigMap with tunnel config:

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: bankapp
data:
  config.yaml: |
    tunnel: f6d8a9b2-1234-5678-90ab-cdef12345678
    credentials-file: /etc/cloudflared/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true

    ingress:
      - hostname: bankapp.ibtisam-iq.com
        service: http://bankapp-gateway-nginx.bankapp.svc.cluster.local:80
        originRequest:
          httpHostHeader: java-monolith.ibtisam-iq.com
      - service: http_status:404
EOF

Deploy cloudflared Deployment:

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: bankapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config.yaml
            - run
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared
              readOnly: true
            - name: credentials
              mountPath: /etc/cloudflared/credentials.json
              subPath: credentials.json
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: cloudflared-config
            items:
              - key: config.yaml
                path: config.yaml
        - name: credentials
          secret:
            secretName: cloudflared-credentials
EOF

Verify deployment:

kubectl get pods -n bankapp -l app=cloudflared
kubectl logs -n bankapp -l app=cloudflared


Step 9: HTTPS is Now Live
--------------------------

Public access (from anywhere):
  https://bankapp.ibtisam-iq.com

Certificate: Cloudflare Universal SSL (free, auto-managed)
  - Valid for *.ibtisam-iq.com
  - Auto-renewed by Cloudflare
  - No cert-manager needed

Internal access (within cluster/network):
  http://172.16.0.2:32262
  http://java-monolith.ibtisam-iq.com:32262


Cloudflare Tunnel Features:
----------------------------

Zero Trust Access:
  - Add authentication (Google, GitHub, email OTP) via Cloudflare Access
  - No VPN needed for remote team access

DDoS Protection:
  - Built-in (Cloudflare edge absorbs attacks)

WAF (Web Application Firewall):
  - Enable in Cloudflare Dashboard → Security

Analytics:
  - Real-time traffic analytics in Cloudflare Dashboard


Troubleshooting Cloudflare Tunnel:
-----------------------------------

Issue: 502 Bad Gateway

  Check tunnel status:
  cloudflared tunnel info bankapp

  Check cloudflared logs:
  journalctl -u cloudflared -f  # if systemd service
  kubectl logs -n bankapp -l app=cloudflared  # if in k8s

  Common causes:
  - Origin service unreachable (wrong IP/port)
  - Host header mismatch (missing httpHostHeader)
  - Service not running

Issue: Connection timed out

  Cause: cloudflared not running

  Check:
  ps aux | grep cloudflared
  systemctl status cloudflared
  kubectl get pods -n bankapp -l app=cloudflared

Issue: DNS not resolving

  Verify CNAME exists:
  dig CNAME bankapp.ibtisam-iq.com

  Should return: <tunnel-id>.cfargotunnel.com


================================================================================
APPROACH 3: SELF-SIGNED CERTIFICATE (TESTING ONLY)
================================================================================

Self-Signed Certificate Overview:
  - Generated locally, not trusted by browsers
  - No external dependencies (no cert-manager, no Let's Encrypt)
  - Fast setup (< 5 minutes)
  - Only for development/testing (browsers show security warning)

Use Cases:
  - Local testing of HTTPS configuration
  - Internal tools (add cert to system trust store)
  - CI/CD pipelines
  - Learning Gateway API TLS


Step 1: Generate Self-Signed Certificate
-----------------------------------------

Using OpenSSL:

# Create private key
openssl genrsa -out tls.key 2048

# Create certificate signing request (CSR)
openssl req -new -key tls.key -out tls.csr -subj "/CN=java-monolith.ibtisam-iq.com/O=Bankapp/C=PK"

# Generate self-signed certificate (valid 365 days)
openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt

# Verify certificate
openssl x509 -in tls.crt -text -noout

Expected output:
  Subject: CN = java-monolith.ibtisam-iq.com, O = Bankapp, C = PK
  Issuer: CN = java-monolith.ibtisam-iq.com, O = Bankapp, C = PK
  Validity:
    Not Before: Apr 27 00:00:00 2026 GMT
    Not After : Apr 27 00:00:00 2027 GMT


Step 2: Create Kubernetes Secret
---------------------------------

kubectl create secret tls java-monolith-tls   --cert=tls.crt   --key=tls.key   -n bankapp

Verify secret:

kubectl get secret java-monolith-tls -n bankapp

kubectl describe secret java-monolith-tls -n bankapp

Expected:
  Type: kubernetes.io/tls
  Data:
    tls.crt: 1234 bytes
    tls.key: 1675 bytes


Step 3: Update Gateway (No cert-manager annotation)
----------------------------------------------------

Edit k8s/overlays/bare-metal/gateway-cert.yaml:

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
  # Remove cert-manager annotation (not needed for self-signed)
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      hostname: java-monolith.ibtisam-iq.com
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      hostname: java-monolith.ibtisam-iq.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: java-monolith-tls  # References secret created in Step 2
            kind: Secret
            group: ""
      allowedRoutes:
        namespaces:
          from: Same


Step 4: Apply Gateway
----------------------

kubectl apply -f k8s/overlays/bare-metal/gateway-cert.yaml

Gateway should immediately become ready (no waiting for cert issuance):

kubectl get gateway -n bankapp

NAME              CLASS   ADDRESS          PROGRAMMED   AGE
bankapp-gateway   nginx   10.110.232.141   True         10s


Step 5: Test HTTPS
------------------

# From within network (with /etc/hosts or DNS)
curl -I https://java-monolith.ibtisam-iq.com:31872/ -k

  -k flag ignores certificate validation (required for self-signed)

Expected:
  HTTP/1.1 302
  location: https://java-monolith.ibtisam-iq.com/login

Without -k flag:
  curl: (60) SSL certificate problem: self-signed certificate


Step 6: Inspect Certificate
----------------------------

openssl s_client -connect java-monolith.ibtisam-iq.com:31872   -servername java-monolith.ibtisam-iq.com < /dev/null 2>/dev/null |   openssl x509 -noout -text

Output shows:
  Issuer: CN = java-monolith.ibtisam-iq.com, O = Bankapp, C = PK
  Subject: CN = java-monolith.ibtisam-iq.com, O = Bankapp, C = PK

Note: Issuer == Subject (self-signed indicator)


Step 7: Trust Certificate (Optional - for browser access)
----------------------------------------------------------

Option A: Trust in Browser (per-session)

  1. Open https://java-monolith.ibtisam-iq.com:31872/ in browser
  2. Click "Advanced" on security warning
  3. Click "Proceed to java-monolith.ibtisam-iq.com (unsafe)"

  This trusts the cert for current session only.


Option B: Add to System Trust Store (permanent)

  Linux (Ubuntu/Debian):

  sudo cp tls.crt /usr/local/share/ca-certificates/java-monolith.crt
  sudo update-ca-certificates

  macOS:

  sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain tls.crt

  Windows:

  certutil -addstore -f "ROOT" tls.crt


Step 8: Automate Certificate Rotation (Optional)
-------------------------------------------------

Self-signed certs expire. Automate renewal with a CronJob:

cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renew-self-signed-cert
  namespace: bankapp
spec:
  schedule: "0 0 1 * *"  # Monthly on 1st at midnight
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: renew-cert
              image: alpine/openssl:latest
              command:
                - /bin/sh
                - -c
                - |
                  openssl genrsa -out /tmp/tls.key 2048
                  openssl req -new -key /tmp/tls.key -out /tmp/tls.csr                     -subj "/CN=java-monolith.ibtisam-iq.com/O=Bankapp/C=PK"
                  openssl x509 -req -days 365 -in /tmp/tls.csr                     -signkey /tmp/tls.key -out /tmp/tls.crt
                  kubectl create secret tls java-monolith-tls                     --cert=/tmp/tls.crt --key=/tmp/tls.key                     -n bankapp --dry-run=client -o yaml | kubectl apply -f -
                  kubectl rollout restart deployment bankapp-gateway-nginx -n bankapp
          restartPolicy: OnFailure
          serviceAccountName: cert-renewer  # Needs RBAC for secret update
EOF

Create RBAC for CronJob:

kubectl create serviceaccount cert-renewer -n bankapp

kubectl create role cert-manager   --verb=get,create,update,patch   --resource=secrets   -n bankapp

kubectl create rolebinding cert-renewer-binding   --role=cert-manager   --serviceaccount=bankapp:cert-renewer   -n bankapp


Self-Signed vs. Real Certificate:
----------------------------------

┌────────────────────┬─────────────────┬────────────────────┐
│ Feature            │ Self-Signed     │ Let's Encrypt      │
├────────────────────┼─────────────────┼────────────────────┤
│ Browser Trust      │ No (warning)    │ Yes (trusted)      │
│ Setup Time         │ 5 minutes       │ 1-3 min (DNS-01)   │
│ External Deps      │ None            │ cert-manager       │
│ Public Internet    │ Not required    │ Required (DNS-01)  │
│ Expiry             │ 1 year (manual) │ 90 days (auto)     │
│ Cost               │ Free            │ Free               │
│ Use Case           │ Internal/test   │ Production         │
└────────────────────┴─────────────────┴────────────────────┘


================================================================================
COMPARISON: WHICH APPROACH TO USE?
================================================================================

┌───────────────┬──────────┬────────────┬─────────────┬─────────────────┐
│ Criteria      │ DNS-01   │ CF Tunnel  │ Self-Signed │ Recommendation  │
├───────────────┼──────────┼────────────┼─────────────┼─────────────────┤
│ Browser Trust │ Yes (LE) │ Yes (CF)   │ No          │ DNS-01/CF       │
│ NAT-friendly  │ Yes      │ Yes        │ Yes         │ All work        │
│ Setup Time    │ 15 min   │ 10 min     │ 5 min       │ Self-signed     │
│ Auto-renewal  │ Yes      │ N/A (CF)   │ Manual/Cron │ DNS-01/CF       │
│ External Deps │ DNS API  │ CF account │ None        │ Self-signed     │
│ Public Access │ Possible │ Built-in   │ No          │ CF Tunnel       │
│ Production    │ Yes      │ Yes        │ No          │ DNS-01/CF       │
│ Learning      │ Medium   │ Low        │ Best        │ Self-signed     │
└───────────────┴──────────┴────────────┴─────────────┴─────────────────┘


Recommended Path by Use Case:
------------------------------

1. Learning / Local Testing:
   → Self-Signed Certificate
   Fastest setup, no external dependencies.

2. Internal Access Only (team VPN):
   → Self-Signed + System Trust Store
   Add cert to all team machines' trust stores.

3. Public Internet Access:
   → Cloudflare Tunnel
   Simplest for NAT bypass, free, includes DDoS protection.

4. Production with Kubernetes-native TLS:
   → DNS-01 Challenge
   Real Let's Encrypt cert, auto-renewal, works behind NAT.

5. Multi-cluster / Edge Deployment:
   → Cloudflare Tunnel
   Single control plane for all clusters, centralized security.


================================================================================
DECISION MATRIX FOR YOUR SPECIFIC SETUP
================================================================================

Your Environment:
  - Bare-metal Kubernetes (kubeadm)
  - Private IP: 172.16.0.2 (behind NAT)
  - Domain: ibtisam-iq.com (managed in Cloudflare)
  - Current: Gateway API with HTTP working
  - Goal: Add HTTPS

Immediate Recommendation:

  START with Self-Signed Certificate (30 minutes total)
    - Validates Gateway TLS configuration works
    - Tests HTTPRoute with HTTPS
    - No external dependencies
    - Quick feedback loop

  THEN choose production approach:

    Option A: Cloudflare Tunnel (recommended)
      Pros:
        - Already using Cloudflare for DNS
        - Free tier sufficient
        - Built-in DDoS protection
        - No cert-manager complexity
      Cons:
        - External dependency on Cloudflare
        - Tunnel must stay connected

    Option B: DNS-01 Challenge
      Pros:
        - Kubernetes-native (cert-manager)
        - No tunnel dependency
        - Full control over TLS
      Cons:
        - Requires Cloudflare API token management
        - More components to maintain
        - Still need public access solution (Cloudflare Tunnel or VPN)


Step-by-Step Path for Your Project:
------------------------------------

Phase 1: Validation (Today)
  1. Generate self-signed cert (Approach 3, Step 1-2)
  2. Update Gateway with HTTPS listener (Approach 3, Step 3-4)
  3. Test HTTPS locally (Approach 3, Step 5-6)
  4. Confirm Gateway API TLS works end-to-end

Phase 2: Production Decision (Tomorrow)

  If you need public access:
    → Deploy Cloudflare Tunnel (Approach 2)
    → Keep self-signed cert for internal or switch to DNS-01

  If only internal access:
    → Keep self-signed cert
    → Add to team machines' trust stores


Phase 3: Production Deployment (Week 1)

  Cloudflare Tunnel path:
    1. Create tunnel (Approach 2, Step 1-3)
    2. Configure routing (Approach 2, Step 4)
    3. Deploy in Kubernetes (Approach 2, Step 8)
    4. Public HTTPS live

  DNS-01 path:
    1. Get Cloudflare API token (Approach 1, Step 1)
    2. Create secret (Approach 1, Step 2)
    3. Update ClusterIssuer (Approach 1, Step 3)
    4. Watch certificate issue (Approach 1, Step 5-7)
    5. Still need Cloudflare Tunnel for public access


================================================================================
NEXT STEPS
================================================================================

Recommended Immediate Action:

1. Test self-signed certificate (fastest validation):

   openssl genrsa -out tls.key 2048
   openssl req -new -key tls.key -out tls.csr -subj "/CN=java-monolith.ibtisam-iq.com"
   openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt
   kubectl create secret tls java-monolith-tls --cert=tls.crt --key=tls.key -n bankapp

   # Update Gateway (remove cert-manager annotation)
   kubectl apply -f k8s/overlays/bare-metal/gateway-cert.yaml

   # Test
   curl -I https://java-monolith.ibtisam-iq.com:31872/ -k

2. If HTTPS works, choose production path:

   For public access: Cloudflare Tunnel (Approach 2)
   For Kubernetes-native TLS: DNS-01 (Approach 1)

3. Document choice in project README with rationale.


All three approaches are production-ready for their intended use cases.
Self-signed is ONLY for testing; DNS-01 and Cloudflare Tunnel are both
valid production solutions for bare-metal behind NAT.

================================================================================
END OF GUIDE
================================================================================

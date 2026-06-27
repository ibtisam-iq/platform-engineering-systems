# Kubernetes Ingress & Gateway API: Complete Networking Guide
# Documentation for bankapp deployment
# Author: Muhammad Ibtisam Iqbal
# Last Updated: 2026-04-27

================================================================================
TABLE OF CONTENTS
================================================================================

1. Core Concepts
   1.1 HTTP Host-Based Routing
   1.2 DNS vs. Host Header Routing
   1.3 ClusterIP vs. NodePort vs. LoadBalancer

2. Ingress Architecture
   2.1 How Ingress Works
   2.2 Ingress Controller Deployment Models
   2.3 Service Exposure Patterns

3. Gateway API Architecture
   3.1 How Gateway API Works
   3.2 Gateway vs. Ingress
   3.3 Resource Model

4. Environment-Specific Implementations
   4.1 Bare-Metal Clusters
   4.2 Cloud Providers (AWS/GCP/Azure)
   4.3 Hybrid (Bare-Metal + Cloudflare Tunnel)

5. IP Address Assignment
   5.1 Where IPs Come From
   5.2 DNS A Record Configuration
   5.3 Testing Without DNS

6. Practical Examples
   6.1 Bare-Metal Ingress Setup
   6.2 Cloud Ingress Setup
   6.3 Gateway API Setup
   6.4 Cloudflare Tunnel Integration

7. Troubleshooting
   7.1 Common Issues
   7.2 Debugging Commands
   7.3 Verification Checklist

================================================================================
1. CORE CONCEPTS
================================================================================

1.1 HTTP HOST-BASED ROUTING
----------------------------

Kubernetes Ingress and Gateway API both route HTTP traffic based on the
**Host header** in HTTP requests, NOT based on DNS records.

Request Flow:
  1. Client resolves hostname via DNS → gets IP address
  2. Client sends HTTP request to that IP
  3. Request includes Host header (e.g., "Host: java-monolith.ibtisam-iq.com")
  4. Ingress/Gateway reads Host header
  5. Routes to backend based on configured host matching rules

Example HTTP Request:
  GET / HTTP/1.1
  Host: java-monolith.ibtisam-iq.com
  User-Agent: curl/8.1.0
  Accept: */*

The "Host: java-monolith.ibtisam-iq.com" line is what Ingress/Gateway matches.


1.2 DNS VS. HOST HEADER ROUTING
--------------------------------

DNS and Host-based routing are TWO SEPARATE STEPS:

┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: DNS RESOLUTION (happens on client side)                 │
│                                                                 │
│ Client asks: "What IP is java-monolith.ibtisam-iq.com?"         │
│ DNS responds: "172.16.0.2"                                      │
│                                                                 │
│ Purpose: Tells client WHERE to send the HTTP request            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: HTTP HOST HEADER ROUTING (happens in cluster)           │
│                                                                 │
│ Request arrives at 172.16.0.2 with "Host: java-monolith..."     │
│ Ingress/Gateway reads Host header                               │
│ Routes to correct backend service                             │
│                                                                 │
│ Purpose: Tells Ingress/Gateway WHICH backend to forward to      │
└─────────────────────────────────────────────────────────────────┘

Key Point:
  - DNS gets the request TO the cluster
  - Host header routes the request WITHIN the cluster
  - Ingress/Gateway does NOT perform DNS lookups
  - DNS A record and Ingress host field are independent


1.3 CLUSTERIP VS. NODEPORT VS. LOADBALANCER
--------------------------------------------

Kubernetes Services expose applications using different types:

┌──────────────┬──────────────┬─────────────────┬───────────────────┐
│ Service Type │ Accessible   │ IP Type         │ Use Case          │
├──────────────┼──────────────┼─────────────────┼───────────────────┤
│ ClusterIP    │ Cluster only │ Internal        │ Internal services │
│              │              │ (10.96.0.0/12)  │ (databases, APIs) │
├──────────────┼──────────────┼─────────────────┼───────────────────┤
│ NodePort     │ External via │ Node IP + port  │ Bare-metal        │
│              │ node port    │ (30000-32767)   │ dev/test clusters │
├──────────────┼──────────────┼─────────────────┼───────────────────┤
│ LoadBalancer │ External via │ Cloud-assigned  │ Cloud production  │
│              │ cloud LB     │ public IP       │ workloads         │
└──────────────┴──────────────┴─────────────────┴───────────────────┘

Example Service Outputs:

ClusterIP (internal only):
  NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
  bankapp-service   ClusterIP   10.96.100.50    <none>        80/TCP

NodePort (bare-metal):
  NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
  bankapp-service   NodePort    10.96.100.50    <none>        80:30080/TCP

LoadBalancer (cloud):
  NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
  bankapp-service   LoadBalancer   10.96.100.50    203.0.113.45    80:30080/TCP


================================================================================
2. INGRESS ARCHITECTURE
================================================================================

2.1 HOW INGRESS WORKS
---------------------

Ingress provides HTTP/HTTPS routing to services based on hostnames and paths.

Components:
  1. Ingress Resource (YAML manifest defining routing rules)
  2. Ingress Controller (implementation that watches Ingress resources)
  3. Service (backend that receives routed traffic)

Traffic Flow:

  Internet/Client
       ↓
  [DNS Resolution: java-monolith.ibtisam-iq.com → 172.16.0.2]
       ↓
  Node IP:NodePort (172.16.0.2:31844)
       ↓
  Ingress Controller (ingress-nginx pod)
       ↓
  [Host Header Matching: java-monolith.ibtisam-iq.com → bankapp-service]
       ↓
  Service ClusterIP (bankapp-service:80)
       ↓
  Pod (bankapp container:8000)


Example Ingress Resource:

  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: bankapp-ingress
    namespace: bankapp
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

What happens:
  - Requests with "Host: java-monolith.ibtisam-iq.com" → bankapp-service:80
  - Requests with different Host header → 404 (no matching rule)


2.2 INGRESS CONTROLLER DEPLOYMENT MODELS
-----------------------------------------

Ingress Controller can be exposed via different Service types:

A. NodePort (Bare-Metal Default):

   kubectl get svc -n ingress-nginx
   NAME                       TYPE       PORT(S)
   ingress-nginx-controller   NodePort   80:31844/TCP,443:31845/TCP

   Access: http://<node-ip>:31844
   DNS A Record: java-monolith.ibtisam-iq.com → <node-ip>


B. LoadBalancer (Cloud):

   kubectl get svc -n ingress-nginx
   NAME                       TYPE           EXTERNAL-IP      PORT(S)
   ingress-nginx-controller   LoadBalancer   203.0.113.45     80:31844/TCP

   Access: http://203.0.113.45
   DNS A Record: java-monolith.ibtisam-iq.com → 203.0.113.45


C. HostNetwork (Advanced Bare-Metal):

   Ingress controller pods bind directly to node ports 80/443.
   No NodePort service needed.
   Access: http://<node-ip> (port 80)
   Requires: No other process on node using ports 80/443


2.3 SERVICE EXPOSURE PATTERNS
------------------------------

Multiple ways to expose applications:

Pattern 1: Direct NodePort (No Ingress)

  Browser → Node:30080 → Service:80 → Pod:8000

  Pros: Simple, no ingress controller needed
  Cons: Each service needs unique NodePort, no host-based routing


Pattern 2: Ingress + NodePort Controller (Bare-Metal)

  Browser → Node:31844 (ingress-nginx) → Ingress routes by host →
  → Service:80 → Pod:8000

  Pros: Host/path-based routing, single NodePort for all apps
  Cons: Requires NodePort for ingress controller


Pattern 3: Ingress + LoadBalancer (Cloud)

  Browser → Cloud LB:80 → Ingress → Service:80 → Pod:8000

  Pros: Standard port 80/443, auto-provisioned public IP
  Cons: Cloud-only, costs money


Pattern 4: Cloudflare Tunnel (NAT Bypass)

  Browser → Cloudflare Edge → cloudflared connector →
  → Node:31844 (or 30080) → Service → Pod

  Pros: Works behind NAT, no public IP needed, built-in DDoS protection
  Cons: Requires Cloudflare account, external dependency


================================================================================
3. GATEWAY API ARCHITECTURE
================================================================================

3.1 HOW GATEWAY API WORKS
--------------------------

Gateway API is the successor to Ingress, providing more expressive routing.

Key Resources:
  - GatewayClass: Controller implementation (like IngressClass)
  - Gateway: Load balancer configuration (listeners, addresses)
  - HTTPRoute: Routing rules (like Ingress rules)

Traffic Flow:

  Internet/Client
       ↓
  [DNS Resolution: java-monolith.ibtisam-iq.com → 172.16.0.2]
       ↓
  Gateway Listener (port 80/443)
       ↓
  [Host Header Matching via HTTPRoute]
       ↓
  Service (bankapp-service:80)
       ↓
  Pod (bankapp:8000)


Example Gateway + HTTPRoute:

  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: bankapp-gateway
    namespace: bankapp
  spec:
    gatewayClassName: nginx
    listeners:
      - name: http
        protocol: HTTP
        port: 80

  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: bankapp-route
    namespace: bankapp
  spec:
    parentRefs:
      - name: bankapp-gateway
    hostnames:
      - java-monolith.ibtisam-iq.com
    rules:
      - backendRefs:
          - name: bankapp-service
            port: 80


3.2 GATEWAY VS. INGRESS
------------------------

┌──────────────────┬─────────────────────┬──────────────────────┐
│ Feature          │ Ingress             │ Gateway API          │
├──────────────────┼─────────────────────┼──────────────────────┤
│ Maturity         │ Stable (v1)         │ Stable (v1 in 2023)  │
│ Host Routing     │ Yes                 │ Yes                  │
│ Path Routing     │ Yes                 │ Yes                  │
│ Header Matching  │ Via annotations     │ Native support       │
│ Traffic Split    │ Via annotations     │ Native support       │
│ Cross-Namespace  │ No                  │ Yes (ReferenceGrant) │
│ TCP/UDP          │ No                  │ Yes (TCPRoute/UDP)   │
│ Resource Model   │ Single resource     │ Gateway + Routes     │
│ Role Separation  │ Limited             │ Strong (infra/app)   │
└──────────────────┴─────────────────────┴──────────────────────┘

When to use Gateway API:
  - Advanced routing (header-based, weighted traffic splits)
  - Multi-team clusters (namespace isolation)
  - TCP/UDP workloads
  - Future-proofing (Gateway API is the future)

When to use Ingress:
  - Simple HTTP routing
  - Established tooling/CI integrations
  - Team familiarity with Ingress


3.3 RESOURCE MODEL
------------------

Gateway API separates infrastructure and application concerns:

Infrastructure Owner (Platform Team):
  - Manages GatewayClass
  - Deploys Gateway (listeners, TLS)
  - Controls IP addresses

Application Developer:
  - Manages HTTPRoute (routing rules)
  - References Gateway via parentRefs
  - No access to Gateway config

Example:

  # Infrastructure team deploys once
  Gateway: bankapp-gateway (listens on port 80)

  # Each app team creates their own route
  HTTPRoute: app1-route → parentRef: bankapp-gateway
  HTTPRoute: app2-route → parentRef: bankapp-gateway
  HTTPRoute: app3-route → parentRef: bankapp-gateway


================================================================================
4. ENVIRONMENT-SPECIFIC IMPLEMENTATIONS
================================================================================

4.1 BARE-METAL CLUSTERS
-----------------------

Characteristics:
  - No cloud provider
  - No automatic LoadBalancer provisioning
  - Services exposed via NodePort or MetalLB

Ingress Controller Installation (NodePort):

  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

Result:
  - Ingress controller runs as Deployment
  - Service type: NodePort (ports 80:31XXX, 443:31YYY)
  - Access via node IP + NodePort

Get NodePort:
  kubectl get svc -n ingress-nginx ingress-nginx-controller

  Output:
  NAME                       TYPE       PORT(S)
  ingress-nginx-controller   NodePort   80:31844/TCP,443:31845/TCP

Access:
  http://<node-ip>:31844  (for Ingress routing)

Get Node IP:
  hostname -I          # On control plane node
  kubectl get nodes -o wide

DNS Configuration:
  Option A: Real DNS (if node has public IP)
    A record: java-monolith.ibtisam-iq.com → <node-public-ip>

  Option B: /etc/hosts (local testing)
    172.16.0.2  java-monolith.ibtisam-iq.com

  Option C: Cloudflare Tunnel (NAT bypass)
    No DNS needed; Cloudflare handles routing

Testing:
  # With /etc/hosts or DNS configured
  curl http://java-monolith.ibtisam-iq.com:31844/

  # Without DNS (manual Host header)
  curl -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/


4.2 CLOUD PROVIDERS (AWS/GCP/AZURE)
------------------------------------

Characteristics:
  - Cloud controller provisions LoadBalancer automatically
  - Public IP assigned by cloud provider
  - Standard ports 80/443 (no NodePort needed)

Ingress Controller Installation (AWS):

  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml

Result:
  - Ingress controller runs as Deployment
  - Service type: LoadBalancer
  - Cloud creates Network Load Balancer (NLB) or ALB
  - Public IP assigned automatically

Get External IP:
  kubectl get svc -n ingress-nginx ingress-nginx-controller

  Output:
  NAME                       TYPE           EXTERNAL-IP      PORT(S)
  ingress-nginx-controller   LoadBalancer   203.0.113.45     80:31844/TCP

  EXTERNAL-IP (203.0.113.45) is the cloud-assigned public IP.

DNS Configuration:
  Cloudflare/Route53:
    Type: A
    Name: java-monolith
    Value: 203.0.113.45  (the EXTERNAL-IP from above)
    TTL: 300

Access:
  http://java-monolith.ibtisam-iq.com  (resolves to 203.0.113.45)

  No port number needed (uses standard port 80/443).

Testing:
  curl http://java-monolith.ibtisam-iq.com
  curl https://java-monolith.ibtisam-iq.com  (with TLS configured)


Cloud-Specific Notes:

AWS:
  - Uses Network Load Balancer (NLB) by default
  - Can use ALB with AWS Load Balancer Controller
  - NLB preserves source IP
  - Costs: ~$16-20/month per NLB

GCP:
  - Uses Cloud Load Balancing (L4 or L7)
  - Global IP addresses supported
  - Integrated with Cloud Armor (DDoS protection)

Azure:
  - Uses Azure Load Balancer
  - Can integrate with Application Gateway
  - Supports static public IPs


4.3 HYBRID (BARE-METAL + CLOUDFLARE TUNNEL)
--------------------------------------------

Use case: Bare-metal cluster behind NAT needs external access.

Architecture:

  Internet → Cloudflare Edge → Cloudflare Tunnel (outbound connection) →
  → cloudflared connector (in cluster) → Ingress/Service → Pod

Advantages:
  - No public IP required on cluster
  - Works behind NAT/firewall
  - Built-in DDoS protection
  - Free tier available

Setup:

1. Install cloudflared connector in cluster
2. Create Cloudflare Tunnel in dashboard
3. Configure tunnel origin service

Tunnel Origin Configuration (for Ingress):

  service: http://172.16.0.2:31844
  originRequest:
    httpHostHeader: java-monolith.ibtisam-iq.com

Tunnel Origin Configuration (for direct NodePort):

  service: http://172.16.0.2:30080

Key Point:
  - Origin must be reachable from cloudflared pod
  - httpHostHeader preserves Host header for Ingress routing
  - Port must match NodePort of target service

DNS:
  Cloudflare automatically creates:
    CNAME: java-monolith.ibtisam-iq.com → <tunnel-id>.cfargotunnel.com

  No manual A record needed.

Access:
  https://java-monolith.ibtisam-iq.com

  Resolves to Cloudflare edge, routed through tunnel.


================================================================================
5. IP ADDRESS ASSIGNMENT
================================================================================

5.1 WHERE IPS COME FROM
-----------------------

The IP you use for DNS A records depends on your environment:

┌─────────────────────┬───────────────────┬──────────────────────┐
│ Environment         │ IP Source         │ How to Get IP        │
├─────────────────────┼───────────────────┼──────────────────────┤
│ Bare-Metal NodePort │ Node IP           │ hostname -I          │
│                     │                   │ kubectl get nodes -o │
├─────────────────────┼───────────────────┼──────────────────────┤
│ Bare-Metal MetalLB  │ MetalLB IP pool   │ kubectl get svc      │
│                     │                   │ (EXTERNAL-IP column) │
├─────────────────────┼───────────────────┼──────────────────────┤
│ AWS/GCP/Azure       │ Cloud LB          │ kubectl get svc      │
│                     │                   │ (EXTERNAL-IP column) │
├─────────────────────┼───────────────────┼──────────────────────┤
│ Cloudflare Tunnel   │ No IP needed      │ CNAME auto-created   │
└─────────────────────┴───────────────────┴──────────────────────┘


Detailed Examples:

A. Bare-Metal NodePort:

  $ hostname -I
  172.16.0.2 10.244.0.0 10.244.0.1

  Use: 172.16.0.2 (first IP, node's primary interface)

  $ kubectl get svc -n ingress-nginx
  NAME                       TYPE       PORT(S)
  ingress-nginx-controller   NodePort   80:31844/TCP

  DNS A Record: java-monolith.ibtisam-iq.com → 172.16.0.2
  Access: http://java-monolith.ibtisam-iq.com:31844


B. Bare-Metal MetalLB:

  $ kubectl get svc -n ingress-nginx
  NAME                       TYPE           EXTERNAL-IP    PORT(S)
  ingress-nginx-controller   LoadBalancer   172.16.0.10    80:31844/TCP

  Use: 172.16.0.10 (assigned from MetalLB IP pool)

  DNS A Record: java-monolith.ibtisam-iq.com → 172.16.0.10
  Access: http://java-monolith.ibtisam-iq.com (port 80)


C. Cloud (AWS):

  $ kubectl get svc -n ingress-nginx
  NAME                       TYPE           EXTERNAL-IP      PORT(S)
  ingress-nginx-controller   LoadBalancer   203.0.113.45     80:31844/TCP

  Use: 203.0.113.45 (cloud-assigned public IP)

  DNS A Record: java-monolith.ibtisam-iq.com → 203.0.113.45
  Access: http://java-monolith.ibtisam-iq.com


D. Cloudflare Tunnel:

  No manual IP/DNS needed.

  Cloudflare creates:
    CNAME: java-monolith.ibtisam-iq.com → <tunnel-id>.cfargotunnel.com

  Access: https://java-monolith.ibtisam-iq.com


5.2 DNS A RECORD CONFIGURATION
-------------------------------

General Process:

1. Get the IP address (see section 5.1)
2. Log in to your DNS provider (Cloudflare, Route53, etc.)
3. Add A record

Example (Cloudflare Dashboard):

  Type: A
  Name: java-monolith  (becomes java-monolith.ibtisam-iq.com)
  IPv4 address: 172.16.0.2  (or cloud LB IP)
  Proxy status: DNS only (grey cloud)  ← CRITICAL for cert-manager
  TTL: Auto

Verify DNS:

  $ nslookup java-monolith.ibtisam-iq.com
  Server:   1.1.1.1
  Address:  1.1.1.1#53

  Non-authoritative answer:
  Name:     java-monolith.ibtisam-iq.com
  Address:  172.16.0.2

  $ dig +short java-monolith.ibtisam-iq.com
  172.16.0.2


Important Notes:

- Cloudflare Proxy (Orange Cloud):
  Enables CDN/DDoS protection but breaks cert-manager HTTP-01 challenges.
  Must be DISABLED (grey cloud) during certificate issuance.
  Can enable after cert issued (cert-manager renews via DNS-01 or keeps proxy off).

- Private IPs (RFC 1918):
  172.16.0.0/12, 192.168.0.0/16, 10.0.0.0/8 are private.
  DNS will resolve them, but they're only reachable within the network.
  Use Cloudflare Tunnel for external access with private IPs.


5.3 TESTING WITHOUT DNS
------------------------

You can test Ingress/HTTPRoute without configuring DNS.

Method 1: /etc/hosts (Local Machine Only)

  Edit /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows):

  172.16.0.2  java-monolith.ibtisam-iq.com

  Save and test:
  curl http://java-monolith.ibtisam-iq.com:31844/

  This ONLY works on the machine where you edited the hosts file.


Method 2: Manual Host Header (Universal)

  curl -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/

  This works from any machine that can reach 172.16.0.2.

  Bypasses DNS entirely by manually setting the Host header.


Method 3: HTTPie (Developer-Friendly)

  http http://172.16.0.2:31844/ Host:java-monolith.ibtisam-iq.com


Method 4: Browser (Dev Tools Override)

  1. Open browser Dev Tools (F12)
  2. Go to Network → Network conditions
  3. Enable "Override headers"
  4. Add custom header: Host: java-monolith.ibtisam-iq.com
  5. Navigate to http://172.16.0.2:31844/


Why Manual Host Header Works:

  The Ingress/HTTPRoute only cares about the HTTP Host header.
  DNS just tells your client what IP to connect to.

  By manually setting the Host header, you skip DNS and directly
  tell Ingress/HTTPRoute which backend to route to.


================================================================================
6. PRACTICAL EXAMPLES
================================================================================

6.1 BARE-METAL INGRESS SETUP
-----------------------------

Environment:
  - kubeadm cluster on 172.16.0.2
  - No public IP (behind NAT)
  - Using NodePort for ingress-nginx

Step-by-Step:

1. Install ingress-nginx:

   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

   kubectl get pods -n ingress-nginx
   # Wait until ingress-nginx-controller pod is Running


2. Get NodePort:

   kubectl get svc -n ingress-nginx ingress-nginx-controller

   Output:
   NAME                       TYPE       PORT(S)
   ingress-nginx-controller   NodePort   80:31844/TCP,443:31845/TCP

   NodePort for HTTP: 31844


3. Deploy application:

   kubectl apply -f k8s/base/
   kubectl apply -f k8s/overlays/bare-metal/


4. Create Ingress:

   # k8s/overlays/bare-metal/ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: bankapp-ingress
     namespace: bankapp
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
       nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
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

   kubectl apply -f k8s/overlays/bare-metal/ingress.yaml


5. Configure local DNS (testing only):

   echo "172.16.0.2  java-monolith.ibtisam-iq.com" | sudo tee -a /etc/hosts


6. Test:

   curl http://java-monolith.ibtisam-iq.com:31844/

   Expected: HTTP 302 redirect to /login (Spring Security)


7. Verify Ingress routing:

   # Should work (correct Host header)
   curl -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/

   # Should fail with 404 (wrong Host header)
   curl -H "Host: wrong-domain.com" http://172.16.0.2:31844/


6.2 CLOUD INGRESS SETUP
-----------------------

Environment:
  - AWS EKS cluster
  - Cloud provisions LoadBalancer automatically

Step-by-Step:

1. Install ingress-nginx:

   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml

   kubectl get svc -n ingress-nginx -w
   # Wait until EXTERNAL-IP shows (takes 2-5 minutes)


2. Get LoadBalancer IP:

   kubectl get svc -n ingress-nginx ingress-nginx-controller

   Output:
   NAME                       TYPE           EXTERNAL-IP      PORT(S)
   ingress-nginx-controller   LoadBalancer   203.0.113.45     80:31844/TCP

   Note EXTERNAL-IP: 203.0.113.45


3. Configure DNS:

   Cloudflare/Route53:
     Type: A
     Name: java-monolith
     Value: 203.0.113.45
     TTL: 300


4. Deploy application and Ingress (same as bare-metal, no NodePort needed)


5. Test:

   curl http://java-monolith.ibtisam-iq.com

   No port number needed (uses standard port 80).


6. Add TLS:

   Install cert-manager:
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml

   Deploy Ingress with TLS (see ingress-cert.yaml in section 6.4)


6.3 GATEWAY API SETUP
---------------------

Environment:
  - Bare-metal or cloud
  - Using Gateway API instead of Ingress

Step-by-Step:

1. Install Gateway API CRDs:

   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml


2. Install Gateway Controller (e.g., nginx-gateway):

   kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.2.0/deploy.yaml


3. Create GatewayClass:

   apiVersion: gateway.networking.k8s.io/v1
   kind: GatewayClass
   metadata:
     name: nginx
   spec:
     controllerName: nginx.org/gateway-controller


4. Create Gateway:

   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
     name: bankapp-gateway
     namespace: bankapp
   spec:
     gatewayClassName: nginx
     listeners:
       - name: http
         protocol: HTTP
         port: 80


5. Get Gateway Address:

   kubectl get gateway bankapp-gateway -n bankapp

   NAME              CLASS   ADDRESS        PROGRAMMED   AGE
   bankapp-gateway   nginx   172.16.0.10    True         5m

   Use ADDRESS (172.16.0.10) for DNS A record.


6. Create HTTPRoute:

   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: bankapp-route
     namespace: bankapp
   spec:
     parentRefs:
       - name: bankapp-gateway
     hostnames:
       - java-monolith.ibtisam-iq.com
     rules:
       - backendRefs:
           - name: bankapp-service
             port: 80


7. Test (same as Ingress):

   curl http://java-monolith.ibtisam-iq.com


6.4 CLOUDFLARE TUNNEL INTEGRATION
----------------------------------

Environment:
  - Bare-metal cluster behind NAT (172.16.0.2)
  - Need external access without public IP

Setup:

1. Install cloudflared in cluster or on node:

   # Download cloudflared
   wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
   sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
   sudo chmod +x /usr/local/bin/cloudflared


2. Authenticate with Cloudflare:

   cloudflared tunnel login
   # Opens browser to log in


3. Create tunnel:

   cloudflared tunnel create bankapp
   # Returns tunnel ID: e.g., f6d8a9b2-1234-5678-90ab-cdef12345678


4. Configure tunnel (config.yml):

   tunnel: f6d8a9b2-1234-5678-90ab-cdef12345678
   credentials-file: /root/.cloudflared/f6d8a9b2-1234-5678-90ab-cdef12345678.json

   ingress:
     - hostname: bankapp.ibtisam-iq.com
       service: http://172.16.0.2:31844
       originRequest:
         httpHostHeader: java-monolith.ibtisam-iq.com
     - service: http_status:404


5. Create DNS entry (Cloudflare dashboard):

   Type: CNAME
   Name: bankapp
   Target: f6d8a9b2-1234-5678-90ab-cdef12345678.cfargotunnel.com
   Proxy: Yes (Orange Cloud)


6. Run tunnel:

   cloudflared tunnel run bankapp


7. Test:

   curl https://bankapp.ibtisam-iq.com

   Traffic flows:
   Browser → Cloudflare Edge → Tunnel → 172.16.0.2:31844 → Ingress → Pod


Key Configuration:

- service: http://172.16.0.2:31844
  Must be reachable from where cloudflared runs.

- httpHostHeader: java-monolith.ibtisam-iq.com
  Preserves Host header so Ingress can route correctly.

  Without this, Host header would be "bankapp.ibtisam-iq.com",
  which doesn't match Ingress rule for "java-monolith.ibtisam-iq.com".


Alternative: Direct to NodePort (bypass Ingress)

  ingress:
    - hostname: bankapp.ibtisam-iq.com
      service: http://172.16.0.2:30080
    - service: http_status:404

This goes directly to application NodePort, skipping Ingress entirely.


================================================================================
7. TROUBLESHOOTING
================================================================================

7.1 COMMON ISSUES
-----------------

Issue 1: 404 Not Found from Ingress

Symptoms:
  curl http://java-monolith.ibtisam-iq.com:31844/
  → 404 page not found (nginx default page)

Causes:
  A. Host header doesn't match Ingress rule
  B. Ingress not created or not in correct namespace
  C. IngressClass mismatch

Debug:
  # Verify Ingress exists
  kubectl get ingress -n bankapp

  # Check Ingress details
  kubectl describe ingress bankapp-ingress -n bankapp

  # Test with explicit Host header
  curl -v -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/

  # Check ingress-nginx logs
  kubectl logs -n ingress-nginx deploy/ingress-nginx-controller


Issue 2: 502 Bad Gateway

Symptoms:
  curl http://java-monolith.ibtisam-iq.com:31844/
  → 502 Bad Gateway

Causes:
  A. Backend service not running
  B. Service port mismatch
  C. Pod not ready

Debug:
  # Check service exists
  kubectl get svc -n bankapp

  # Verify service endpoints
  kubectl get endpoints bankapp-service -n bankapp
  # Should show pod IPs

  # Check pod status
  kubectl get pods -n bankapp

  # Test service directly (from within cluster)
  kubectl run -it --rm debug --image=curlimages/curl --restart=Never --     curl http://bankapp-service.bankapp.svc.cluster.local


Issue 3: DNS Not Resolving

Symptoms:
  curl http://java-monolith.ibtisam-iq.com
  → Could not resolve host

Causes:
  A. DNS A record not created
  B. DNS not propagated yet
  C. Wrong DNS server

Debug:
  # Check DNS resolution
  nslookup java-monolith.ibtisam-iq.com
  dig +short java-monolith.ibtisam-iq.com

  # Force specific DNS server
  nslookup java-monolith.ibtisam-iq.com 1.1.1.1

  # Bypass DNS with /etc/hosts
  echo "172.16.0.2  java-monolith.ibtisam-iq.com" | sudo tee -a /etc/hosts


Issue 4: Cloudflare Tunnel 502 Error

Symptoms:
  curl https://bankapp.ibtisam-iq.com
  → Cloudflare 502 Bad Gateway page

Causes:
  A. Tunnel origin unreachable
  B. Wrong origin port
  C. Host header mismatch

Debug:
  # Check cloudflared logs
  journalctl -u cloudflared -f

  # Test origin from tunnel host
  curl http://172.16.0.2:31844/

  # Test with correct Host header
  curl -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/

  # Verify tunnel config
  cat ~/.cloudflared/config.yml


Issue 5: Certificate Not Issued (cert-manager)

Symptoms:
  kubectl get certificate -n bankapp
  → READY = False

Causes:
  A. Cloudflare proxy enabled (HTTP-01 challenge fails)
  B. Port 80 not accessible from internet
  C. Ingress missing cert-manager annotation

Debug:
  # Check certificate status
  kubectl describe certificate java-monolith-tls -n bankapp

  # Check certificate request
  kubectl get certificaterequest -n bankapp
  kubectl describe certificaterequest -n bankapp

  # Check challenges
  kubectl get challenges -n bankapp
  kubectl describe challenge -n bankapp

  # Check cert-manager logs
  kubectl logs -n cert-manager deploy/cert-manager

  # Verify Let's Encrypt can reach cluster
  curl http://java-monolith.ibtisam-iq.com/.well-known/acme-challenge/test


7.2 DEBUGGING COMMANDS
----------------------

Ingress:
  kubectl get ingress -n bankapp
  kubectl describe ingress bankapp-ingress -n bankapp
  kubectl get ingressclass
  kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

Services:
  kubectl get svc -n bankapp
  kubectl get endpoints bankapp-service -n bankapp
  kubectl describe svc bankapp-service -n bankapp

Pods:
  kubectl get pods -n bankapp
  kubectl logs <pod-name> -n bankapp
  kubectl describe pod <pod-name> -n bankapp

Gateway API:
  kubectl get gateway -n bankapp
  kubectl describe gateway bankapp-gateway -n bankapp
  kubectl get httproute -n bankapp
  kubectl describe httproute bankapp-route -n bankapp

DNS:
  nslookup java-monolith.ibtisam-iq.com
  dig +short java-monolith.ibtisam-iq.com
  host java-monolith.ibtisam-iq.com

Connectivity:
  curl -v http://172.16.0.2:31844/
  curl -v -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/
  telnet 172.16.0.2 31844

Certificate:
  kubectl get certificate -n bankapp
  kubectl describe certificate java-monolith-tls -n bankapp
  kubectl get secret java-monolith-tls -n bankapp
  openssl s_client -connect java-monolith.ibtisam-iq.com:443 -servername java-monolith.ibtisam-iq.com


7.3 VERIFICATION CHECKLIST
---------------------------

Before declaring "it works":

□ Ingress resource exists in correct namespace
□ Ingress shows ADDRESS in kubectl get ingress
□ Service exists and has endpoints
□ Pods are Running and Ready
□ DNS resolves to correct IP
□ Port is reachable (telnet <ip> <port>)
□ curl with explicit Host header returns expected response
□ curl with DNS hostname returns expected response
□ TLS certificate issued (if using HTTPS)
□ Certificate is valid (check with openssl s_client)
□ Cloudflare tunnel shows "healthy" status (if applicable)


Full Test Sequence:

# 1. Check Kubernetes resources
kubectl get ingress,svc,pods -n bankapp

# 2. Verify DNS
nslookup java-monolith.ibtisam-iq.com

# 3. Test connectivity (bypass DNS)
curl -v -H "Host: java-monolith.ibtisam-iq.com" http://172.16.0.2:31844/

# 4. Test with DNS
curl -v http://java-monolith.ibtisam-iq.com:31844/

# 5. Test HTTPS (if configured)
curl -v https://java-monolith.ibtisam-iq.com

# 6. Check TLS certificate
openssl s_client -connect java-monolith.ibtisam-iq.com:443 -servername java-monolith.ibtisam-iq.com < /dev/null


================================================================================
END OF DOCUMENTATION
================================================================================

This documentation covers Kubernetes Ingress and Gateway API networking
comprehensively, from core concepts to production deployment.

For project-specific implementation, refer to:
  - k8s/overlays/bare-metal/ingress.yaml
  - k8s/overlays/bare-metal/ingress-cert.yaml
  - k8s/overlays/bare-metal/patch-service-nodeport.yaml

Maintained by: Muhammad Ibtisam Iqbal (github.com/ibtisam-iq)
Last updated: 2026-04-27

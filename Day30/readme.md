# TLS/SSL Certificates in Kubernetes: Complete Mastery Guide

This is a multi-layer topic. I'll go from first principles → Kubernetes PKI internals → workload certificate management → mTLS → production failure modes → the bleeding edge. Buckle up.

## PART 1 — THE WHY: The Problem TLS Solves

**Before TLS: What Was Broken?**

In a Kubernetes cluster, you have dozens of components talking to each other over a network:

```
kubectl → API Server → etcd
                     → kubelet (on every node)
                     → scheduler
                     → controller-manager
kubelet → API Server (reverse direction too)
etcd node A → etcd node B → etcd node C (peer replication)
```

Without TLS on every one of these channels:

**1. Eavesdropping:** Anyone on the network path reads all API traffic. kubectl get secrets leaks your secrets in plaintext. An attacker who compromises any node can read all pod specs, Secrets, ConfigMaps flowing through the API server.

**2. Impersonation (No Server Identity):** A compromised node can spin up a fake API server. Kubelets on other nodes have no way to verify they're talking to the real API server — they'd happily execute pod spec commands from the attacker.

**3. No Client Identity:** The API server can't verify who is making the request. Any process that can reach port 6443 could issue commands. No AuthN → no AuthZ → no RBAC.

**4. No Integrity:** An on-path attacker can modify in-flight API responses. kubectl apply sends your Deployment, attacker intercepts and modifies the container image to point to their registry.

This is why every single communication channel in Kubernetes is TLS-protected and certificate-authenticated by default. Kubernetes is not just using TLS for web traffic — it uses it as its fundamental identity and encryption layer across the entire control plane.

## PART 2 — DEEP-DIVE MECHANICS

**Layer 0: How TLS Actually Works (The Foundation You Must Know)**

**The TLS 1.3 Handshake:**

```
Client                                     Server
  |                                            |
  |--- ClientHello (TLS version, ciphers, ---->|
  |    client_random, key_share)               |
  |                                            |
  |<-- ServerHello (chosen cipher, -----------|
  |    server_random, key_share)               |
  |<-- Certificate (server's X.509 cert) -----|
  |<-- CertificateVerify (signed with priv key)|
  |<-- Finished (MAC over transcript) ---------|
  |                                            |
  |--- Finished (MAC over transcript) -------->|
  |                                            |
  |====== Encrypted Application Data =========|
```

Key insight: The session key is never transmitted. Both sides independently derive the same symmetric key using ECDHE (Elliptic Curve Diffie-Hellman Ephemeral). This gives you Perfect Forward Secrecy (PFS) — even if the server's private key is later compromised, past sessions can't be decrypted because the ephemeral keys were discarded.

**X.509 Certificate Anatomy — Every Field Matters:**

```
Certificate:
  Data:
    Version: 3
    Serial Number: 3a:f2:...
    Signature Algorithm: sha256WithRSAEncryption
    Issuer: CN=kubernetes, O=kubernetes  ← who signed this
    Validity:
      Not Before: Jan 1 2025
      Not After:  Jan 1 2026
    Subject: CN=kube-apiserver           ← who owns this cert
    Subject Public Key Info:
      Public Key Algorithm: id-ecPublicKey
      Public Key: (256 bit EC key)
    X509v3 Extensions:
      X509v3 Key Usage: critical
        Digital Signature, Key Encipherment
      X509v3 Extended Key Usage:
        TLS Web Server Authentication    ← server cert
        TLS Web Client Authentication    ← also client cert
      X509v3 Subject Alternative Name:  ← CRITICAL FIELD
        DNS:kubernetes
        DNS:kubernetes.default
        DNS:kubernetes.default.svc
        DNS:kubernetes.default.svc.cluster.local
        IP Address:10.96.0.1            ← service cluster IP
        IP Address:172.31.10.5          ← node IP
        IP Address:54.x.x.x             ← load balancer IP
  Signature: ...signed by Issuer's private key...
```

**The SAN (Subject Alternative Name) is the most common failure point in Kubernetes.** Modern TLS clients (Go 1.15+, all browsers, openssl) completely ignore the CN field for hostname verification. If your API server's IP or DNS name is not in the SANs, every connection fails with x509: certificate is valid for X, not Y.

**Layer 1: The Kubernetes Internal PKI**

Kubernetes runs **three independent Certificate Authorities**. This is intentional — blast radius isolation.

**kubeadm PKI directory structure:**

```
/etc/kubernetes/pki/
├── ca.crt                          # Root CA #1: Kubernetes CA
├── ca.key                          # !! PROTECT THIS WITH YOUR LIFE !!
│
├── apiserver.crt                   # API server TLS serving cert
├── apiserver.key
│
├── apiserver-kubelet-client.crt    # API server → kubelet (client cert)
├── apiserver-kubelet-client.key
│
├── apiserver-etcd-client.crt       # API server → etcd (client cert)
├── apiserver-etcd-client.key
│
├── front-proxy-ca.crt              # Root CA #2: API Aggregation CA
├── front-proxy-ca.key
│
├── front-proxy-client.crt          # For aggregated API servers (metrics-server, etc.)
├── front-proxy-client.key
│
├── sa.pub                          # Service Account key pair (NOT a cert)
├── sa.key                          # Used to sign/verify SA tokens (JWT)
│
└── etcd/
    ├── ca.crt                      # Root CA #3: etcd CA
    ├── ca.key
    ├── server.crt                  # etcd server TLS cert
    ├── server.key
    ├── peer.crt                    # etcd peer-to-peer replication cert
    ├── peer.key
    ├── healthcheck-client.crt      # Used by liveness probes
    └── healthcheck-client.key
```

**Why three separate CAs?**

<img width="905" height="283" alt="image" src="https://github.com/user-attachments/assets/949974f8-f42a-4cfe-9444-a5865b649c5c" />

Separating etcd CA is critical. If the main Kubernetes CA were also used for etcd, a compromised scheduler cert could potentially be escalated toward etcd access. Separation enforces that only the apiserver-etcd-client cert (signed by etcd CA) can talk to etcd.

**Certificate → Identity Mapping Table:**

<img width="922" height="527" alt="image" src="https://github.com/user-attachments/assets/6c9dc343-7f67-49e4-8d76-6b40f3882b83" />

This table reveals something important: **the CN and O fields in an X.509 client cert ARE the username and groups in Kubernetes**. If you generate a cert with CN=evil-user, O=system:masters, that cert gives cluster-admin access. This is why you must protect your CA private key with extreme care.

**Layer 2: Kubeconfig Files — Certs in Disguise**

Every kubeconfig file is just a structured bundle of TLS credentials:

```
apiVersion: v1
kind: Config
clusters:
- name: my-cluster
  cluster:
    certificate-authority-data: <base64 of ca.crt>  # To verify server
    server: https://api.example.com:6443
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <base64 of client.crt>  # Client identity
    client-key-data: <base64 of client.key>           # Client private key
contexts:
- name: kubernetes-admin@my-cluster
  context:
    cluster: my-cluster
    user: kubernetes-admin
```

When kubectl makes an API call:

- Loads kubeconfig, finds context
- Establishes TLS connection, presents client.crt to API server
- API server verifies client.crt was signed by ca.crt
- Extracts CN → username, O → groups
- RBAC evaluates: can kubernetes-admin (member of system:masters) do GET /api/v1/pods?
- system:masters is bound to cluster-admin ClusterRole → allowed

**Layer 3: TLS Bootstrapping — How Worker Nodes Get Their Certs**

A worker node joining the cluster faces a chicken-and-egg problem: it needs a cert to talk to the API server, but it needs to talk to the API server to get a cert.

**The solution: Bootstrap tokens + CSR API**

```
New Node                    API Server              Controller-Manager
    |                           |                          |
    |-- HTTPS with bootstrap --->|                          |
    |   token (not a cert yet)  |                          |
    |                           |                          |
    |-- Submit CSR (kubelet- --->|                          |
    |   bootstrap.kubeconfig)   |-- Create CSR object ----->|
    |                           |                          |
    |                           |<-- Auto-approve CSR ------|
    |                           |   (NodeBootstrapper role) |
    |                           |                          |
    |<-- Signed kubelet cert ----|                          |
    |                           |                          |
    | Now uses real cert for all subsequent calls           |
```

**Bootstrap token format:** <6-char-token-id>.<16-char-secret> e.g. abcdef.0123456789abcdef

This is encoded in /etc/kubernetes/bootstrap-kubelet.conf on the node (generated during kubeadm join).

**Kubelet certificate rotation:**

Once bootstrapped, the kubelet continuously rotates its own cert:

- Feature gates RotateKubeletClientCertificate and RotateKubeletServerCertificate (both default-on since K8s 1.19+)
- At 70-80% of cert lifetime, kubelet submits a new CSR
- Controller-manager auto-approves kubelet client CSRs
- Kubelet serving cert CSRs require manual approval by default (security consideration — a compromised node shouldn't be able to issue itself a serving cert)

**Layer 4: The Kubernetes CSR API**

You can use Kubernetes as a CA for user-facing certs:

```
# Step 1: Generate private key and CSR
openssl genrsa -out chetan.key 2048
openssl req -new -key chetan.key \
  -subj "/CN=chetan/O=platform-team" \
  -out chetan.csr

# Step 2: Submit CSR to Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: chetan
spec:
  request: $(base64 -w 0 chetan.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400   # 24 hours
  usages:
  - client auth
EOF

# Step 3: Approve it
kubectl certificate approve chetan

# Step 4: Fetch signed cert
kubectl get csr chetan -o jsonpath='{.status.certificate}' | base64 -d > chetan.crt

# Step 5: Build kubeconfig
kubectl config set-credentials chetan \
  --client-certificate=chetan.crt \
  --client-key=chetan.key
```

**Signer names and what they mean:**

<img width="902" height="400" alt="image" src="https://github.com/user-attachments/assets/24c19c0f-1a27-4d31-a000-5f2f34b98962" />

**Layer 5: TLS Secrets and Workload Certificates**

**The kubernetes.io/tls Secret type:**

```
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64>   # Full chain: leaf cert + intermediate CAs (NOT root)
  tls.key: <base64>   # Private key (PEM format)
```

**Critical detail on tls.crt content**: It must contain the complete certificate chain: your leaf cert concatenated with intermediate CA cert(s). Why not the root? Because clients already have the root CA in their trust store (or in ca.crt for internal CAs). Including the root wastes bytes and some validators reject it.

```
# Correct tls.crt content order:
-----BEGIN CERTIFICATE-----
<leaf cert>
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
<intermediate CA cert>
-----END CERTIFICATE-----
```

**Mounting TLS certs into pods:**

```
spec:
  volumes:
  - name: tls
    secret:
      secretName: myapp-tls
  containers:
  - name: app
    volumeMounts:
    - name: tls
      mountPath: /etc/tls
      readOnly: true
    env:
    - name: TLS_CERT_FILE
      value: /etc/tls/tls.crt
    - name: TLS_KEY_FILE
      value: /etc/tls/tls.key
```

When the Secret is updated (cert renewal), volume-mounted files update automatically (eventually, within ~1 minute via kubelet sync period). Environment variables do NOT — pods must be restarted. This is why volume mounts are strongly preferred for TLS.

**Layer 6: cert-manager — The Full Mechanics**

cert-manager is a Kubernetes controller (not just a tool) that extends Kubernetes with certificate lifecycle management. It is the de facto standard.

**CRD hierarchy:**

```
ClusterIssuer / Issuer
        │
        │ (cert-manager creates)
        ▼
   Certificate
        │
        │ (cert-manager creates)
        ▼
CertificateRequest
        │
        │ (for ACME issuers, cert-manager creates)
        ▼
     Order
        │
        │ (cert-manager creates one per domain)
        ▼
   Challenge
        │
        │ (solved via HTTP-01 or DNS-01)
        ▼
   ACME CA signs → cert stored in Secret
```

**cert-manager controller loop internals:**

cert-manager runs several controllers:

- certificate controller: watches Certificate objects, triggers CertificateRequest creation
- certificaterequest controller: routes to the right issuer
- acme/order controller: manages ACME orders
- acme/challenge controller: creates/verifies challenges
- ingress-shim controller: watches Ingress objects for cert-manager.io/cluster-issuer annotation → auto-creates Certificate objects

**Full ClusterIssuer + Certificate example for a production setup:**

```
# ClusterIssuer with both HTTP-01 and DNS-01 solvers
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key  # ACME account private key
    solvers:
    - selector:                           # HTTP-01 for non-wildcard
        dnsZones:
        - "example.com"
      http01:
        ingress:
          class: nginx
          podTemplate:
            spec:
              nodeSelector:
                kubernetes.io/os: linux
    - selector:                           # DNS-01 for wildcards
        matchLabels:
          cert-manager.io/dns-solver: route53
      dns01:
        route53:
          region: us-east-1
          hostedZoneID: Z1234567890
          accessKeyIDSecretRef:
            name: route53-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: route53-credentials
            key: secret-access-key
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  duration: 2160h       # 90 days (Let's Encrypt default)
  renewBefore: 720h     # Start renewal 30 days before expiry
  subject:
    organizations:
    - "My Company"
  privateKey:
    algorithm: ECDSA    # Prefer ECDSA over RSA: smaller, faster
    size: 256           # P-256 curve
    rotationPolicy: Always  # Generate new key on every renewal
  usages:
  - server auth
  dnsNames:
  - myapp.example.com
  - www.myapp.example.com
  - "*.api.example.com"  # Wildcard — requires DNS-01 solver
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    group: cert-manager.io
```

**HTTP-01 Challenge deep-dive:**

```
cert-manager                 Kubernetes               Let's Encrypt
     |                           |                         |
     |-- Create Ingress rule --->|                         |
     |   /.well-known/acme-      |                         |
     |   challenge/<token>       |                         |
     |                           |                         |
     |-- Notify LE: ready -------------------------------->|
     |                           |                         |
     |                           |<-- GET /.well-known/... |
     |                           |   (from LE's servers)   |
     |                           |-- Route to cert-manager |
     |                           |   solver pod ---------->|
     |                           |                   returns token
     |                           |<---------------------------------|
     |<-- Order finalized ----------------------------------|
     |-- Delete Ingress rule --->|
     |-- Store cert in Secret -->|
```

**DNS-01 Challenge deep-dive:**

```
cert-manager            Route53 / DNS Provider       Let's Encrypt
     |                           |                         |
     |-- CreateResourceRecordSet>|                         |
     |   _acme-challenge.        |                         |
     |   myapp.example.com TXT   |                         |
     |   "<token>"               |                         |
     |                           |                         |
     |-- Wait for DNS propagation (can take 60-300 seconds)|
     |                           |                         |
     |-- Notify LE: ready -------------------------------->|
     |                           |                         |
     |                           |<-- DNS TXT lookup -------|
     |                           |   "_acme-challenge..."   |
     |<-- Order finalized ----------------------------------|
     |-- Delete TXT record ----->|
     |-- Store cert in Secret -->|
```

**cert-manager internal CA for cluster-internal services:**

```
# Self-signed issuer to bootstrap a CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
# The CA Certificate itself
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "internal-ca"
  secretName: internal-ca-keypair
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
---
# CA Issuer that uses the CA cert above
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-keypair  # References the CA cert+key
---
# Now any service can get an internal cert
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payments-service-cert
  namespace: production
spec:
  secretName: payments-tls
  dnsNames:
  - payments.production.svc.cluster.local
  - payments.production.svc
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
```

**Layer 7: Ingress TLS vs Gateway API TLS**

**Ingress TLS (classic):**

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls         # cert-manager creates this
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

**Gateway API TLS (modern replacement):**

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod  # cert-manager supports Gateway API too
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.example.com"
    tls:
      mode: Terminate            # Terminate TLS at gateway; backend gets HTTP
      certificateRefs:
      - name: wildcard-tls
        kind: Secret
    allowedRoutes:
      namespaces:
        from: All
---
# TLS Passthrough (backend terminates TLS)
  - name: tls-passthrough
    port: 8443
    protocol: TLS
    tls:
      mode: Passthrough          # SNI routing; backend handles TLS
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp-route
  namespace: production
spec:
  parentRefs:
  - name: prod-gateway
    namespace: production
    sectionName: https
  hostnames:
  - myapp.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: myapp
      port: 80
```

**Gateway API TLS advantages over Ingress:**

- Infrastructure (Gateway) owned by platform team; routes (HTTPRoute) owned by app teams → proper separation of concerns
- TLS Passthrough is a first-class concept (Ingress couldn't do this cleanly)
- Cross-namespace route references with ReferenceGrant (explicit security model)
- No annotation sprawl — everything is typed and spec-driven

**Layer 8: mTLS and Service Meshes**

**mTLS mechanics:**

In regular TLS (one-way), only the server presents a certificate. In mTLS:

```
Client                                     Server
  |--- ClientHello ------------------------>|
  |<-- ServerHello + Certificate ----------|
  |<-- CertificateRequest (!!mTLS only!!) -|  ← Server asks client to auth
  |--- Certificate (client's cert) ------->|  ← Client sends its cert
  |--- CertificateVerify (signed with -----> ← Proves it holds the private key
  |    client's private key)               |
  |<--- Finished (both sides verified) ----|
  |====== Encrypted + Mutually Auth'd =====|
```

**Why this matters for microservices**: Without mTLS, a network policy is your only defense. If an attacker gets a pod into your cluster, it can talk to any other pod within allowed network policy rules — and the receiving service has no way to verify the caller's identity. With mTLS, every call is cryptographically authenticated to a specific service identity.

**Istio mTLS architecture:**

```
Pod A                                      Pod B
┌─────────────────────┐         ┌─────────────────────┐
│  app container      │         │  app container      │
│  (talks HTTP :8080) │         │  (listens on :8080) │
│         │           │         │         ▲           │
│         ▼           │         │         │           │
│  envoy sidecar      │  mTLS   │  envoy sidecar      │
│  (:15001 outbound)  │────────>│  (:15006 inbound)   │
│                     │         │                     │
└─────────────────────┘         └─────────────────────┘
                                         │
                               cert from Istiod CA
                               SPIFFE ID: spiffe://cluster.local/
                                         ns/production/sa/myapp

Istiod (control plane):
- Watches K8s API for pod/SA changes
- Issues X.509 SVIDs to each Envoy via xDS (SDS - Secret Discovery Service)
- Rotates certs every 24 hours by default
```

**SPIFFE SVIDs in Istio:**
Each workload gets an identity like: spiffe://cluster.local/ns/production/sa/payments-service

This identity is embedded in the X.509 SAN as a URI SAN. When pod A calls pod B:

- Envoy in A presents cert with SPIFFE ID of pod A's service account
- Envoy in B verifies the cert, extracts SPIFFE ID
- Istio AuthorizationPolicy can then allow/deny based on SPIFFE ID:

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payments-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: payments-service
  rules:
  - from:
    - source:
        # Only allow calls from checkout service
        principals:
        - "cluster.local/ns/production/sa/checkout-service"
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/charge"]
```

**PeerAuthentication for mTLS mode:**

```
# Cluster-wide STRICT mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system   # istio-system = cluster-wide scope
spec:
  mtls:
    mode: STRICT
---
# Namespace-level override
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: legacy-namespace
spec:
  mtls:
    mode: PERMISSIVE   # Allows both mTLS and plaintext (migration period)
```

**Layer 9: Certificate Expiry Management and Monitoring**

**kubeadm cert management:**

```
# Check expiry of all control plane certs
kubeadm certs check-expiration

# Output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME  
# admin.conf                 Jan 01, 2026 00:00 UTC   364d
# apiserver                  Jan 01, 2026 00:00 UTC   364d
# etcd-healthcheck-client    Jan 01, 2026 00:00 UTC   364d
# ... etc

# Renew all certs
kubeadm certs renew all

# Renew specific cert
kubeadm certs renew apiserver

# After renewal, restart static pods to pick up new certs
# (static pods read certs from disk on startup)
kill -s SIGHUP $(pidof kube-apiserver)
# OR
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

**cert-manager monitoring metrics:**

```
# Key Prometheus metrics from cert-manager:
certmanager_certificate_expiry_seconds{name="...", namespace="...", issuer="..."}
certmanager_certificate_ready_status{...}
certmanager_http_acme_client_request_duration_seconds{...}

# Critical alert rule:
- alert: CertificateExpiringIn7Days
  expr: certmanager_certificate_expiry_seconds < (7 * 24 * 3600)
  labels:
    severity: critical
  annotations:
    summary: "Certificate {{ $labels.name }} in {{ $labels.namespace }} expires in < 7 days"
```

## PART 3 — ALTERNATIVE LANDSCAPE

**Certificate Management Tool Comparison**

<img width="912" height="487" alt="image" src="https://github.com/user-attachments/assets/49a5c234-8ec8-4754-ba20-9c5d63cadcf1" />

<img width="906" height="182" alt="image" src="https://github.com/user-attachments/assets/5403ed70-1170-4454-8e42-5825a9e27aa9" />

**TLS Termination Location Comparison**

<img width="912" height="452" alt="image" src="https://github.com/user-attachments/assets/15ee10af-202f-4135-8cc6-8979711b2c63" />

**Production recommendation:** AWS ALB (ACM) + Ingress for external traffic, service mesh mTLS for internal service-to-service. Defense in depth.

## PART 4 — INTERVIEW POV & EDGE CASES

How This Topic Is Asked in Interviews

**System design level:**

"Design the security architecture for a multi-tenant Kubernetes cluster. How do you ensure workloads in namespace A cannot impersonate workloads in namespace B?"

Answer path: Separate service accounts, RBAC, NetworkPolicy, and mTLS with service mesh (SPIFFE identity is namespace-scoped). cert-manager with per-namespace Issuers (not ClusterIssuers) for additional isolation.

**Scenario-based:**

"Your API server cert just expired in production. Walk me through your recovery."

Answer path:

- Determine severity — can existing authenticated connections still work? (Active TLS sessions survive, new connections fail)
- kubeadm certs renew apiserver on each control plane node
- Restart kube-apiserver static pod
- Update any kubeconfigs that embed the CA (they use the same CA, which didn't change, so they should still work)
- Verify: openssl s_client -connect <api-server>:6443 | openssl x509 -noout -dates
- Post-mortem: implement kubeadm certs check-expiration in monitoring pipeline

**Deep technical:**

"Why can't you use HTTP-01 ACME challenge to get a wildcard cert?"

Answer: HTTP-01 proves you control a specific FQDN by serving a token at http://<fqdn>/.well-known/acme-challenge/<token>. A wildcard cert *.example.com doesn't map to a specific FQDN — the ACME spec explicitly requires DNS-01 for wildcards, because DNS proves you control the entire zone, not just one hostname.

**The Gotchas (Senior Engineer Failure Points)**

**1. Missing SANs — The #1 Production Incident**

```
# Diagnose cert SAN mismatch:
openssl s_client -connect api.example.com:6443 2>/dev/null | \
  openssl x509 -noout -text | grep -A5 "Subject Alternative Name"

# Error you'll see in clients:
# x509: certificate is valid for kubernetes, kubernetes.default, ...,
# not 54.x.x.x
```

Fix: You must regenerate the API server cert with the correct SANs including all IPs, DNS names, and the LoadBalancer IP. With kubeadm:

```
kubeadm init phase certs apiserver \
  --apiserver-cert-extra-sans=54.x.x.x,api.mycluster.com
```

**2. Let's Encrypt Rate Limits**

5 failures per account per hostname per hour for HTTP-01/DNS-01. 50 certs per registered domain per week. If you delete and recreate Certificate objects rapidly in dev/staging, you'll hit this and be blocked for hours. Always use the staging ACME server for testing:

```
server: https://acme-staging-v02.api.letsencrypt.org/directory
```

**3. cert-manager Down = Certs Don't Renew**

cert-manager is a critical infrastructure component, not an optional addon. If it crashes and a cert expires, your services go down. Mitigations:

- Run cert-manager with multiple replicas (HA mode)
- Monitor certmanager_certificate_expiry_seconds
- Set renewBefore to 30+ days, not the default 15 days
- Alert at 30 days, page at 7 days

**4. kubeadm 1-Year Cert Expiry**

kubeadm certificates default to 1-year validity. Many clusters reach their first birthday and the control plane completely dies. Your API server stops accepting connections. Even kubectl get nodes fails. This is a silent killer — no warnings, just sudden failure.

kubeadm certs check-expiration should be in your monitoring pipeline, not just run manually.

**5. etcd Peer Cert Rotation Gone Wrong**

If you rotate etcd peer certs on all nodes simultaneously, new certs won't be trusted by nodes still running old certs → etcd can't form quorum → cluster is read-only or completely down. Always rotate one etcd member at a time, verify quorum health between each rotation:

```
etcdctl endpoint health --cluster
etcdctl member list
```

**6. The Ingress Annotation vs. Certificate Object Conflict**

If you create both a Certificate object with secretName: myapp-tls AND an Ingress with cert-manager.io/cluster-issuer annotation pointing to the same secret, cert-manager's ingress-shim will create a second Certificate object. Now two Certificate objects are fighting over the same Secret. One will win, one will error. Pick one approach per resource.

**7. Cross-Namespace Secret Reference is Blocked**

Ingress controllers can only reference TLS Secrets in the same namespace as the Ingress by default (nginx) or a designated set of namespaces. A ClusterIssuer-created cert in namespace production cannot be referenced by an Ingress in namespace staging. Solution: either create a cert per namespace, or use Gateway API's ReferenceGrant for explicit cross-namespace access.

**8. PERMISSIVE mTLS Mode Left Enabled**

```
mTLS PERMISSIVE = false sense of security
```

In PERMISSIVE mode, Istio accepts both mTLS and plaintext. This is intended for gradual migration. If you forget to move to STRICT, a non-meshed pod can call a meshed pod in plaintext. Your AuthorizationPolicies are unenforced because the source.principal is empty for non-mTLS connections. Always switch to STRICT as the final migration step.

**9. Private Key in etcd (Secret Security Model)**

By default, cert-manager generates the private key and stores it in a Kubernetes Secret, which is stored in etcd. etcd at-rest encryption must be enabled, or anyone with etcd access can read all private keys. This is why high-security environments use Vault CSI driver (keys generated and stored in Vault) or SPIFFE/SPIRE (keys are ephemeral, never persisted).

**10. Certificate Renewal Requires Pod Restart for Env Var Injection**

If a pod reads TLS_KEY_FILE from an environment variable (injected from a Secret), it will use the old cert value until it's restarted. Volume mounts update automatically. In production, prefer volume mounts and configure your application to watch for file changes (or use inotify/fsnotify in your app code).

## PART 5 — THE BETTER WAY (EVOLUTION)

**SPIFFE/SPIRE — Workload Identity Done Right**

**cert-manager limitation:** It manages certificates, but identity is tied to what name you put in the Certificate spec — a human-defined string. It's not cryptographically tied to the actual workload.

**SPIFFE** (Secure Production Identity Framework for Everyone) defines a standard:

- Every workload gets a globally unique identity: spiffe://<trust-domain>/ns/<ns>/sa/<sa>
- Identity is attested — the SPIRE agent verifies the workload via kernel-level attestation (Linux kernel, K8s API, cgroups, process ancestry)
- SVIDs (X.509 or JWT) are short-lived (hours, not days) and rotated automatically
- Even if a cert is stolen, it's worthless in ~1 hour

```
SPIRE Agent (on each node)             SPIRE Server
     |                                       |
     |-- Node attestation (BootstrapToken) ->|
     |<-- Node SVID -----------------------|
     |                                       |
     |-- Workload attestation (via K8s API)->|  ← Verifies pod/SA
     |<-- Workload SVID (X.509 cert) -------|
     |-- Serve SVID via Unix socket -------->|
     |   /run/spire/sockets/agent.sock       |  ← App or sidecar reads this
```

**Cilium mTLS — eBPF Native (No Sidecar)**

The future of service mesh is eBPF. Cilium with eBPF enforces mTLS at the kernel network layer:

- No Envoy sidecar = lower latency (sub-millisecond vs. ~1ms sidecar overhead)
- No iptables rules = faster packet processing
- Uses SPIFFE for identity
- Wireguard encryption for node-to-node traffic
- This is what Grafana Labs and Cloudflare use at scale

**Secrets Store CSI Driver + Vault PKI**

```
Pod startup → CSI driver fetches cert from Vault PKI → mounts as tmpfs volume
                                                        (private key never hits etcd)
```

Vault can generate certs with very short TTLs (minutes to hours) and the CSI driver rotates them automatically. The private key is generated inside Vault's HSM-backed memory — it is never extractable.

**Policy-as-Code for Certificate Governance (Kyverno)**

```
# Enforce that all certs in production use ECDSA and last < 90 days
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cert-policy
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-cert-duration
    match:
      resources:
        kinds: ["Certificate"]
        namespaces: ["production"]
    validate:
      message: "Certs must use ECDSA and duration <= 2160h"
      pattern:
        spec:
          duration: "<=2160h"
          privateKey:
            algorithm: ECDSA
```

**Master Reference: The TLS Decision Tree**

```
                     Do you need certs for...?
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
  Control Plane          External Traffic     Internal Service
  Components             (user-facing)        to Service
         │                    │                    │
  kubeadm certs          cert-manager +       Service Mesh
  (static, 1yr,          Let's Encrypt        (Istio/Linkerd/Cilium)
  rotate annually)       (ACME, 90d,          SPIFFE SVIDs
                         auto-renewal)        (24h, auto-rotation)
```

**Interview Cheat Sheet (Bookmark This)**

<img width="881" height="505" alt="image" src="https://github.com/user-attachments/assets/9df51964-7d62-400e-acf5-4502fc063a79" />

<img width="853" height="282" alt="image" src="https://github.com/user-attachments/assets/97a73cd7-e2eb-4d6e-835c-c88968586f68" />

**************************************************************************************************************************************************************

# Kubernetes TLS Certificate Masterclass

**The One Mental Model That Explains Everything**

Before a single technical term — burn this analogy into your brain. Every K8s certificate concept maps to it perfectly.

```
Real World                    Kubernetes World
─────────────────────────────────────────────────────
Government                =   Certificate Authority (CA)
Passport                  =   X.509 Certificate
Your face/biometrics      =   Private Key (proves YOU hold the cert)
Passport number/name      =   CN (Common Name) — your identity
Border officer            =   The component receiving the connection
Showing passport          =   TLS Handshake
Both sides show passport  =   mTLS (Mutual TLS)
Passport expiry date      =   Certificate NotAfter field
Passport book (full chain)=   Certificate chain (leaf + intermediate CA)
```

Every time you see "certificate" in Kubernetes, ask yourself: **who is showing their passport to whom, and who issued that passport?** That question answers 90% of all cert problems.

## Chapter 1: What Is a Certificate? (Crystal Clear)

A certificate is just **a text file with two jobs:**

**Job 1: Announce identity** — "I am the Kubernetes API server"
**Job 2: Carry a trustworthy signature** — "The CA vouch for this identity"

Without the signature, anyone could make a file saying "I am the API server." The CA's signature is what makes it trustworthy.

```
┌──────────────────────────────────────────────────┐
│              X.509 CERTIFICATE                    │
│                                                   │
│  WHO AM I?                                        │
│  Subject CN: kube-apiserver                       │
│  Subject O:  system:masters                       │
│                                                   │
│  WHAT CAN I BE USED FOR?                         │
│  Key Usage: TLS Server Auth, TLS Client Auth      │
│                                                   │
│  WHAT NAMES/IPs CAN I PROVE?                     │
│  SANs: kubernetes, 10.96.0.1, 172.31.5.2         │
│  ← THIS IS THE MOST CRITICAL FIELD               │
│                                                   │
│  WHEN AM I VALID?                                 │
│  Not Before: Jan 1 2025                           │
│  Not After:  Jan 1 2026                           │
│                                                   │
│  WHO VOUCHES FOR ME?                              │
│  Issuer: CN=kubernetes (the CA that signed this)  │
│                                                   │
│  SIGNATURE: [cryptographic proof by CA]           │
└──────────────────────────────────────────────────┘
```

**The SAN field is the #1 interview gotcha**. Since Go 1.15 (which Kubernetes is written in), the CN field is completely ignored for hostname verification. Only SANs matter. If your API server IP is not in the SANs, every single connection fails.

## Chapter 2: How TLS Handshake Works (Plain English)

Forget cryptography jargon. Here's what actually happens when kubectl connects to the API server:

```
kubectl (Client)                    API Server
        │                                │
        │──── "Hello, I want to connect" ──▶│
        │   (I support TLS 1.3,             │
        │    here's a random number)         │
        │                                │
        │◀── "Here's my certificate" ────│
        │   (cert contains: public key,    │
        │    my identity, CA's signature)   │
        │                                │
        │   [kubectl verifies cert was     │
        │    signed by the CA it trusts,   │
        │    and the server name matches   │
        │    a SAN in the cert]            │
        │                                │
        │──── "I trust you. Let's        ──▶│
        │     generate a shared secret      │
        │     using your public key"        │
        │                                │
        │◀──────── Encrypted Channel ────▶│
        │   (nobody on the network can     │
        │    read this anymore)             │
```

**For mTLS**, the API server also says: "Wait — you show me YOUR passport too."

```
        │──── "Here's MY certificate" ───▶│
        │   (kubectl's client cert,         │
        │    signed by the cluster CA)      │
        │                                │
        │   [API server verifies cert,     │
        │    reads CN as username,         │
        │    reads O as groups,            │
        │    passes to RBAC]               │
```

**This is the key insight**: In Kubernetes, client certificate authentication IS the authentication mechanism for control plane components. The CN and O fields in the client cert are literally the username and group that RBAC sees.

## Chapter 3: The Private Key — Why It Never Moves

The certificate is public. Anyone can see it. The private key is secret and proves you genuinely own the certificate.

```
Certificate (Public)          Private Key (Secret)
─────────────────────         ───────────────────────────
Like your name on a           Like your actual face/
passport — anyone can         fingerprints — only you
read it                       have them

Can be shared freely          NEVER leaves the machine
                              that generated it

Proves: "I claim to be X"     Proves: "I actually AM X"
```

The TLS handshake works because:

- The server signs a challenge with its private key
- The client verifies that signature using the public key in the certificate
- If it verifies → the server genuinely holds the private key → it IS who it claims to be

**In Kubernetes:** every .key file is a private key. Every .crt file is a certificate (public). The .key file must always stay on the machine that owns that identity.

## Chapter 4: Certificate Authorities (CA) — The Trust Anchor

A CA is just a certificate that has the power to sign other certificates. That's it.

```
        ┌─────────────────────────────┐
        │      KUBERNETES CA          │
        │  ca.crt + ca.key            │
        │  (Self-signed — trusts      │
        │   itself)                   │
        └──────────┬──────────────────┘
                   │ Signs
       ┌───────────┼─────────────────┐
       │           │                 │
       ▼           ▼                 ▼
  apiserver.crt  scheduler.crt  kubelet-client.crt
  (API server    (Scheduler     (Kubelet's identity
   identity)      identity)      on this node)
```

**How trust propagates:**

- You trust ca.crt
- ca.crt signed apiserver.crt
- Therefore you trust apiserver.crt
- Chain of trust — exactly like how you trust a passport because you trust the government that issued it

**Critical rule:** The CA private key (ca.key) is the master key to your entire cluster. Whoever has ca.key can issue a certificate with CN=hacker, O=system:masters and become cluster-admin. It must never leave the control plane.

## Chapter 5: Kubernetes Has THREE Separate CAs

This confuses everyone. Here's why it exists and how to remember it.

```
┌──────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                             │
│                                                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  KUBERNETES CA   │  │   ETCD CA        │  │ FRONT-PROXY CA │  │
│  │  ca.crt/ca.key   │  │  etcd/ca.crt     │  │ front-proxy-   │  │
│  │                  │  │  etcd/ca.key     │  │ ca.crt/key     │  │
│  │  Signs:          │  │                  │  │                │  │
│  │  - API server    │  │  Signs:          │  │  Signs:        │  │
│  │  - Scheduler     │  │  - etcd server   │  │  - API         │  │
│  │  - Controller    │  │  - etcd peer     │  │    aggregation │  │
│  │    manager       │  │  - etcd client   │  │    layer       │  │
│  │  - Kubelet       │  │                  │  │                │  │
│  └─────────────────┘  └─────────────────┘  └────────────────┘  │
│                                                                   │
│  ANALOGY: Three different governments for three different         │
│  departments. A visa from the K8s government doesn't let          │
│  you into etcd territory.                                         │
└──────────────────────────────────────────────────────────────────┘
```

**Why three CAs? Blast radius isolation.**

If someone compromises the etcd CA private key → they can fake etcd identities → bad, but they can't impersonate the API server or create cluster-admin users.

If there was ONE CA for everything → one compromise = total cluster takeover.

**In interviews:** When asked "why separate CAs," say: "Blast radius isolation. Compromise of one CA doesn't cascade to the other trust domains."

## Chapter 6: The Complete PKI File Map

Every file in /etc/kubernetes/pki/. Know every single one.

```
/etc/kubernetes/pki/
│
├── ca.crt           ← Kubernetes Root CA certificate (PUBLIC — everyone gets this)
├── ca.key           ← Kubernetes Root CA private key (NEVER SHARE)
│
├── apiserver.crt              ← API server's IDENTITY cert (server TLS)
├── apiserver.key              ← API server's private key
│
├── apiserver-kubelet-client.crt  ← When API server CALLS kubelet
├── apiserver-kubelet-client.key  ← (API server acts as CLIENT to kubelet)
│
├── apiserver-etcd-client.crt  ← When API server CALLS etcd
├── apiserver-etcd-client.key  ← (API server acts as CLIENT to etcd)
│
├── front-proxy-ca.crt         ← CA for API aggregation (metrics-server, etc.)
├── front-proxy-ca.key
├── front-proxy-client.crt     ← Identity for aggregated API requests
├── front-proxy-client.key
│
├── sa.pub                     ← Service Account TOKEN signing (NOT a cert)
├── sa.key                     ← Used to sign JWT tokens, not TLS
│
└── etcd/
    ├── ca.crt                 ← etcd Root CA (separate from k8s CA)
    ├── ca.key
    ├── server.crt             ← etcd server's IDENTITY cert
    ├── server.key
    ├── peer.crt               ← etcd node talks to OTHER etcd nodes
    ├── peer.key
    ├── healthcheck-client.crt ← Used by liveness probe to check etcd
    └── healthcheck-client.key
```

**The mental model for each cert**: ask "who is calling whom?" The caller needs a client cert. The callee needs a server cert.

```
Caller                    Direction    Callee
─────────────────────────────────────────────────────────────────
kubectl                   ──────────▶  API server  (apiserver.crt)
API server                ──────────▶  kubelet     (apiserver-kubelet-client.crt)
API server                ──────────▶  etcd        (apiserver-etcd-client.crt)
etcd node A               ──────────▶  etcd node B  (etcd/peer.crt)
kube-scheduler            ──────────▶  API server  (scheduler.conf has client cert)
controller-manager        ──────────▶  API server  (controller-manager.conf)
kubelet                   ──────────▶  API server  (kubelet.conf has client cert)
```

## Chapter 7: How Identity Works (The CN/O Secret)

This is the bridge between TLS and Kubernetes AuthN/AuthZ. Understand this and you understand everything.

```
Client Certificate         →    Kubernetes Sees
──────────────────────────────────────────────────────
CN=kubernetes-admin             username: kubernetes-admin
O=system:masters                group: system:masters
                                ↓
                           RBAC checks: ClusterRoleBinding
                           "system:masters" → ClusterRole "cluster-admin"
                                ↓
                           FULL ACCESS


CN=system:node:worker-1         username: system:node:worker-1
O=system:nodes                  group: system:nodes
                                ↓
                           Node Authorizer: can only modify
                           own node's resources


CN=system:kube-scheduler        username: system:kube-scheduler
                                ↓
                           ClusterRoleBinding gives scheduler
                           permission to read pods, update
                           pod/binding, etc.
```

**Interview gold**: "How does kubectl authenticate?" Answer: It sends a client TLS certificate. The API server verifies it was signed by the cluster CA, extracts CN as username and O as groups, then passes them to RBAC. No passwords, no tokens — pure certificate-based identity.

## Chapter 8: Kubeconfig — Certs Packaged for Humans

A kubeconfig is just a certificate bundle in YAML format. Nothing magical.

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <base64 of ca.crt>    # "Trust this CA"
    server: https://172.31.5.2:6443
  name: my-cluster
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <base64 of client.crt>   # "This is my passport"
    client-key-data: <base64 of client.key>            # "This proves I own it"
contexts:
- context:
    cluster: my-cluster
    user: kubernetes-admin
  name: kubernetes-admin@my-cluster
```

These three kubeconfig files are generated by kubeadm and placed on the control plane:

```
/etc/kubernetes/admin.conf                ← for cluster admins (kubectl)
/etc/kubernetes/controller-manager.conf  ← for kube-controller-manager
/etc/kubernetes/scheduler.conf           ← for kube-scheduler
```

Each one embeds a different client cert with a different CN, giving each component a different identity and different RBAC permissions.

## Chapter 9: TLS Bootstrapping — How Worker Nodes Get Their Certs

The chicken-and-egg problem: a node needs a cert to talk to the API server, but needs to talk to the API server to get a cert.

**The solution in 5 steps:**

```
Step 1: Admin creates a bootstrap token
        Format: <6-chars>.<16-chars>  e.g., abcdef.0123456789abcdef
        Stored as a Secret in kube-system namespace
        Has permission to CREATE CertificateSigningRequest objects

Step 2: Node starts kubelet with /etc/kubernetes/bootstrap-kubelet.conf
        This kubeconfig uses the bootstrap token instead of a cert
        Kubelet connects to API server using this token

Step 3: kubelet submits a CSR (Certificate Signing Request)
        CertificateSigningRequest object created in K8s API
        Content: "Please sign a cert for system:node:worker-1"

Step 4: Controller-manager auto-approves it
        (if NodeBootstrapper ClusterRoleBinding exists)
        Signs the CSR with the cluster CA

Step 5: kubelet receives its signed cert
        Saves it to /var/lib/kubelet/pki/
        Deletes bootstrap-kubelet.conf
        Uses real cert for all future connections
```

After bootstrapping, **kubelet auto-rotates its cert** before expiry. This is enabled by default since K8s 1.19 via RotateKubeletClientCertificate feature gate.

## Chapter 10: cert-manager — Automating Certificate Lifecycle

The problem without cert-manager:

- You manually create CSRs
- Manually submit to Let's Encrypt
- Manually download certs
- Manually create Kubernetes Secrets
- Manually remember to renew before expiry
- Miss one renewal → production outage

**cert-manager solves this completely.** It's a Kubernetes controller that watches for Certificate objects and handles everything.

**The 4 Objects You Must Know**

```
1. ClusterIssuer / Issuer
   "HOW do I get certs signed?"
   (Let's Encrypt, internal CA, Vault, etc.)
   ClusterIssuer = works across all namespaces
   Issuer = works only within one namespace

2. Certificate
   "WHAT cert do I want?"
   (domains, duration, algorithm, secret name)

3. CertificateRequest  [cert-manager creates this internally]
   "The actual CSR being sent to the issuer"

4. Order + Challenge  [cert-manager creates these internally for ACME]
   "The Let's Encrypt verification dance"
```

**The Complete cert-manager Flow**

```
You apply Certificate YAML
         │
         ▼
cert-manager controller sees it
         │
         ▼
Does the Secret exist and is it valid?
    YES ──────────────────────────────────▶ Done, watch for expiry
    NO
         │
         ▼
Create CertificateRequest
(generate private key, create CSR)
         │
         ▼
    Which Issuer type?
    ┌────┴─────┬──────────────┐
    ▼          ▼              ▼
 ACME      Internal CA     Vault PKI
(Let's Encrypt)
    │          │              │
    ▼          │              │
Create Order   │              │
    │          │              │
    ▼          │              │
Create Challenge(s)           │
HTTP-01 or DNS-01             │
    │          │              │
    ▼          │              │
Solve challenge│              │
(create Ingress│              │
 rule or DNS   │              │
 TXT record)   │              │
    │          │              │
    ▼          │              │
LE verifies ◀──┘              │
and issues cert               │
    │                         │
    └──────────┬──────────────┘
               │
               ▼
    Store cert + key in Secret
    (tls.crt and tls.key fields)
               │
               ▼
    Watch expiry — renew at renewBefore threshold
    (default: 2/3 through certificate lifetime)
```

**HTTP-01 vs DNS-01 — Crystal Clear**

```
HTTP-01 Challenge                     DNS-01 Challenge
─────────────────────────────────     ──────────────────────────────────
How: serve a token at                 How: create a DNS TXT record at
     http://yourdomain.com/           _acme-challenge.yourdomain.com
     .well-known/acme-challenge/X
Who verifies: LE fetches the URL      Who verifies: LE does DNS lookup

Works for: public domains             Works for: public AND private domains
Does NOT work for: wildcard certs     Works for: wildcard certs ✓
                   private domains     
Requires: public-facing Ingress       Requires: DNS provider API access
                                                (Route53, Cloudflare, etc.)

Use when: simple public domain cert   Use when: *.example.com wildcard
                                               or internal/private domain
```

**cert-manager in YAML — Production Setup**

```
# Step 1: ClusterIssuer — How to get certs
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@yourcompany.com
    privateKeySecretRef:
      name: letsencrypt-account-key   # LE account private key
    solvers:
    - http01:                          # For regular domains
        ingress:
          class: nginx
    - dns01:                           # For wildcards
        route53:
          region: us-east-1
          hostedZoneID: ZXXXXX
          accessKeyIDSecretRef:
            name: route53-creds
            key: access-key-id
          secretAccessKeySecretRef:
            name: route53-creds
            key: secret-access-key
---
# Step 2: Certificate — What cert you want
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls              # cert stored here when issued
  duration: 2160h                    # 90 days
  renewBefore: 720h                  # Renew 30 days before expiry
  privateKey:
    algorithm: ECDSA                 # Faster + smaller than RSA
    size: 256
    rotationPolicy: Always           # New key on every renewal
  dnsNames:
  - myapp.example.com
  - www.myapp.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

## Chapter 11: TLS Secrets — What's Actually Inside

```
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: myapp-tls
  namespace: production
data:
  tls.crt: <base64>    # Cert chain: YOUR cert + intermediate CA cert(s)
  tls.key: <base64>    # Your private key
```

**What goes in tls.crt — common confusion:**

```
CORRECT tls.crt (full chain):          WRONG tls.crt (leaf only):
──────────────────────────────         ──────────────────────────────
-----BEGIN CERTIFICATE-----            -----BEGIN CERTIFICATE-----
<Your leaf certificate>                <Your leaf certificate>
-----END CERTIFICATE-----              -----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----            
<Intermediate CA cert>                 ← Missing! Clients that don't
-----END CERTIFICATE-----                have this intermediate in
                                         their store will fail
```

**Order matters:** leaf cert first, then intermediates. Never include the root CA.

Why? Because clients already have root CAs in their trust store. Including the full chain up to (but not including) the root ensures any client can verify the chain.

## Chapter 12: Ingress TLS — The Complete Picture

```
Internet User
     │
     │  HTTPS request to myapp.example.com
     ▼
AWS Load Balancer (passes TCP)
     │
     ▼
Ingress Controller (nginx/traefik)
     │  Terminates TLS here
     │  Reads tls.crt and tls.key from the Secret
     │  Presents cert to the client
     ▼
Your Pod (receives plain HTTP on port 80)
```

**Ingress with cert-manager annotation (simplest way):**

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    # This single annotation triggers cert-manager ingress-shim
    # It auto-creates a Certificate object for you
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls    # cert-manager will create this Secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
```

What happens automatically:

- cert-manager ingress-shim sees the annotation
- Creates a Certificate object targeting myapp-tls Secret
- Runs the ACME challenge
- Stores cert in myapp-tls Secret
- nginx reads the Secret and serves HTTPS
- cert-manager renews automatically before expiry

**Gateway API TLS (modern approach — know this too):**

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: myapp.example.com
    tls:
      mode: Terminate              # Gateway terminates TLS
      certificateRefs:
      - name: myapp-tls            # Same tls Secret
        kind: Secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp-route
spec:
  parentRefs:
  - name: prod-gateway
  hostnames:
  - myapp.example.com
  rules:
  - backendRefs:
    - name: myapp-svc
      port: 80
```

**TLS modes in Gateway API:**

```
Terminate:   Gateway reads cert, decrypts traffic, sends HTTP to backend
             Use when: you want to inspect traffic, app doesn't need TLS

Passthrough: Gateway looks at SNI hostname only, forwards raw TLS to backend
             Backend must have the cert and terminate TLS itself
             Use when: end-to-end encryption required, backend handles TLS
```

## Chapter 13: mTLS — The Complete Picture

**Regular TLS vs mTLS in one diagram:**

```
Regular TLS (one-way):
  Client ──── "prove you're real" ────▶ Server shows cert
  Client verifies server cert
  Encrypted channel open
  Server has NO IDEA who the client is (only IP address)


mTLS (two-way):
  Client ──── "prove you're real" ────▶ Server shows cert
  Client verifies server cert
  Server ──── "you prove too" ────────▶ Client shows cert
  Server verifies client cert, extracts identity
  Both parties are cryptographically verified
```

**Why mTLS for microservices?**

```
Without mTLS:
  payments-service ──── HTTP ────▶ accounts-service
  
  Problem: accounts-service has no idea if the caller is really
  payments-service or an attacker who compromised a pod.
  Network policy is the only defense.


With mTLS:
  payments-service ──── mTLS ────▶ accounts-service
  
  accounts-service verifies: "The caller's cert says SPIFFE ID
  spiffe://cluster.local/ns/prod/sa/payments-service
  signed by our cluster CA — it genuinely IS payments-service"
```

**How Istio implements mTLS without changing your app:**

```
Pod A (payments-service)          Pod B (accounts-service)
┌───────────────────────┐         ┌───────────────────────┐
│  app code             │         │  app code             │
│  talks HTTP :8080     │         │  listens HTTP :8080   │
│         │             │         │         ▲             │
│         ▼             │  mTLS   │         │             │
│  envoy proxy          │◀───────▶│  envoy proxy          │
│  (sidecar container)  │         │  (sidecar container)  │
│                       │         │                       │
│  SPIFFE cert from     │         │  SPIFFE cert from     │
│  Istiod               │         │  Istiod               │
└───────────────────────┘         └───────────────────────┘

Your app code never changes. Envoy intercepts all traffic transparently.
```

**Istiod provisions certificates using SPIFFE:**

```
SPIFFE ID format:
spiffe://cluster.local/ns/<namespace>/sa/<service-account>

Example:
payments-service in namespace prod with SA payments-sa gets:
spiffe://cluster.local/ns/prod/sa/payments-sa

This ID is embedded as a URI SAN in the X.509 cert.
Rotated every 24 hours automatically.
```

**The three Istio mTLS modes you must know:**

```
PERMISSIVE: Accepts both mTLS AND plaintext
            ← Migration mode only
            ← NEVER use in production long-term
            ← AuthorizationPolicies are UNENFORCED for plaintext callers

STRICT:     Accepts ONLY mTLS connections
            ← Production standard
            ← Any non-meshed pod is rejected at the network level

DISABLE:    No mTLS
            ← Legacy workloads only
```

```
# Enforce STRICT mTLS cluster-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system   # istio-system = cluster-wide scope
spec:
  mtls:
    mode: STRICT
```

## Chapter 14: Certificate Rotation and Expiry

**kubeadm cert expiry — the silent killer:**

kubeadm creates control plane certs with 1-year validity by default. On your cluster's first birthday, the control plane silently dies if you haven't renewed.

```
# CHECK THIS IN EVERY CLUSTER YOU MANAGE
kubeadm certs check-expiration

# Expected output:
CERTIFICATE                EXPIRES                  RESIDUAL TIME
admin.conf                 Jan 01, 2026 00:00 UTC   90d
apiserver                  Jan 01, 2026 00:00 UTC   90d
apiserver-etcd-client      Jan 01, 2026 00:00 UTC   90d
etcd-healthcheck-client    Jan 01, 2026 00:00 UTC   90d
etcd-peer                  Jan 01, 2026 00:00 UTC   90d
etcd-server                Jan 01, 2026 00:00 UTC   90d
front-proxy-client         Jan 01, 2026 00:00 UTC   90d
scheduler.conf             Jan 01, 2026 00:00 UTC   90d

CA certs are valid for 10 years (don't confuse with component certs)

# RENEW ALL CERTS
kubeadm certs renew all

# After renewal, restart control plane components (they read certs on startup)
# For static pods, move the manifest out and back in — kubelet restarts them
```

**cert-manager auto-renewal timeline:**

```
Cert issued (day 0)
        │
        │         Cert valid period
        │    (90 days for Let's Encrypt)
        │
        ├──────────────────────────────────────────────────▶
        0%                  66%              100%
        Issued              ↑                Expires
                     cert-manager
                     starts renewal
                     at this point
                     (or at renewBefore threshold)
```

**Volume mounts vs environment variables for cert rotation:**

```
Volume mount (CORRECT):
  Secret updated → kubelet syncs within ~60s → file on disk changes
  App reads file → sees new cert
  → Zero downtime rotation

Environment variable (WRONG for rotation):
  Secret updated → nothing happens
  Env var still holds old value
  Cert expires → app crashes
  Pod restart required → brief downtime

ALWAYS mount TLS certs as volumes, not env vars.
```

## Chapter 15: Production Failure Modes (Interview Gold)

**Failure 1: SAN Mismatch**

```
Symptom: x509: certificate is valid for kubernetes,
         kubernetes.default, not 54.23.11.5

Root cause: API server cert was issued before an NLB was added.
            NLB's IP is not in the SANs.

Fix:
kubeadm init phase certs apiserver \
  --apiserver-cert-extra-sans=54.23.11.5,api.mycluster.com

Then restart kube-apiserver static pod.

Diagnose:
openssl s_client -connect <api-server-ip>:6443 2>/dev/null \
  | openssl x509 -noout -text \
  | grep -A5 "Subject Alternative Name"
```

**Failure 2: Expired kubeadm Certs**

```
Symptom: kubectl returns "Unable to connect to server: EOF"
         or "certificate has expired or is not yet valid"

Root cause: 1-year kubeadm cert expiry reached.
            Nobody set up monitoring.

Recovery (even if already expired):
kubeadm certs renew all
# kubeadm can renew expired certs
# Then restart all control plane static pods

Prevention:
# Prometheus alert
- alert: KubeadmCertExpiryIn30Days
  expr: (kubeadm_cert_expiry_timestamp_seconds - time()) < (30 * 24 * 3600)
  severity: warning
```

**Failure 3: cert-manager PERMISSIVE Mode Left Enabled**

```
Symptom: mTLS is configured but services can still be called
         by unknown/unauthorized pods.

Root cause: PeerAuthentication is PERMISSIVE, not STRICT.
            AuthorizationPolicies only check source.principal,
            which is empty for non-mTLS (plaintext) calls.
            Policy effectively unenforced.

Fix: Switch to STRICT after all workloads are meshed.
     Test: kubectl exec non-meshed-pod -- curl http://payments-service
     Should return: Connection reset (STRICT)
     If it returns 200 (PERMISSIVE) → you have a problem
```

**Failure 4: Let's Encrypt Rate Limit Hit**

```
Symptom: cert-manager Order stuck in "pending", challenge errors show
         "too many certificates already issued for exact set of domains"

Root cause: Let's Encrypt allows 50 certs per registered domain per week.
            Dev/staging environment churning through certs.

Fix: ALWAYS use staging for non-production:
     https://acme-staging-v02.api.letsencrypt.org/directory
     
     Staging issues fake certs (not trusted by browsers, but
     the whole flow works for testing)

Recovery: Wait 7 days or use a different subdomain.
```

**Failure 5: Missing Certificate Chain in tls.crt**

```
Symptom: Chrome shows "NET::ERR_CERT_AUTHORITY_INVALID"
         curl shows "unable to verify the first certificate"
         openssl verify shows "unable to get local issuer certificate"

Root cause: tls.crt only contains the leaf cert.
            Intermediate CA cert is missing.
            Clients that don't have the intermediate cached fail.

Fix: Concatenate full chain into tls.crt:
cat mycert.crt intermediate-ca.crt > tls.crt
# Leaf cert FIRST, then intermediate CA

Verify:
openssl verify -CAfile ca.crt tls.crt
# Should say: tls.crt: OK
```

**Failure 6: Cross-Namespace Secret Reference Blocked**

```
Symptom: Ingress can't find TLS Secret even though it exists.
         "secret 'other-namespace/myapp-tls' not found"

Root cause: Ingress TLS Secrets must be in the SAME namespace as the Ingress.
            This is enforced by nginx ingress controller by default.

Fix options:
1. Create the cert in the same namespace
   (use an Issuer not ClusterIssuer, scoped to that namespace)

2. Use cert-manager's "additionalOutputFormats" or "secretTemplate"
   to copy secrets across namespaces

3. Use Gateway API ReferenceGrant for explicit cross-namespace access:

apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-tls-secret
  namespace: certs       # namespace WHERE the secret lives
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: production   # namespace ACCESSING the secret
  to:
  - group: ""
    kind: Secret
```

## Chapter 16: The Interview Questions (With Model Answers)

**Q1: "Walk me through what happens when I run kubectl get pods"**

kubectl reads ~/.kube/config, finds the client certificate and private key for the current context. It establishes a TLS connection to the API server at port 6443, presenting the client certificate. The API server verifies the cert was signed by the cluster CA, extracts the CN field as username and O field as groups, passes to RBAC which checks if this user has get permission on pods in the requested namespace. If allowed, returns the pod list over the encrypted TLS channel.

**Q2: "Why does my cert work in Postman but fail in my Go service?"**

Go 1.15+ dropped support for the CN field for hostname verification. Only SANs are checked. Postman (or older curl versions) may still check CN. Your cert is missing the correct SAN. Fix: regenerate the cert with the correct DNS/IP SANs.

**Q3: "What happens if cert-manager goes down?"**

Existing certs continue to work until they expire. New certificate requests won't be processed. Renewals won't happen. If cert-manager is down long enough that a cert expires, that service loses HTTPS. Prevention: run cert-manager with 2+ replicas, monitor certmanager_certificate_expiry_seconds metric, alert at 30 days, page at 7 days. Set renewBefore: 720h (30 days) to give yourself a wide recovery window.

*Q4: "How do you issue a wildcard cert like *.example.com?"*

Must use DNS-01 ACME challenge. HTTP-01 cannot issue wildcard certs — this is an ACME protocol restriction because HTTP-01 only proves control of one specific hostname, while DNS-01 proves control of the entire DNS zone. Configure cert-manager with a DNS-01 solver using Route53/Cloudflare API credentials, then specify *.example.com in the Certificate dnsNames.

**Q5: "What is the difference between an Issuer and a ClusterIssuer?"**

An Issuer is namespace-scoped — it can only issue certs for Certificate objects in the same namespace. A ClusterIssuer is cluster-scoped — it can issue certs for Certificate objects in any namespace. Use ClusterIssuer for shared infrastructure like Let's Encrypt. Use Issuer per-namespace for multi-tenant isolation where different teams manage their own CAs.

**Q6: "An attacker compromised a worker node. What certificates are at risk?"**

The node holds: (1) The kubelet's client certificate and private key in /var/lib/kubelet/pki/ — this gives system:node:<nodename> identity and the Node authorizer grants it read access to secrets for pods scheduled on that node. (2) The ca.crt (public, no risk). (3) Any TLS secrets mounted into pods running on that node. Critically, it does NOT hold ca.key (only on control plane) so the attacker cannot issue new certificates. Mitigation: immediately drain and cordon the node, revoke/rotate all service account tokens for pods that ran on it, audit what secrets those pods had access to.

**Q7: "Explain mTLS vs network policy. Which is stronger?"**

Network policy is layer 3/4 — it allows or denies based on IP addresses and ports. It cannot verify application identity; a compromised pod at an allowed IP can still make any call. mTLS is layer 4/7 — it cryptographically verifies the identity of both parties using certificates. The identity is tied to the service account (SPIFFE SVID), not the IP. mTLS is stronger: it provides cryptographic identity verification and works even if network policy is misconfigured. Best practice: use both — network policy for defense in depth at the network layer, mTLS for application-level zero-trust identity.

**The Master Cheat Sheet**

```
┌──────────────────────────────────────────────────────────────────────┐
│               KUBERNETES TLS MASTER REFERENCE                        │
├──────────────────────────────────────────────────────────────────────┤
│ CERTIFICATE = Identity card + CA's signature                         │
│ CA = Trusted issuer. 3 in K8s: k8s CA, etcd CA, front-proxy CA     │
│ PRIVATE KEY = Proof of ownership. Never leaves origin machine        │
│ SAN = The ONLY field modern TLS checks for hostname verification     │
│ CN+O in client cert = username+groups for Kubernetes RBAC            │
├──────────────────────────────────────────────────────────────────────┤
│ CERT LOCATIONS                                                        │
│ Control plane:   /etc/kubernetes/pki/                                │
│ etcd:            /etc/kubernetes/pki/etcd/                           │
│ kubelet:         /var/lib/kubelet/pki/                               │
│ kubeconfigs:     /etc/kubernetes/*.conf                              │
├──────────────────────────────────────────────────────────────────────┤
│ CERT-MANAGER FLOW                                                    │
│ ClusterIssuer → Certificate → CertificateRequest → Secret           │
│ ACME: HTTP-01 (public domains) | DNS-01 (wildcards + private)       │
│ Renewal: at renewBefore threshold (default ~60d for 90d cert)        │
├──────────────────────────────────────────────────────────────────────┤
│ ROTATION RULES                                                        │
│ kubeadm certs: 1yr default → monitor + renew before expiry          │
│ cert-manager: auto-renews → set renewBefore: 720h (30 days)         │
│ Secrets: volume mount → auto-updates | env vars → needs pod restart │
├──────────────────────────────────────────────────────────────────────┤
│ mTLS RULES                                                            │
│ STRICT = production | PERMISSIVE = migration only, never leave here │
│ Service mesh (Istio/Linkerd) = automatic mTLS without app changes   │
│ SPIFFE SVID = workload identity tied to namespace/serviceaccount    │
├──────────────────────────────────────────────────────────────────────┤
│ TOP FAILURE MODES                                                    │
│ 1. Missing SAN → x509: cert valid for X not Y                       │
│ 2. Kubeadm 1yr expiry → cluster goes silent                         │
│ 3. cert-manager down → no renewals → eventual expiry                │
│ 4. LE rate limits → use staging for all testing                     │
│ 5. Incomplete cert chain → intermediate CA missing in tls.crt       │
│ 6. PERMISSIVE mTLS → AuthorizationPolicies unenforced               │
└──────────────────────────────────────────────────────────────────────┘
```

The one thing to remember above everything else: a certificate is just a signed identity claim, and everything in Kubernetes uses them to answer the question "who are you, and can I trust you?" Every complexity in this topic — the three CAs, mTLS, cert-manager, bootstrap tokens — is just a different context where that same question is being asked.

********************************************************************************************************************************************************************

## Booking.com / Adyen Platform Engineer Interview

**TLS & Certificate Security in Kubernetes**

```
INTERVIEW CONTEXT
─────────────────────────────────────────────────────
Role:       Senior Platform Engineer / SRE
Companies:  Booking.com / Adyen (combined style)
Round:      Technical Deep-Dive (60 minutes)
Interviewer: Staff Engineer, Platform Security Team

RULES FOR THIS SIMULATION
─────────────────────────────────────────────────────
→ Answer each question BEFORE scrolling to feedback
→ I will score your answer, show you what was missing,
  and give you the model answer a Staff Engineer gives
→ Questions get progressively harder
→ I will challenge your answers like a real interviewer
→ No looking back at the masterclass — answer from memory
→ At the end: overall verdict + hiring recommendation
─────────────────────────────────────────────────────
Total Questions: 8
Format: Warm-up → Technical → Scenario → System Design
```

```
👤 INTERVIEWER:

"Hi Chetan, thanks for joining. I'm a Staff Engineer on the 
Platform team here. We run about 400 microservices on Kubernetes 
across multiple regions. Today I want to go deep on certificate 
management and TLS security — it's something we deal with at 
scale every day.

Let's start straightforward and build up from there."
```

**Question 1 of 8 — Warm Up**

```
👤 INTERVIEWER:

"When a developer runs kubectl get secrets in our cluster,
walk me through exactly what happens from a TLS and 
authentication perspective. 

Be specific — I want to know what certificates are involved,
what fields are read, and how Kubernetes knows whether to 
allow or deny the request."
```

**Take 2–3 minutes. Write your full answer below.**

When you reply, I'll score it on:

- Did you cover the TLS handshake role?
- Did you identify client certificates specifically?
- Did you mention CN and O fields → username/groups?
- Did you connect it to RBAC?
- Did you mention what the API server verifies?

**Question 1 — Model Answer Breakdown**

No problem at all. This is exactly what the learning process looks like. Let me build the answer layer by layer so you understand it deeply, not just memorize it.

**The Mental Framework First**

Before answering any "walk me through" question in an interview, use this structure:

```
EVERY kubectl command follows this exact journey:

  Step 1: READ      → kubectl reads credentials from kubeconfig
  Step 2: CONNECT   → TLS handshake with API server
  Step 3: IDENTIFY  → API server extracts who you are
  Step 4: AUTHORIZE → RBAC decides what you can do
  Step 5: RESPOND   → Answer comes back over encrypted channel
```

Never skip steps. Interviewers want to see you think in sequence.

### The Complete Answer (What a Staff Engineer Says)

**STEP 1 — kubectl Reads Kubeconfig**

```
Developer runs:
$ kubectl get secrets -n production

kubectl immediately looks for credentials in:
  → $KUBECONFIG env var
  → OR ~/.kube/config (default)

Inside kubeconfig, it finds THREE pieces of information:

┌─────────────────────────────────────────────────────┐
│ 1. certificate-authority-data                        │
│    = base64 of ca.crt                               │
│    = "Here is the CA I should TRUST"                │
│                                                      │
│ 2. client-certificate-data                           │
│    = base64 of client.crt                           │
│    = "This is MY IDENTITY (my passport)"            │
│                                                      │
│ 3. client-key-data                                   │
│    = base64 of client.key                           │
│    = "This PROVES I own my identity"                │
└─────────────────────────────────────────────────────┘

Also reads:
  → server: https://api.mycluster.com:6443
    (where to connect)
```

**STEP 2 — TLS Handshake (This is Where Certs Are Used)**

```
kubectl (Client)                   API Server (:6443)
        │                                │
        │──── TCP connect ───────────────▶│
        │                                │
        │──── ClientHello ───────────────▶│
        │     "I want TLS 1.3,           │
        │      here's my random nonce"   │
        │                                │
        │◀─── ServerHello ───────────────│
        │◀─── apiserver.crt ─────────────│  ← API server shows ITS passport
        │                                │
        │  [kubectl now verifies:        │
        │   1. Was apiserver.crt signed  │
        │      by the ca.crt I trust?   │
        │   2. Does the server address   │
        │      match a SAN in the cert?  │
        │   If either fails → ABORT]     │
        │                                │
        │◀─── "Show me YOUR cert too" ───│  ← THIS is what makes it mTLS
        │                                │
        │──── client.crt ───────────────▶│  ← kubectl shows its passport
        │──── Proof of private key ─────▶│  ← proves it genuinely owns cert
        │                                │
        │  [API server verifies:         │
        │   Was client.crt signed by     │
        │   cluster CA? If yes → valid]  │
        │                                │
        │◀═══════ Encrypted Channel ════▶│
        │   (everything from here is     │
        │    encrypted and authenticated)│
```

**The critical interview point here:**

Kubernetes does not use usernames and passwords. The client TLS certificate IS the authentication mechanism for kubectl and all control plane components.

**STEP 3 — API Server Extracts Identity**

```
API server opens the client.crt and reads two fields:

┌─────────────────────────────────────────────────────┐
│            CLIENT CERTIFICATE                        │
│                                                      │
│  Subject: CN=kubernetes-admin, O=system:masters      │
│           ──────────────────   ─────────────────     │
│                  │                    │              │
│                  ▼                    ▼              │
│           Username in K8s      Group in K8s          │
│           "kubernetes-admin"   "system:masters"      │
└─────────────────────────────────────────────────────┘

This is passed to the Authentication layer:
"The caller is user 'kubernetes-admin',
 member of group 'system:masters'"

No database lookup. No token validation.
Just reading two fields from a certificate.
This is why protecting ca.key is critical —
anyone with ca.key can create a cert with
O=system:masters and become cluster-admin.
```

**STEP 4 — RBAC Authorization**

```
Authentication result:
  user:   kubernetes-admin
  groups: [system:masters]

RBAC checks:
  "Is there a RoleBinding or ClusterRoleBinding
   that gives this user/group permission to
   GET secrets in namespace 'production'?"

K8s ships with a built-in ClusterRoleBinding:

ClusterRoleBinding: cluster-admin
  subjects:
  - kind: Group
    name: system:masters      ← our user is in this group
  roleRef:
    name: cluster-admin       ← this ClusterRole has all permissions
    kind: ClusterRole

cluster-admin ClusterRole has:
  rules:
  - apiGroups: ["*"]
    resources: ["*"]         ← everything
    verbs: ["*"]             ← all verbs

Result: ALLOWED
```

**STEP 5 — Response Over Encrypted Channel**

```
API server:
  → Queries etcd for secrets in 'production' namespace
  → Serializes response to JSON
  → Sends back over the already-encrypted TLS channel

kubectl:
  → Decrypts response (TLS handles this transparently)
  → Formats and prints to terminal

Developer sees their secrets.
Every byte of that response was encrypted
by the TLS session established in Step 2.
```

**The Complete Answer in Interview Format**

This is exactly what you say out loud in an interview. Read this until you can say it naturally:

```
"When kubectl get secrets runs, five things happen:

First, kubectl reads the kubeconfig file. It finds three 
things — the cluster CA certificate to verify the server, 
the client certificate as the developer's identity, and 
the client private key to prove ownership.

Second, kubectl initiates a TLS handshake with the API 
server on port 6443. The API server presents its serving 
certificate — apiserver.crt. kubectl verifies two things: 
was this cert signed by the CA it trusts, and does the 
server's hostname or IP appear in the cert's SAN field. 
If either check fails, the connection is aborted.

Third — and this is the mTLS part — the API server also 
requests a certificate from kubectl. kubectl presents its 
client certificate. The API server verifies it was signed 
by the cluster CA. This is how Kubernetes authenticates 
the caller — no passwords, no tokens for kubectl — pure 
certificate-based identity.

Fourth, the API server reads the CN field of the client 
cert as the username, and the O field as the group. For a 
typical admin kubeconfig, that's CN=kubernetes-admin and 
O=system:masters. These are passed to RBAC. The 
system:masters group is bound to the cluster-admin 
ClusterRole via a built-in ClusterRoleBinding, which 
grants all permissions. So the request is allowed.

Fifth, the API server fetches the secrets from etcd and 
returns them over the encrypted TLS channel.

One thing worth flagging as a security concern: because 
the O field in a client cert maps directly to a Kubernetes 
group, anyone who has access to the cluster CA private key 
can sign a cert with O=system:masters and get full cluster 
access. Protecting ca.key is therefore the single most 
critical security control in a Kubernetes cluster."
```

**Why That Last Paragraph Matters**

```
AVERAGE CANDIDATE:  Describes the flow correctly, stops there.

SENIOR CANDIDATE:   Describes the flow AND proactively identifies
                    the security implication without being asked.

STAFF ENGINEER:     Does both of the above AND mentions the
                    mitigation (ca.key restricted to control
                    plane nodes, no remote access, HSM in
                    regulated environments).
```

Always end a "walk me through" answer with a **security implication or operational gotcha**. That's what separates senior from staff-level answers.

**Memory Anchor for This Question**

Burn this into your brain:

```
kubectl get secrets =

READ kubeconfig
  ↓
CONNECT via TLS (verify server cert SANs)
  ↓
PROVE identity (present client cert)
  ↓
EXTRACT identity (CN = username, O = group)
  ↓
RBAC check (group → ClusterRoleBinding → ClusterRole → verbs)
  ↓
RETURN data over encrypted channel

GOTCHA: CA private key = master key to bypass all of this
```

## Complete Interview Q&A — All 8 Questions

**Booking.com / Adyen Senior Platform Engineer Style**

```
HOW TO USE THIS DOCUMENT
──────────────────────────────────────────────────────────────
Read Question → Try to answer in your head → Read Model Answer
Do this 3 times per question over 2 days.
On day 3, close the doc and answer from memory.
That is when you are interview-ready.
──────────────────────────────────────────────────────────────

QUESTION PROGRESSION
──────────────────────────────────────────────────────────────
Q1 → Warm up       (Fundamentals)
Q2 → Technical     (Deep mechanics)
Q3 → Technical     (cert-manager internals)
Q4 → Scenario      (Production incident)
Q5 → Scenario      (Security breach)
Q6 → Deep Tech     (mTLS + service mesh)
Q7 → System Design (Multi-region architecture)
Q8 → Staff Level   (You challenge the interviewer)
──────────────────────────────────────────────────────────────
```

## QUESTION 1 — Fundamentals

**"Walk me through what happens when kubectl get secrets runs"**

```
👤 INTERVIEWER:
"When a developer runs kubectl get secrets in our cluster,
walk me through exactly what happens from a TLS and
authentication perspective. Be specific — what certificates
are involved, what fields are read, and how Kubernetes knows
whether to allow or deny the request."
```

**The Mental Map**

```
READ kubeconfig
      ↓
TLS HANDSHAKE (verify server cert)
      ↓
PRESENT client cert (mTLS)
      ↓
EXTRACT identity (CN → username, O → groups)
      ↓
RBAC decision
      ↓
RETURN data over encrypted channel
```

**Step 1 — kubectl Reads Kubeconfig**

```
$ kubectl get secrets -n production

kubectl reads ~/.kube/config and extracts:

┌─────────────────────────────────────────────────────┐
│ certificate-authority-data  = ca.crt (base64)        │
│ "Trust this CA to verify the server"                 │
│                                                      │
│ client-certificate-data     = client.crt (base64)   │
│ "This is MY identity — my passport"                  │
│                                                      │
│ client-key-data             = client.key (base64)   │
│ "This PROVES I genuinely own my identity"            │
│                                                      │
│ server = https://api.mycluster.com:6443              │
│ "Connect here"                                       │
└─────────────────────────────────────────────────────┘
```

**Step 2 — TLS Handshake**

```
kubectl                              API Server (:6443)
   │                                        │
   │──── ClientHello ──────────────────────▶│
   │     "TLS 1.3, here's my random nonce"  │
   │                                        │
   │◀─── ServerHello ───────────────────────│
   │◀─── apiserver.crt ─────────────────────│
   │     (API server shows its passport)    │
   │                                        │
   │  kubectl verifies TWO things:          │
   │  1. Was apiserver.crt signed by        │
   │     the ca.crt I trust? → YES/NO       │
   │  2. Is the server IP/DNS in the        │
   │     cert's SAN field? → YES/NO         │
   │  If either is NO → connection aborts   │
   │                                        │
   │◀─── "Show me YOUR cert too" ───────────│
   │     (This is what makes it mTLS)       │
   │                                        │
   │──── client.crt ───────────────────────▶│
   │──── Proof of private key ownership ───▶│
   │                                        │
   │◀══════ Encrypted Channel Open ════════▶│
```

**Step 3 — API Server Extracts Identity**

```
API server opens client.crt and reads:

Subject: CN=kubernetes-admin, O=system:masters
         ─────────────────   ─────────────────
                │                   │
                ▼                   ▼
         Username in K8s      Group in K8s

NO database lookup.
NO password check.
Just two fields in a certificate.

This is passed to the Authentication layer:
→ user:   "kubernetes-admin"
→ groups: ["system:masters"]
```

**Step 4 — RBAC Authorization**

```
RBAC evaluates:
"Does user 'kubernetes-admin' in group 'system:masters'
 have GET permission on secrets in namespace 'production'?"

Built-in ClusterRoleBinding (shipped with K8s):
  name: cluster-admin
  subjects:
  - kind: Group
    name: system:masters      ← our user is in this group
  roleRef:
    kind: ClusterRole
    name: cluster-admin       ← grants ALL permissions

Result: ALLOWED → API server queries etcd → returns secrets
```

**The Gotcha Every Senior Engineer Adds**

```
SECURITY IMPLICATION (say this unprompted):

Because O=system:masters in a client cert grants full
cluster-admin access, the cluster CA private key (ca.key)
is the single most critical secret in the entire cluster.

Anyone who holds ca.key can:
  openssl genrsa -out evil.key 2048
  openssl req -new -key evil.key \
    -subj "/CN=hacker/O=system:masters" -out evil.csr
  openssl x509 -req -in evil.csr -CA ca.crt \
    -CAkey ca.key -out evil.crt

And they now have permanent, certificate-based
cluster-admin access that survives RBAC changes.

Mitigation:
→ ca.key never leaves control plane nodes
→ No SSH access to control plane in prod
→ Regulated environments use HSM-backed CAs
→ Audit all cert signing operations
```

**Model Answer (Say Exactly This)**

"When kubectl get secrets runs, five things happen in sequence.

First, kubectl reads the kubeconfig file — it finds the cluster CA cert to verify the server, a client certificate as the user's identity, and the matching private key.

Second, kubectl establishes a TLS connection to the API server on port 6443. The API server presents its serving certificate. kubectl verifies two things: was this cert signed by the CA in the kubeconfig, and does the server's address match a SAN in the cert. If either check fails, the connection aborts immediately.

Third — this is the mTLS part — the API server also requests a client certificate from kubectl. kubectl presents its client cert, and the API server verifies it was signed by the cluster CA. This is Kubernetes' authentication mechanism for kubectl — no passwords, no tokens.

Fourth, the API server reads the CN field as the username and O field as the group. For a standard admin kubeconfig that's kubernetes-admin and system:masters. RBAC checks if that group has GET access on secrets. system:masters is bound to cluster-admin ClusterRole, so it's allowed.

Fifth, the API server fetches the secrets from etcd and returns them over the encrypted channel.

One security point worth noting: because the O field in a client cert directly maps to a Kubernetes group, the CA private key is the master key to the cluster. Whoever has ca.key can generate a cert with system:masters and bypass all RBAC. In production, ca.key must never leave the control plane."

## QUESTION 2 — Deep Mechanics

**"Explain SAN fields and why they are the #1 cause of TLS failures"**

```
👤 INTERVIEWER:
"We've had two incidents this quarter where TLS connections
failed after infrastructure changes. Both turned out to be
SAN-related. Can you explain what SANs are, why they matter
more than the CN field, and how you would diagnose and fix
a SAN mismatch in production?"
```

**What SAN Is and Why CN No Longer Matters**

```
X.509 Certificate has TWO places that claim identity:

┌─────────────────────────────────────────────────────┐
│ CN (Common Name) — OLD way                           │
│ Subject: CN=api.mycluster.com                        │
│                                                      │
│ SAN (Subject Alternative Name) — CURRENT standard   │
│ X509v3 Subject Alternative Name:                     │
│   DNS:kubernetes                                     │
│   DNS:kubernetes.default                             │
│   DNS:kubernetes.default.svc                         │
│   DNS:kubernetes.default.svc.cluster.local           │
│   IP Address:10.96.0.1    ← Service ClusterIP        │
│   IP Address:172.31.5.2   ← Node/Master IP           │
│   IP Address:54.23.11.5   ← Load Balancer IP         │
└─────────────────────────────────────────────────────┘

Since Go 1.15 (Kubernetes is written in Go):
→ CN field is COMPLETELY IGNORED for hostname verification
→ ONLY SANs are checked
→ If your IP or DNS is not in SANs → connection fails
   regardless of what CN says
```

**Why This Causes Production Incidents**

```
SCENARIO: You add a Network Load Balancer to your cluster.
          External clients now connect via 54.23.11.5.

PROBLEM:  The API server cert was issued BEFORE the NLB
          existed. 54.23.11.5 is not in the SANs.

RESULT:   Every external connection fails with:
          x509: certificate is valid for 172.31.5.2,
          10.96.0.1, not 54.23.11.5

TIMELINE OF FAILURE:
  Monday:  NLB added → cert unchanged
  Monday:  Developers connect via NLB → TLS error
  Monday:  Incident raised, 2 hours to diagnose
  Monday:  Cert regenerated with NLB IP in SANs → fixed

WHAT SHOULD HAVE HAPPENED:
  Before adding NLB → update cert SANs → then add NLB
```

**Diagnosis Commands (Know These Cold)**

```
# 1. Check what SANs are currently in the cert
openssl s_client -connect <api-server-ip>:6443 2>/dev/null \
  | openssl x509 -noout -text \
  | grep -A5 "Subject Alternative Name"

# Expected good output:
# X509v3 Subject Alternative Name:
#   DNS:kubernetes, DNS:kubernetes.default,
#   DNS:kubernetes.default.svc,
#   DNS:kubernetes.default.svc.cluster.local,
#   IP Address:10.96.0.1, IP Address:172.31.5.2

# 2. Check cert expiry at the same time
openssl s_client -connect <api-server-ip>:6443 2>/dev/null \
  | openssl x509 -noout -dates

# 3. Verify the cert chain is valid
openssl verify -CAfile /etc/kubernetes/pki/ca.crt \
  /etc/kubernetes/pki/apiserver.crt

# 4. Test exact TLS error a client sees
curl -v --cacert /etc/kubernetes/pki/ca.crt \
  https://54.23.11.5:6443/healthz
# Look for: SSL: certificate subject name does not match
```

**Fix — Regenerate with Correct SANs**

```
# Backup existing cert first
cp /etc/kubernetes/pki/apiserver.crt \
   /etc/kubernetes/pki/apiserver.crt.bak
cp /etc/kubernetes/pki/apiserver.key \
   /etc/kubernetes/pki/apiserver.key.bak

# Regenerate API server cert with new SANs
kubeadm init phase certs apiserver \
  --apiserver-cert-extra-sans=54.23.11.5,api.mycluster.com

# Verify new cert has the SAN you added
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -A5 "Subject Alternative Name"

# Restart API server (it's a static pod — move manifest out/in)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Verify API server is back
kubectl get nodes
```

**Complete SAN Checklist for API Server Cert**

```
Every API server cert MUST include ALL of these:

DNS SANs:
  ✓ kubernetes
  ✓ kubernetes.default
  ✓ kubernetes.default.svc
  ✓ kubernetes.default.svc.cluster.local
  ✓ <master-hostname>        e.g. master-1
  ✓ <any custom DNS name>    e.g. api.mycluster.com

IP SANs:
  ✓ 10.96.0.1               ← Kubernetes service ClusterIP
  ✓ <control plane node IP>  ← Node's private IP
  ✓ <load balancer IP>       ← Any LB in front of API server
  ✓ 127.0.0.1               ← Localhost (for local kubectl)

COMMON MISS: Load balancer IP added AFTER cert issuance
COMMON MISS: New control plane node in HA cluster not added
COMMON MISS: Elastic IP changed during infrastructure migration
```

**Model Answer**

"SANs — Subject Alternative Names — are the fields in an X.509 certificate that list all the DNS names and IP addresses the certificate is valid for. Since Go 1.15, the CN field is completely ignored for hostname verification. Only SANs are checked.

This causes production incidents when infrastructure changes outpace certificate updates. The most common case: a load balancer is added in front of the API server. Its IP is not in the existing cert's SANs. Every connection via that LB fails with 'certificate is valid for X, not Y'.

To diagnose, I'd run openssl s_client against the endpoint and pipe to openssl x509 -noout -text to read the SAN extension. I'd compare what's there against every IP and DNS name clients are using to reach the server.

To fix on a kubeadm cluster, I'd regenerate the API server cert using kubeadm init phase certs apiserver with the --apiserver-cert-extra-sans flag including the missing IP or hostname, then restart the API server static pod.

The prevention is to treat the SAN list as infrastructure-as-code — managed in Terraform or Ansible — so any infrastructure change that adds a new endpoint triggers an automatic cert regeneration."

## QUESTION 3 — cert-manager Internals

**"How does cert-manager work under the hood?"**

```
👤 INTERVIEWER:
"We use cert-manager for all our certificate management.
A junior engineer asks you: 'when I apply a Certificate
object, what actually happens?' Walk me through the
complete internal flow. Also — what is the difference
between HTTP-01 and DNS-01 challenges and when do you
use each?"
```

**cert-manager Object Hierarchy**

```
YOU CREATE:
  ClusterIssuer    "HOW to get certs signed"
  Certificate      "WHAT cert I want"

CERT-MANAGER CREATES INTERNALLY:
  CertificateRequest  "The actual CSR sent to issuer"
  Order               "An ACME order for the domains"
  Challenge           "The verification per domain"

FINAL OUTPUT:
  Secret (type: kubernetes.io/tls)
    tls.crt  → signed certificate + chain
    tls.key  → private key
```

**The Complete Internal Flow**

```
YOU:  kubectl apply -f certificate.yaml
            │
            ▼
CERT-MANAGER CERTIFICATE CONTROLLER
  Watches all Certificate objects
  Checks: does Secret 'myapp-tls' exist AND is it valid?
            │
       ┌────┴─────────┐
    YES (valid)    NO / missing / expiring
       │               │
       ▼               ▼
  Watch for        Generate new private key
  expiry           Create CertificateRequest object
                         │
                         ▼
               CERTIFICATEREQUEST CONTROLLER
               Converts to CSR (PEM format)
               Routes to correct issuer
                         │
                ┌────────┴────────┐
             ACME Issuer      Internal CA Issuer
             (Let's Encrypt)       │
                │            Sign directly with
                ▼            CA private key
             ORDER CONTROLLER    │
             Create Order        │
             with ACME API       │
                │                │
                ▼                │
           CHALLENGE CONTROLLER  │
           One Challenge         │
           per domain            │
                │                │
       ┌────────┴────────┐       │
    HTTP-01           DNS-01     │
       │                │        │
       ▼                ▼        │
  Create Ingress   Create DNS    │
  rule serving     TXT record    │
  token at         at _acme-     │
  /.well-known/    challenge.    │
  acme-challenge/  domain.com    │
       │                │        │
       └────────┬────────┘       │
                ▼                │
         LE/CA verifies          │
         challenge               │
                │                │
                ▼                │
         Order finalized ────────┘
                │
                ▼
         cert-manager stores result in Secret:
           tls.crt = signed cert + intermediate CA
           tls.key = private key
                │
                ▼
         Ingress/Gateway reads Secret
         Serves HTTPS to users
                │
                ▼
         cert-manager watches expiry
         At renewBefore threshold → start over
```

**HTTP-01 vs DNS-01 — Every Detail**

```
HTTP-01 CHALLENGE
──────────────────────────────────────────────────────
Concept:
  "Prove you control this domain by serving a
   specific token at a specific URL"

What happens:
  1. LE says: "Serve this token at
     http://myapp.example.com/.well-known/
     acme-challenge/abc123xyz"
  2. cert-manager creates a temporary Ingress rule
     pointing /.well-known/acme-challenge/* to a
     solver pod
  3. LE's servers fetch the URL from the internet
  4. Token matches → domain ownership proven
  5. cert-manager deletes the temporary Ingress

Requirements:
  → Domain must be publicly reachable
  → Port 80 must be open from internet
  → Ingress controller must be running

Does NOT work for:
  → Wildcard certs (*.example.com)
  → Private/internal domains
  → Environments without public HTTP access

Use when:
  → Simple public-facing domains
  → You don't have DNS API access
  → myapp.example.com (single domain)


DNS-01 CHALLENGE
──────────────────────────────────────────────────────
Concept:
  "Prove you control this domain by creating a
   specific DNS TXT record"

What happens:
  1. LE says: "Create this DNS TXT record:
     _acme-challenge.myapp.example.com
     value: 'abc123xyz'"
  2. cert-manager calls Route53/Cloudflare/etc API
     to create the TXT record
  3. cert-manager waits for DNS propagation
     (60-300 seconds typically)
  4. LE's servers do a DNS lookup for the TXT record
  5. Value matches → domain ownership proven
  6. cert-manager deletes the TXT record

Requirements:
  → DNS provider API access (Route53, Cloudflare, etc.)
  → IAM/API credentials for cert-manager

Works for:
  → Wildcard certs (*.example.com) ✓
  → Private/internal domains ✓
  → Air-gapped environments ✓
  → Any domain regardless of HTTP access

Use when:
  → *.example.com wildcard cert
  → Internal domain not reachable from internet
  → You have DNS provider API access

CRITICAL RULE:
  Wildcard certs REQUIRE DNS-01
  This is an ACME protocol specification — not configurable
  HTTP-01 proves control of one specific hostname
  DNS-01 proves control of the entire zone
```

**Production cert-manager YAML**

```
# STEP 1: ClusterIssuer (one per cluster, shared)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@company.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:                      # For regular public domains
        ingress:
          class: nginx
    - selector:
        matchLabels:
          use-dns-solver: "true"
      dns01:                       # For wildcards / private domains
        route53:
          region: eu-west-1
          hostedZoneID: ZXXXXXXXXX
          accessKeyIDSecretRef:
            name: route53-creds
            key: access-key-id
          secretAccessKeySecretRef:
            name: route53-creds
            key: secret-access-key
---
# STEP 2: Certificate (one per app/domain)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  duration: 2160h         # 90 days
  renewBefore: 720h       # Start renewing 30 days before expiry
  privateKey:
    algorithm: ECDSA      # Faster + smaller than RSA
    size: 256
    rotationPolicy: Always # New private key on every renewal
  dnsNames:
  - myapp.example.com
  - "*.api.example.com"   # Wildcard — uses DNS-01
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**Debugging cert-manager (Know These Commands)**

```
# Check Certificate status
kubectl get certificate -n production
kubectl describe certificate myapp-cert -n production
# Look for: Ready=True, and the reason if not ready

# Check CertificateRequest (created automatically)
kubectl get certificaterequest -n production
kubectl describe certificaterequest myapp-cert-xxxxx -n production

# Check Order (for ACME issuers)
kubectl get order -n production
kubectl describe order myapp-cert-xxxxx -n production
# Look for: State: valid (success) or State: errored

# Check Challenge (the actual verification step)
kubectl get challenge -n production
kubectl describe challenge myapp-cert-xxxxx-0 -n production
# Common issue: HTTP-01 Ingress not reachable
# Common issue: DNS-01 propagation timeout

# Check cert-manager logs
kubectl logs -n cert-manager \
  deploy/cert-manager -f | grep -i error

# Manually check what's in the resulting Secret
kubectl get secret myapp-tls -n production \
  -o jsonpath='{.data.tls\.crt}' \
  | base64 -d \
  | openssl x509 -noout -text \
  | grep -E "Subject:|Not After|DNS:|IP:"
```

**Let's Encrypt Rate Limits (Know These)**

```
PRODUCTION LIMITS:
  50 certificates per registered domain per week
  5 duplicate certs per week
  5 failed validations per account per hour

STAGING LIMITS:
  Much higher — designed for testing

GOLDEN RULE:
  ALWAYS use staging for development and testing:
  server: https://acme-staging-v02.api.letsencrypt.org/directory

  Staging issues FAKE certs (not trusted by browsers)
  but the entire flow works identically.

  If you hit production rate limits → wait 7 days.
  No override, no support ticket.
  This is why staging exists.
```

**Model Answer**

"When a Certificate object is applied, cert-manager's certificate controller detects it and checks if the target Secret already exists with a valid, non-expiring cert. If not, it generates a new private key and creates a CertificateRequest object. The certificaterequest controller picks this up, creates a CSR, and routes it to the referenced issuer.

For an ACME issuer like Let's Encrypt, the order controller creates an Order with the ACME API, and the challenge controller creates individual Challenge objects per domain. For HTTP-01, it creates a temporary Ingress rule and a solver pod that serves the validation token. For DNS-01, it calls the DNS provider API to create a TXT record at _acme-challenge.domain.com. Once the ACME server verifies the challenge, the order completes, the cert is issued, and cert-manager stores it in a Secret with fields tls.crt and tls.key. cert-manager then continuously monitors expiry and starts renewal at the renewBefore threshold.

HTTP-01 proves control of a specific hostname by serving a token over HTTP — it requires public internet access and cannot issue wildcard certs. DNS-01 proves control of the entire DNS zone by creating a TXT record — it works for wildcards, private domains, and internal infrastructure. Wildcards are DNS-01 only — that's an ACME protocol requirement, not a cert-manager limitation.

In production I always use staging first to avoid hitting the 50 certs per domain per week rate limit, and I set renewBefore to 720 hours — 30 days — to give enough runway if cert-manager has issues."

## QUESTION 4 — Production Incident

**"Your API server cert expired at 2 AM. Walk me through recovery."**

```
👤 INTERVIEWER:
"It's 2 AM. You get paged. The on-call engineer reports
that kubectl is returning errors cluster-wide. Nobody can
run any kubectl commands. You SSH into the master node.
Walk me through exactly how you diagnose this and recover.
What is the most common root cause for this exact symptom?"
```

**Immediate Diagnosis (First 5 Minutes)**

```
# SSH into control plane node
ssh ubuntu@master-1

# Step 1: Can we reach the API server at all?
curl -k https://localhost:6443/healthz
# -k skips cert verification — if this works, API server is up
# If no response → API server process might be down

# Step 2: Check what error kubectl actually gives
kubectl get nodes 2>&1
# Common output:
# Unable to connect to the server:
# x509: certificate has expired or is not yet valid

# Step 3: Confirm it IS a cert issue
openssl s_client -connect localhost:6443 2>/dev/null \
  | openssl x509 -noout -dates
# Output:
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Jan  1 00:00:00 2025 GMT  ← EXPIRED

# Step 4: Check ALL cert expiries at once
kubeadm certs check-expiration
# This shows every cert and its remaining time
# EXPIRED ones show: MISSING or a past date
```

**Recovery Procedure**

```
# RECOVERY STEP 1: Renew all certificates
# (kubeadm CAN renew already-expired certs)
kubeadm certs renew all

# Expected output:
# certificate embedded in kubeconfig for admin
#   cluster-admin in /etc/kubernetes/admin.conf renewed
# certificate for serving the Kubernetes API renewed
# certificate the API server uses to access etcd renewed
# ... (all certs renewed)

# RECOVERY STEP 2: Verify new expiry dates
kubeadm certs check-expiration
# All should now show ~365 days residual time

# RECOVERY STEP 3: Restart control plane static pods
# Static pods are managed by kubelet, not the API server
# They read certs from disk on startup
# Moving the manifest out → kubelet stops the pod
# Moving it back → kubelet starts it fresh with new certs

cd /etc/kubernetes/manifests/

# Restart API server
mv kube-apiserver.yaml /tmp/
sleep 10
mv /tmp/kube-apiserver.yaml .

# Wait for API server to come back
watch kubectl get nodes

# Then restart other components
mv kube-controller-manager.yaml /tmp/
sleep 5
mv /tmp/kube-controller-manager.yaml .

mv kube-scheduler.yaml /tmp/
sleep 5
mv /tmp/kube-scheduler.yaml .

# RECOVERY STEP 4: Update your local kubeconfig
# The admin.conf was also renewed
sudo cp /etc/kubernetes/admin.conf ~/.kube/config

# RECOVERY STEP 5: Verify everything works
kubectl get nodes
kubectl get pods -A
kubectl get cs  # component statuses

# RECOVERY STEP 6: If HA cluster, repeat on ALL control plane nodes
# Do one node at a time
# Verify cluster health between each node
```

**The Full Incident Timeline**

```
02:00  Paged. On-call reports kubectl failures cluster-wide.

02:03  SSH to master-1. Run: kubeadm certs check-expiration
       Confirm: apiserver cert expired 2 hours ago.

02:05  Run: kubeadm certs renew all
       All certs renewed in ~30 seconds.

02:06  Restart kube-apiserver static pod.
       Wait ~60 seconds for it to come back.

02:08  Test: kubectl get nodes → nodes are Ready
       API server is back. Incident resolved.

02:10  Repeat on master-2 and master-3 (HA cluster)
       Verify etcd quorum after each.

02:20  Send incident update: "API server certs expired.
       Renewed and restarted. All components healthy."

02:30  Write post-mortem action items (see below).
```

**Post-Mortem Action Items**

```
ROOT CAUSE:
  kubeadm default cert validity = 1 year.
  No monitoring was in place for cert expiry.
  Cluster silently hit its first birthday.

IMMEDIATE ACTIONS:
  1. Add cert expiry monitoring NOW:

# Prometheus alert
- alert: KubeadmCertExpiringIn30Days
  expr: |
    (kubeadm_cert_expiry_timestamp_seconds - time())
    < (30 * 24 * 3600)
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Control plane cert expiring in < 30 days"

- alert: KubeadmCertExpiringIn7Days
  expr: |
    (kubeadm_cert_expiry_timestamp_seconds - time())
    < (7 * 24 * 3600)
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "CRITICAL: Control plane cert expiring in < 7 days"

  2. Add to monthly runbook:
     "Run kubeadm certs check-expiration on all clusters"

  3. Consider extending cert validity at next renewal:
     kubeadm certs renew all --config kubeadm-config.yaml
     (set certValidity in config to extend beyond 1 year)

  4. Automate via CronJob or external monitoring script.
```

**What If You Can't SSH to the Master?**

```
WORST CASE: Cert expired + no SSH access

Option 1: Emergency SSH via cloud provider console
  AWS: EC2 Systems Manager Session Manager
  GCP: Cloud Shell / OS Login
  Azure: Bastion Host

Option 2: If etcd is still accessible
  Restore from a recent etcd snapshot
  etcdctl snapshot restore <backup>
  This restores cluster state but not the cert fix

Option 3: Nuke and rebuild (last resort)
  Restore from infrastructure-as-code
  Rejoin worker nodes

LESSON: Always have out-of-band access to control plane.
        SSH key backup in secrets manager.
        Cloud provider console access for emergencies.
```

**Model Answer**

"The most common cause of cluster-wide kubectl failure is an expired control plane certificate — specifically the API server cert. kubeadm defaults to 1-year validity and many teams forget this.

First I'd SSH to the master node and run kubeadm certs check-expiration to confirm. I'd also run openssl s_client against localhost:6443 to read the cert dates directly.

If it's expired, the recovery is: kubeadm certs renew all — this renews all control plane certs even if they've already expired. Then I restart each static pod by moving its manifest out of /etc/kubernetes/manifests/ and back in. kubelet detects the change and restarts the pod, which reads the new certs on startup. In an HA cluster I do this one node at a time and verify cluster health between each.

The post-mortem action items: add Prometheus alerts for cert expiry at 30 days and 7 days, add this to the monthly ops runbook, and potentially automate the annual renewal.

The broader lesson is that kubeadm cert expiry is a silent killer — no warnings, no degraded state. One day it's fine, the next day it's completely dead. It must be in your monitoring pipeline."

## QUESTION 5 — Security Breach

**"A worker node has been compromised. What do you do?"**

```
👤 INTERVIEWER:
"Security team alerts you: a worker node running production
workloads is suspected to be compromised. An attacker may
have root access to that node. From a certificates and
secrets perspective, what is at risk, what is your immediate
response, and how do you prevent lateral movement?"
```

**What Lives on a Worker Node**

```
WHAT AN ATTACKER WITH ROOT ACCESS CAN READ:

/var/lib/kubelet/pki/
  kubelet-client-current.pem    ← Kubelet's identity cert + key
                                  CN=system:node:worker-1
                                  O=system:nodes
                                  → Can make API calls as this node

/etc/kubernetes/
  bootstrap-kubelet.conf        ← Bootstrap token (if not cleaned up)

/var/lib/kubelet/pods/*/        ← All pod specs and volumes
  volumes/kubernetes.io~secret/ ← ALL SECRETS MOUNTED IN PODS
  volumes/kubernetes.io~projected/ ← Service Account tokens

Memory (if attacker has root):
  → Can dump process memory
  → Read any secret any pod has in its environment
  → Read any in-memory encryption keys

WHAT THEY CANNOT ACCESS:
  /etc/kubernetes/pki/ca.key   ← Only on control plane
  Other nodes' kubelet certs   ← Per-node, not shared
  etcd data directly           ← Only if etcd is on this node
  Other namespaces' secrets    ← Unless pods on this node use them
```

**Immediate Response (The First 15 Minutes)**

```
# MINUTE 1-2: ISOLATE the node
# Do NOT delete it yet — preserve evidence

# Cordon: prevent new pods from scheduling here
kubectl cordon worker-3

# Label it for investigation tracking
kubectl label node worker-3 security-incident=true

# MINUTE 3-5: DRAIN existing pods
# This moves workloads away from compromised node
kubectl drain worker-3 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# MINUTE 5-10: REVOKE node identity
# Delete node object — its kubelet cert still works
# but the Node Authorizer limits what it can do
kubectl delete node worker-3

# If you want to immediately prevent the node's
# kubelet cert from being used, you need to
# invalidate it via cert rotation:
# The cert itself can't be "revoked" easily in K8s
# (no built-in CRL/OCSP). The practical mitigation
# is deleting the node object + network isolation.

# MINUTE 10-15: NETWORK ISOLATION
# Via cloud provider security group — block all
# traffic from this node's IP except to your
# forensic bastion host
aws ec2 revoke-security-group-ingress \
  --group-id sg-cluster \
  --protocol all \
  --source-group sg-worker-3
# (or equivalent for your cloud provider)
```

**What to Rotate and Why**

```
ROTATE IMMEDIATELY:

1. All Service Account tokens for pods that ran on this node
   → Attacker may have read them from pod volume mounts
   → Rotate by deleting the SA token Secret:
     kubectl delete secret <sa-token-secret> -n <namespace>
     K8s auto-creates a new one

2. All application secrets that were mounted into pods
   on this node
   → grep pod specs for which Secrets were mounted
   → Rotate the actual credentials (DB passwords, API keys)
   → Update the K8s Secrets with new values

3. The node's kubelet cert
   → If you're re-adding the node, it gets a new cert
   → Old cert is useless once node object is deleted
   → Node Authorizer won't honor it anyway

DO NOT NEED TO ROTATE:
   → Cluster CA cert/key (not on worker nodes)
   → Other nodes' kubelet certs (per-node, not shared)
   → etcd certs (not on worker nodes unless etcd is there)

CHECK IF etcd WAS ON THIS NODE:
   → If this was a single-node cluster or etcd node:
     CRITICAL — treat as full cluster compromise
     Rotate everything including CA certs
     Consider rebuilding the cluster
```

**Audit — Find the Blast Radius**

```
# Find ALL pods that ran on the compromised node
kubectl get pods -A \
  --field-selector spec.nodeName=worker-3 \
  -o json | jq '.items[].metadata | 
  {name: .name, namespace: .namespace}'

# For each pod, find what Secrets it mounted
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.volumes[*].secret.secretName}'

# Find what ServiceAccounts those pods used
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}'

# Check API audit logs for that node's kubelet cert
# Look for API calls from system:node:worker-3
# AFTER the suspected compromise time
# Any calls outside normal kubelet operations are attacker activity
grep "system:node:worker-3" /var/log/kubernetes/audit.log \
  | grep -v '"verb":"watch"' \
  | grep -v '"verb":"list"'

# Check if the attacker used the kubelet cert
# to access OTHER nodes' resources
# (Node Authorizer should have prevented this
#  but verify in audit logs)
```

**Lateral Movement Prevention**

```
WITHOUT mTLS (network policy only):
  Compromised pod can call any other pod
  that network policy allows
  → Attacker gets in-cluster network access
  → Can probe other services

WITH mTLS (Istio STRICT):
  Compromised pod's service account cert
  is checked against AuthorizationPolicy
  Even if attacker calls payments-service,
  the cert says they're the compromised pod's SA
  → AuthorizationPolicy blocks it
  → No lateral movement via service calls

This is the production argument for mTLS:
  Network policy = perimeter security
  mTLS = zero-trust identity (works even
         after perimeter is breached)
```

**Model Answer**

"From a certificate and secrets perspective, a compromised worker node exposes several things. The node's kubelet client certificate in /var/lib/kubelet/pki/ gives system:node access — limited by the Node Authorizer to only access resources related to pods on that node. More critically, the attacker has access to every Secret mounted into every pod that ran on that node, every service account token, and potentially in-memory secrets from running processes.

My immediate response: cordon and drain the node to move workloads away, delete the node object to de-register it from the cluster, and use cloud provider security groups to network-isolate the node immediately.

Then I'd audit the blast radius — find every pod that ran on that node, every Secret it mounted, every service account it used. Rotate all of those credentials: delete SA token Secrets so Kubernetes generates fresh ones, and rotate the actual underlying credentials for any application secrets.

What I don't need to rotate: the cluster CA, other nodes' certs, or etcd certs — those don't live on worker nodes.

The important exception: if etcd was running on this node — single-node cluster or combined etcd/control-plane topology — treat it as full cluster compromise. Rotate everything and consider rebuilding.

Long-term, this is exactly the argument for service mesh mTLS in STRICT mode. With network policy alone, a compromised pod can call any other pod within policy rules. With mTLS, even if the attacker compromises a pod, its calls to other services carry the cryptographic identity of that service account, which AuthorizationPolicies can restrict. No lateral movement even after a breach."

## QUESTION 6 — mTLS Deep Dive

**"Design mTLS for 400 microservices. What can go wrong?"**

```
👤 INTERVIEWER:
"We want to roll out mTLS across all 400 of our microservices.
We're using Istio. Walk me through how you'd do this safely,
what PERMISSIVE mode means and why it's dangerous to leave it
enabled, and what are the failure modes that have killed
production for teams doing this rollout."
```

**How mTLS Works in Istio (The Full Picture)**

```
BEFORE ISTIO (plain HTTP between services):

payments-pod ──── HTTP ──── accounts-pod
                           "Who are you?"
                           "I don't know,
                            you're at 10.0.0.5"

AFTER ISTIO with mTLS:

┌─────────────────────────────────────────────────────┐
│  payments-pod         │         accounts-pod         │
│                       │                              │
│  App code             │         App code             │
│  (talks HTTP :8080)   │         (talks HTTP :8080)   │
│         │             │                 ▲            │
│         ▼             │                 │            │
│  Envoy sidecar        │mTLS    Envoy sidecar         │
│  :15001 outbound      │───────▶:15006 inbound        │
│                       │                              │
│  SPIFFE cert:         │         Verifies peer cert   │
│  spiffe://cluster/    │         Extracts identity    │
│  ns/prod/sa/          │         Enforces AuthzPolicy │
│  payments-sa          │                              │
└─────────────────────────────────────────────────────┘

Your app code NEVER changes.
Envoy intercepts all traffic transparently.
Istiod provisions and rotates certs every 24h.
```

**The Safe Rollout Strategy**

```
NEVER enable STRICT mTLS across the whole cluster at once.
Some services may not have sidecars yet.
STRICT mode rejects all plaintext → those services break.

THE SAFE ROLLOUT (4 phases):

PHASE 1: Install Istio, default mode = PERMISSIVE
  All services work (mTLS and plaintext both accepted)
  No impact to existing traffic
  Sidecars not injected yet

PHASE 2: Enable sidecar injection namespace by namespace
  kubectl label namespace production \
    istio-injection=enabled
  Rolling restart pods to inject sidecars:
  kubectl rollout restart deployment -n production
  Verify: all pods show 2/2 containers (app + envoy)
  Traffic between meshed pods is now mTLS automatically
  Traffic from non-meshed pods is still plaintext → ALLOWED
  (because mode is still PERMISSIVE)

PHASE 3: Verify all services are meshed
  # Check which pods DON'T have sidecars yet
  kubectl get pods -A \
    -o json | jq '.items[] |
    select(.spec.containers | length == 1) |
    {name: .metadata.name,
     ns: .metadata.namespace}'

PHASE 4: Switch to STRICT only after ALL pods are meshed
  Apply STRICT PeerAuthentication per namespace first:
  Test → then cluster-wide

  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: production    ← namespace-level first
  spec:
    mtls:
      mode: STRICT

  Monitor errors for 24 hours.
  Then promote to cluster-wide:
  metadata:
    namespace: istio-system  ← cluster-wide
```

**PERMISSIVE Mode — Why It's Dangerous**

```
PERMISSIVE MODE BEHAVIOR:

  meshed-pod ────── mTLS ──────▶ accounts-service ✓
  non-meshed-pod ── HTTP ──────▶ accounts-service ✓  ← ALSO ALLOWED

WHAT THIS BREAKS:

  AuthorizationPolicy:
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: accounts-policy
  spec:
    selector:
      matchLabels:
        app: accounts-service
    rules:
    - from:
      - source:
          principals:
          - "cluster.local/ns/prod/sa/payments-sa"

  THIS POLICY IS UNENFORCED IN PERMISSIVE MODE

  Why? source.principal comes from the mTLS cert.
  For plaintext callers → no cert → no principal.
  principal is empty string.
  The policy's "from" clause doesn't match.
  Istio's behavior for no-match:
    → if no DENY policy matches AND no ALLOW policy matches
    → DEFAULT: ALLOW

  RESULT: Unknown pods can bypass AuthorizationPolicies
          You think you have zero-trust → you don't

DETECTION:
  # Check if any AuthorizationPolicies are being bypassed
  kubectl exec -it debug-pod -n production -- \
    curl http://accounts-service/api/v1/balance
  # If this works from a non-meshed debug pod
  # you're in PERMISSIVE mode or have no authz policy

CORRECT STATE:
  STRICT mode + AuthorizationPolicies = zero-trust ✓
  PERMISSIVE mode + AuthorizationPolicies = false sense
                                             of security ✗
```

**Production Failure Modes in mTLS Rollout**

```
FAILURE 1: Injecting sidecars into system namespaces
  DON'T inject into: kube-system, kube-public, cert-manager
  These components don't expect mTLS
  API server health checks break
  etcd client connections break
  FIX: Label these namespaces to SKIP injection:
    kubectl label namespace kube-system \
      istio-injection=disabled

FAILURE 2: DaemonSets missed in rolling restart
  kubectl rollout restart deployment works
  DaemonSets are separate:
  kubectl rollout restart daemonset -n production
  If missed → DaemonSet pods have no sidecar
  → STRICT mode breaks communication with them

FAILURE 3: CronJobs and Jobs
  Short-lived jobs complete before sidecar injection
  Sidecar takes ~3 seconds to start
  Job finishes → sidecar never started
  In STRICT mode: job can't make any outbound calls
  FIX: Add annotation to skip sidecar for jobs:
    annotations:
      sidecar.istio.io/inject: "false"
  Or use holdApplicationUntilProxyStarts: true

FAILURE 4: Health checks failing after STRICT mode
  Kubernetes liveness/readiness probes come from
  the kubelet → not a meshed component → plaintext
  In STRICT mode: kubelet health check → rejected
  Pods continuously crash with probe failures
  FIX: Istio automatically handles this since 1.9+
  by rewriting probes to go through the sidecar
  Verify: your Istio version supports this

FAILURE 5: Database connections break
  External DBs (RDS, Cloud SQL) are NOT meshed
  If you put a database client pod in STRICT mode
  and it tries to talk to external RDS → works fine
  (STRICT is about INCOMING connections to the pod)
  But if you misconfigure DestinationRule to require
  mTLS for external services → connections fail
  FIX: Use DestinationRule with tls mode DISABLE
  for external services outside the mesh

FAILURE 6: Sidecars not ready before app container
  Race condition on pod startup
  App starts, tries to connect out
  Envoy sidecar not initialized yet
  Connection fails → app crashes → CrashLoopBackOff
  FIX:
    annotations:
      proxy.istio.io/config: |
        holdApplicationUntilProxyStarts: true
```

**Model Answer**

"For a safe mTLS rollout across 400 services, I'd use a four-phase approach. First, install Istio with the default PERMISSIVE mode — no traffic impact. Second, enable sidecar injection namespace by namespace with rolling restarts, verify all pods show 2/2 containers. Third, use tooling to confirm every pod has a sidecar. Fourth, switch to STRICT mode per-namespace, monitor for 24 hours, then promote to cluster-wide.

PERMISSIVE mode is dangerous to leave enabled long-term because it silently breaks AuthorizationPolicies. AuthorizationPolicies evaluate source.principal, which comes from the peer's mTLS certificate. For plaintext callers — non-meshed pods — there's no cert, so principal is empty. The policy's source check doesn't match, and Istio's default is to allow. So you have what looks like zero-trust enforcement but isn't — unknown pods bypass all your authorization policies.

The worst production failure modes I've seen: injecting sidecars into system namespaces and breaking API server health checks, missing DaemonSets in the rolling restart so they enter STRICT mode without sidecars, and the liveness probe issue where kubelet probes come from outside the mesh and get rejected. Istio 1.9+ handles probe rewriting automatically, but you need to verify your version supports it.

The general principle: never switch to STRICT until you have 100% sidecar coverage and have tested that health checks, CronJobs, and external service connections all work correctly."

## QUESTION 7 — System Design

**"Design certificate management for 3 regions, 400 services"**

```
👤 INTERVIEWER:
"Let's go to system design. We run 400 microservices across
3 AWS regions — eu-west-1, us-east-1, ap-southeast-1.
Each region has its own Kubernetes cluster. We need:
  1. Automated cert management for all services
  2. mTLS between ALL services including cross-region
  3. External HTTPS for customer-facing APIs
  4. Zero downtime cert rotation
  5. A cert compromise doesn't affect other regions

Design this system. Talk me through your architecture
decisions and trade-offs."
```

**The Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                   GLOBAL LAYER                               │
│                                                              │
│  AWS Route53    ←── DNS routing for external traffic         │
│  AWS ACM        ←── Public certs for customer-facing APIs   │
│  SPIRE Server   ←── Federated SPIFFE identity across regions │
│  HashiCorp Vault ←── Internal CA, secret storage            │
└──────────────────────────────┬──────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   EU-WEST-1     │  │   US-EAST-1     │  │  AP-SOUTHEAST-1 │
│   K8s Cluster   │  │   K8s Cluster   │  │   K8s Cluster   │
│                 │  │                 │  │                 │
│ cert-manager    │  │ cert-manager    │  │ cert-manager    │
│ Istio           │  │ Istio           │  │ Istio           │
│ SPIRE Agent     │  │ SPIRE Agent     │  │ SPIRE Agent     │
│ Vault Agent     │  │ Vault Agent     │  │ Vault Agent     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

**Decision 1 — Separate CA Per Region**

```
OPTION A: One global CA for all regions
  Pros: Simpler, one place to manage
  Cons: One CA compromise = all 3 regions compromised
        Single point of failure for cert issuance
        Cross-region CA calls add latency

OPTION B: One CA per region (CHOOSE THIS)
  Pros: Blast radius contained to one region
        Regional autonomy — cert issuance works even
        if inter-region connectivity is down
        Meets data residency requirements (EU certs
        signed by EU CA)
  Cons: Cross-region mTLS needs federation

IMPLEMENTATION:
  Each region has its own intermediate CA
  All intermediate CAs signed by a single root CA
  Root CA is offline (not connected to anything)
  Only used to sign new intermediate CAs

  Root CA (offline, HSM-backed)
      │
      ├── EU Intermediate CA  (Vault cluster in eu-west-1)
      ├── US Intermediate CA  (Vault cluster in us-east-1)
      └── AP Intermediate CA  (Vault cluster in ap-southeast-1)

  A compromise of the EU Intermediate CA:
  → Revoke EU intermediate cert
  → Issue new EU intermediate from root
  → US and AP are completely unaffected
```

**Decision 2 — External Traffic (Customer-Facing APIs)**

```
EXTERNAL HTTPS ARCHITECTURE:

Internet → Route53 → ALB (per region) → Ingress → Service → Pod

CERTIFICATE STRATEGY:
  Use AWS ACM for ALB certs
  ACM is free, auto-renews, integrated with ALB
  cert-manager is NOT needed for this layer

  ALB terminates TLS → HTTP to Ingress controller
  Ingress controller routes to services (HTTP)
  mTLS handles encryption INSIDE the cluster

  EXCEPTION: If you need end-to-end encryption
  (compliance requirement), then:
  ALB (TLS passthrough) → Nginx (TLS passthrough)
  → Pod terminates TLS (cert-manager cert in Secret)
  But this is complex and rarely needed if your
  internal mTLS is solid

MULTI-REGION DNS:
  Route53 latency-based routing:
  api.company.com → eu-west-1 ALB (if user is in Europe)
  api.company.com → us-east-1 ALB (if user is in US)

  Each region's ACM cert covers:
  api.company.com (same domain, different cert per region)
  Wildcard: *.company.com per region
```

**Decision 3 — Internal Service-to-Service (mTLS)**

```
INTRA-REGION mTLS:
  Istio with STRICT PeerAuthentication per cluster
  cert-manager issues service certs from regional CA
  OR Istio's built-in CA (Istiod) issues SPIFFE SVIDs
  Automatic — no app changes needed

CROSS-REGION mTLS:
  Services in EU calling services in US need:
  1. Network path (VPC peering or Transit Gateway)
  2. Mutual trust between CAs

  SOLUTION: SPIFFE Federation

  SPIRE Server in each region registers with others:
  eu-west-1 SPIRE Server trusts us-east-1 SPIFFE root
  us-east-1 SPIRE Server trusts eu-west-1 SPIFFE root

  Now EU pod with spiffe://eu.cluster/ns/prod/sa/payments-sa
  can call US pod with spiffe://us.cluster/ns/prod/sa/accounts-sa
  Both verify each other's SPIFFE IDs from trusted federation

  AuthorizationPolicy becomes:
    source:
      principals:
      - "spiffe://eu.cluster/ns/prod/sa/payments-sa"
      - "spiffe://us.cluster/ns/prod/sa/payments-sa"
    (allow payments-service from EITHER region)
```

**Decision 4 — Zero Downtime Cert Rotation**

```
WHY ROTATION CAN CAUSE DOWNTIME:
  At rotation time, new cert is issued
  If server restarts before client trusts new CA
  → TLS handshake fails
  → Brief downtime

THE DUAL-CERT PATTERN:

STEP 1: Add new CA to trust bundle of ALL clients
        (don't issue certs with it yet)
        All clients now trust BOTH old and new CA

STEP 2: Start issuing certs from new CA
        Servers present new CA cert
        Clients trust it (added in step 1) → no break

STEP 3: Wait for all old certs to expire naturally
        (or force rotation)

STEP 4: Remove old CA from trust bundle
        All connections now use new CA

CERT-MANAGER HANDLES THIS AUTOMATICALLY:
  renewBefore: 720h  (start renewing 30 days before expiry)
  rotationPolicy: Always  (new key on each renewal)
  Old cert remains valid during renewal period
  → no downtime

FOR CONTROL PLANE CERTS (kubeadm):
  Renew during maintenance window
  Restart one control plane component at a time
  Verify health between each restart
  In HA clusters: active connections survive
  because other control plane nodes continue serving
```

**Decision 5 — Monitoring and Alerting**

```
WHAT TO MONITOR:

1. cert-manager metrics (via Prometheus):
   certmanager_certificate_expiry_seconds{name,namespace}
   certmanager_certificate_ready_status{name,namespace}

   Alert: expiry < 30d → WARNING
   Alert: expiry < 7d  → CRITICAL (page on-call)
   Alert: ready_status = false for > 15min → CRITICAL

2. Control plane cert expiry (kubeadm):
   Custom script or kube-state-metrics exporter
   Run: kubeadm certs check-expiration
   Export remaining days as a metric

3. Ingress/external cert expiry:
   AWS CloudWatch metric for ACM cert expiry
   OR blackbox exporter probing external HTTPS endpoints:
   probe_ssl_earliest_cert_expiry{instance="api.company.com"}

4. mTLS enforcement:
   Istio metrics:
   istio_requests_total{
     response_code="403",
     source_principal=""
   }
   High rate of 403s with empty principal =
   PERMISSIVE mode being abused by non-meshed pods

GRAFANA DASHBOARD:
  Panel 1: All cert expiry dates (30d time range view)
  Panel 2: cert-manager certificate ready status
  Panel 3: mTLS coverage (% pods with sidecars)
  Panel 4: AuthorizationPolicy denial rates
```

## The Full Architecture Summary

```
EXTERNAL TRAFFIC:
  Route53 (latency routing) → ALB → ACM cert (free, auto-renew)
  → Ingress controller → Pod (HTTP inside cluster)

INTERNAL SERVICE TRAFFIC:
  Istio mTLS STRICT across all 3 clusters
  SPIFFE SVIDs issued by regional Istiod/SPIRE
  SPIFFE Federation for cross-region trust
  AuthorizationPolicies enforce service-to-service rules

CERTIFICATE ISSUANCE:
  Internal services: cert-manager + regional Vault intermediate CA
  External services: ACM
  Control plane: kubeadm (renewed annually + monitored)

ISOLATION:
  One intermediate CA per region
  Compromise of one region → rotate that intermediate only
  Root CA offline, HSM-backed

ROTATION:
  cert-manager: automatic at renewBefore threshold
  Control plane: kubeadm renew all during maintenance window
  Zero downtime: dual-cert trust bundle pattern during CA rotation

MONITORING:
  Prometheus + cert-manager metrics
  Alert at 30d and 7d before expiry
  mTLS coverage dashboard in Grafana
```

**Model Answer**

"I'd design this in four layers: external traffic, internal service mesh, cross-region federation, and observability.

For external customer-facing APIs, I'd use ACM certificates on ALBs per region with Route53 latency-based routing. ACM is free, auto-renews, and native to AWS — no cert-manager needed at this layer.

For internal service-to-service traffic, I'd use Istio with STRICT mTLS across all three clusters. Each cluster gets its own intermediate CA, all signed by a single offline root CA. This is the blast radius isolation decision — a compromise in EU affects only the EU intermediate CA, which can be rotated without touching US or AP.

For cross-region mTLS, I'd use SPIFFE Federation via SPIRE. Each region's SPIRE server registers a trust bundle with the others. A service in EU presenting its SPIFFE SVID can be verified by a service in US — they share federated trust. AuthorizationPolicies then specify allowed principals by SPIFFE ID including region prefix.

For zero-downtime rotation, I'd use the dual-cert trust bundle pattern: add the new CA to all client trust bundles first, then start issuing certs from the new CA, wait for old certs to expire, then remove old CA from trust bundles. cert-manager's renewBefore field handles this automatically for service certs.

For monitoring: Prometheus scraping cert-manager's expiry metrics, alerting at 30 days warning and 7 days critical, plus a Grafana dashboard showing mTLS coverage percentage — you want 100% sidecar injection before switching to STRICT."

## QUESTION 8 — Staff Level

**"What would you change about how Kubernetes handles certificates?"**

```
👤 INTERVIEWER:
"Last question — and this one is more open-ended.
If you could redesign how Kubernetes handles certificate
management and workload identity, what would you change?
What are the current limitations, and what does the
bleeding edge look like?

This is less about right/wrong answers and more about
seeing how you think about the evolution of the ecosystem."
```

**Current Limitations (Be Honest About These)**

```
LIMITATION 1: No native certificate revocation
  X.509 certificates have CRL (Certificate Revocation List)
  and OCSP (Online Certificate Stapling Protocol).
  Kubernetes doesn't implement either for its internal certs.

  What this means:
  If a cert is stolen, it remains valid until it expires.
  A stolen kubelet cert with 300 days remaining validity
  → 300 days of valid attacker access.

  Current workaround:
  → Delete the node object (limits what the cert can do)
  → Network isolation
  → Use very short-lived certs (hours, not days)
  → Service meshes rotate every 24h — reduces window

LIMITATION 2: CA private key stored in etcd (via Secrets)
  cert-manager stores private keys in Kubernetes Secrets.
  Kubernetes Secrets are stored in etcd.
  If etcd at-rest encryption is not enabled (it's opt-in),
  all private keys are in plaintext in etcd.
  This means etcd access = full PKI compromise.

  What should happen:
  Private keys generated in HSM, never extractable.
  cert-manager + Vault PKI solves this but adds complexity.

LIMITATION 3: cert-manager is not built-in
  Every cluster needs cert-manager installed and managed.
  It's the de facto standard but it's a separate system.
  A cert-manager outage means no cert renewals.
  K8s doesn't have a native certificate controller
  beyond the basic CSR API.

LIMITATION 4: SPIFFE/mTLS is opt-in and complex
  In an ideal world, every pod would have a SPIFFE identity
  from birth, provisioned by the platform.
  Today, you need a service mesh (Istio/Linkerd) which
  adds operational complexity, upgrade burden, sidecar
  overhead, and debugging difficulty.

LIMITATION 5: kubeadm's 1-year cert expiry
  Default 1-year validity with no warnings.
  Clusters die silently on their first birthday.
  This should default to longer validity +
  built-in monitoring alerts.
```

**Where the Ecosystem Is Heading**

```
TREND 1: Sidecarless service mesh (eBPF)
  CURRENT: Istio injects Envoy sidecar into every pod
           +1 container per pod
           network overhead
           complex debugging

  FUTURE:  Cilium with eBPF enforces mTLS at kernel level
           No sidecar container
           Sub-millisecond overhead
           SPIFFE identity without proxy
           This is what Grafana Labs and Cloudflare run

TREND 2: SPIFFE everywhere as the default
  SPIFFE/SPIRE becoming the standard workload identity layer
  Vault, AWS IAM, GCP Workload Identity all speaking SPIFFE
  Cross-cloud identity federation without custom glue code
  "Your pod's SPIFFE ID is its identity — everywhere"

TREND 3: Short-lived certs replacing long-lived ones
  Vault PKI can issue 1-hour certs
  CSI driver rotates them continuously
  Private key never touches etcd
  Generated in Vault's HSM memory
  If stolen → unusable in 1 hour

  This makes revocation largely unnecessary:
  Cert expires before attacker can exploit it

TREND 4: Keyless signing (Sigstore)
  For container image signing, not pod TLS
  But same principle: ephemeral cert issued at signing time
  tied to OIDC identity (GitHub Actions, etc.)
  No long-lived signing keys to protect

TREND 5: cert-manager becoming more of a standard
  cert-manager moving toward CNCF graduation
  Gateway API + cert-manager becoming the
  standard pattern replacing Ingress + annotations
  More issuers, more enterprise CA integrations
```

**What You Would Actually Change**

```
IF YOU WERE REDESIGNING THIS:

1. SPIFFE identity built into Kubernetes natively
   Every pod gets a SPIFFE SVID automatically
   No service mesh required for basic identity
   kubelet acts as the SPIRE agent
   Issuance is part of pod admission, not a sidecar

2. Short-lived certs as the default
   Default cert TTL: 24 hours (not 1 year)
   Auto-rotation built into every component
   Theft window reduced from 365 days to 24 hours

3. etcd at-rest encryption on by default
   Currently opt-in configuration
   Should be default-on
   Private key compromise requires etcd compromise
   which already requires cluster-admin access

4. Built-in cert monitoring
   kubeadm should emit Prometheus metrics natively
   for cert expiry
   No external monitoring setup required
   Warning events on cert objects when expiry < 30d

5. Native cert revocation
   Kubernetes CertificateSigningRequest objects
   could include a revocation endpoint
   Revoked certs rejected at the admission webhook level
   even before expiry

WHAT EXISTS TODAY THAT GETS CLOSEST:
  → SPIRE + Cilium + short-TTL Vault PKI
    covers items 1, 2, 3
  → cert-manager metrics + Prometheus covers item 4
  → Service mesh with STRICT mTLS + short TTL
    makes revocation less critical (item 5)
```

**Model Answer**

"The biggest limitation in Kubernetes' current certificate model is the lack of native certificate revocation. If a cert or private key is compromised, it remains valid until expiry. With kubeadm's 1-year defaults, that's a very long window. The practical mitigation is short-lived certs — Vault PKI issuing 24-hour certs via the CSI driver, so a stolen cert is worthless within hours. But this should be the default behavior, not something you have to architect separately.

The second limitation is that private keys land in etcd via Kubernetes Secrets. etcd at-rest encryption is opt-in, meaning most clusters have their private keys in plaintext in etcd. Vault PKI solves this — keys are generated inside Vault's memory, never extractable — but again it's additional operational complexity.

The third gap is that SPIFFE identity and mTLS require a service mesh, which is an expensive operational bet — Envoy sidecars add latency, complicate debugging, and create upgrade dependencies. eBPF-based service meshes like Cilium are the answer here: SPIFFE identity and mTLS enforcement at the kernel level with no sidecar. That's where I think the ecosystem is heading — eBPF native mTLS will replace sidecar-based service meshes within 3-5 years for most use cases.

If I were redesigning this from scratch: SPIFFE identity built into the kubelet for every pod at admission time, short-lived certs as the default TTL, etcd encryption on by default, and cert-manager's monitoring capabilities integrated into core Kubernetes as first-class events and metrics. The individual pieces exist today — SPIRE, Vault, Cilium, cert-manager — but they're not integrated. The future state is one coherent identity and certificate layer that just works out of the box."

## MASTER QUICK REFERENCE

**Read This Before The Interview**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE 10 THINGS TO KNOW COLD                   │
├─────────────────────────────────────────────────────────────────┤
│ 1. kubectl auth = client TLS cert. CN=username, O=groups.       │
│    O=system:masters → cluster-admin. Protect ca.key above all.  │
├─────────────────────────────────────────────────────────────────┤
│ 2. SANs only. CN is ignored since Go 1.15. Every IP and DNS     │
│    name must be in SAN or connections fail.                      │
├─────────────────────────────────────────────────────────────────┤
│ 3. Three CAs: k8s CA, etcd CA, front-proxy CA.                  │
│    Blast radius isolation. etcd CA compromise ≠ full takeover.  │
├─────────────────────────────────────────────────────────────────┤
│ 4. kubeadm certs = 1 year default. Silent killer.               │
│    kubeadm certs check-expiration. Alert at 30d.                │
├─────────────────────────────────────────────────────────────────┤
│ 5. cert-manager flow: ClusterIssuer → Certificate →             │
│    CertificateRequest → Order → Challenge → Secret              │
├─────────────────────────────────────────────────────────────────┤
│ 6. HTTP-01: public domains, no wildcards.                        │
│    DNS-01: wildcards + private domains. Requires DNS API.        │
│    Wildcards REQUIRE DNS-01. Non-negotiable.                     │
├─────────────────────────────────────────────────────────────────┤
│ 7. Volume mounts auto-update on Secret change.                   │
│    Env vars DO NOT. Always mount TLS certs as volumes.          │
├─────────────────────────────────────────────────────────────────┤
│ 8. PERMISSIVE mTLS = AuthorizationPolicies unenforced.          │
│    Empty source.principal bypasses all source checks.           │
│    STRICT = production. PERMISSIVE = migration only.            │
├─────────────────────────────────────────────────────────────────┤
│ 9. Node compromise: rotate all SA tokens + app secrets           │
│    for pods that ran there. ca.key is safe (not on workers).    │
├─────────────────────────────────────────────────────────────────┤
│ 10. Future: eBPF mTLS (Cilium), SPIFFE everywhere,              │
│     short-lived certs (24h), Vault PKI for HSM-backed keys.     │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  COMMANDS TO KNOW COLD                           │
├─────────────────────────────────────────────────────────────────┤
│ kubeadm certs check-expiration                                   │
│ kubeadm certs renew all                                          │
│ kubeadm init phase certs apiserver --apiserver-cert-extra-sans= │
│                                                                  │
│ openssl s_client -connect <host>:6443 2>/dev/null               │
│   | openssl x509 -noout -text | grep -A5 "Subject Alt"         │
│                                                                  │
│ kubectl describe certificate <name> -n <ns>                      │
│ kubectl describe order <name> -n <ns>                            │
│ kubectl describe challenge <name> -n <ns>                        │
│                                                                  │
│ kubectl cordon <node>                                            │
│ kubectl drain <node> --ignore-daemonsets --delete-emptydir-data │
│ kubectl delete node <node>                                       │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               FAILURE MODE → ROOT CAUSE → FIX                   │
├─────────────────────────────────────────────────────────────────┤
│ x509: cert valid for X not Y  → Missing SAN                     │
│   Fix: regenerate cert with correct SANs                         │
│                                                                  │
│ kubectl returns EOF / expired  → kubeadm 1yr cert expiry        │
│   Fix: kubeadm certs renew all + restart static pods            │
│                                                                  │
│ ACME rate limit hit            → Too many LE requests            │
│   Fix: use staging server. Wait 7 days for production.          │
│                                                                  │
│ AuthzPolicy not enforced       → PERMISSIVE mTLS mode           │
│   Fix: switch to STRICT after 100% sidecar coverage             │
│                                                                  │
│ Pods CrashLoop after STRICT    → Liveness probes rejected        │
│   Fix: upgrade Istio (1.9+ handles probe rewriting)             │
│                                                                  │
│ cert-manager cert not renewing → cert-manager pod down           │
│   Fix: check cert-manager logs, restart, set renewBefore: 720h  │
└─────────────────────────────────────────────────────────────────┘
```


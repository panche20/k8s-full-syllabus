# Day 20 — Istio Service Mesh Deep Dive

*Day 18 introduced Istio conceptually. Today you go deep — traffic management, observability, security, and the internals that interviewers actually ask about. This is what separates engineers who've read about service meshes from engineers who've operated them.*

## 🧠 Part 1: Why a Service Mesh Exists

*Before Istio, every team solved these problems independently in application code:*

```
Retries           → each team wrote retry logic differently
Timeouts          → inconsistent across services
mTLS              → nobody did it because it was hard
Circuit breaking  → some services had it, most didn't
Distributed trace → every service needed SDK instrumentation
Canary traffic    → required app-level feature flags or separate deployments
```

*Istio moves all of this into the infrastructure layer. Apps stop caring about it. A Go service and a Python service get identical behavior — because the proxy handles it, not the app.*

```
Without mesh:  Service A ──────────────────────→ Service B
With mesh:     Service A → Envoy → mTLS tunnel → Envoy → Service B
                              ↓                      ↓
                         metrics/trace           metrics/trace
                         retry logic             circuit breaker
                         auth policy             authz check
```

## 🏗️ Part 2: Istio Architecture — Every Component

```
Control Plane (istiod):
├── Pilot         — pushes xDS config to all Envoy proxies
├── Citadel       — issues mTLS certificates (SPIFFE/SPIRE)
└── Galley        — validates and distributes config

Data Plane:
└── Envoy sidecar — injected into every pod, intercepts all traffic

Supporting:
├── Ingress Gateway  — replaces nginx-ingress for north-south traffic
├── Egress Gateway   — controls and observes outbound traffic
└── istio-cni        — alternative injection without init containers
```

**How Envoy gets injected**

```
# When a namespace has the label:
kubectl label namespace production istio-injection=enabled

# The MutatingAdmissionWebhook intercepts pod creation
# Adds two containers before the pod starts:
# 1. istio-init (init container) — sets up iptables rules to redirect traffic
# 2. istio-proxy (sidecar) — the Envoy proxy

kubectl get pod url-shortener-xxx -n production -o yaml \
  | grep -E "name:|image:" | grep -v "^--"
# name: url-shortener     ← your app
# name: istio-proxy       ← injected by webhook
# name: istio-init        ← init container for iptables
```

**iptables redirection — how Envoy intercepts without app changes**

```
# istio-init runs this before your app starts:
# All inbound traffic on any port → Envoy port 15006
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006

# All outbound traffic → Envoy port 15001
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001

# Except traffic from Envoy itself (prevents infinite loop)
iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN

# Your app thinks it's talking directly to other services
# It has no idea Envoy is in the middle
```

**xDS — how istiod pushes config to Envoy**

```
xDS = a family of discovery APIs Envoy uses to get config dynamically

LDS  Listener Discovery Service    — what ports to listen on
RDS  Route Discovery Service       — how to route requests
CDS  Cluster Discovery Service     — what backends exist
EDS  Endpoint Discovery Service    — which IPs back each cluster
SDS  Secret Discovery Service      — TLS certificates
```

```
# See what config istiod has pushed to a specific Envoy
istioctl proxy-config all \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# See listeners (what Envoy is listening on)
istioctl proxy-config listeners \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# See routes
istioctl proxy-config routes \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# See clusters (backends)
istioctl proxy-config clusters \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# See endpoints
istioctl proxy-config endpoints \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# Check if Envoy config is in sync with istiod
istioctl proxy-status
# NAME                              CDS    LDS    EDS    RDS    ISTIOD
# url-shortener-xxx.production      SYNCED SYNCED SYNCED SYNCED istiod-xxx
```

## 🚦 Part 3: Traffic Management — Full Depth

**VirtualService — the routing brain**

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: url-shortener
  namespace: production
spec:
  hosts:
  - url-shortener          # applies to traffic going to this K8s Service
  - short.yourdomain.com   # also for external traffic via Gateway

  http:
  # Rule 1: Canary — header-based routing
  - match:
    - headers:
        x-canary:
          exact: "true"
    - headers:
        cookie:
          regex: ".*canary=true.*"
    route:
    - destination:
        host: url-shortener
        subset: v2
      weight: 100

  # Rule 2: Internal traffic gets v1
  - match:
    - sourceLabels:
        app: internal-client
    route:
    - destination:
        host: url-shortener
        subset: v1

  # Rule 3: Default — weighted split
  - route:
    - destination:
        host: url-shortener
        subset: v1
      weight: 90
    - destination:
        host: url-shortener
        subset: v2
      weight: 10

    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,retriable-4xx

    # Timeout
    timeout: 10s

    # Fault injection for chaos testing
    fault:
      delay:
        percentage:
          value: 5.0          # inject 500ms delay to 5% of requests
        fixedDelay: 500ms
      abort:
        percentage:
          value: 1.0          # return 500 to 1% of requests
        httpStatus: 500
```

**DestinationRule — connection and circuit breaking**

```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: url-shortener
  namespace: production
spec:
  host: url-shortener
  trafficPolicy:
    # Connection pool — prevent cascade failure
    connectionPool:
      tcp:
        maxConnections: 100       # max TCP connections to this service
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10    # force reconnect every 10 requests
        maxRetries: 3

    # Circuit breaker — eject unhealthy hosts
    outlierDetection:
      consecutive5xxErrors: 5         # eject after 5 consecutive 5xx
      consecutiveGatewayErrors: 5
      interval: 30s                   # scan interval
      baseEjectionTime: 30s           # how long to eject
      maxEjectionPercent: 50          # never eject more than 50% of endpoints
      minHealthPercent: 30            # stop ejecting if < 30% remain

    # Load balancing algorithm
    loadBalancer:
      simple: LEAST_REQUEST            # ROUND_ROBIN | LEAST_REQUEST | RANDOM | PASSTHROUGH
      # Or consistent hash for stickiness:
      # consistentHash:
      #   httpHeaderName: x-user-id

    # mTLS mode for this specific service
    tls:
      mode: ISTIO_MUTUAL              # use Istio-managed mTLS

  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 50  # v1 gets smaller pool (legacy)

  - name: v2
    labels:
      version: v2
```

**Gateway — north-south traffic (replaces Ingress)**

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: url-shortener-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway       # use the Istio ingress gateway pod
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE              # terminate TLS at gateway
      credentialName: url-shortener-tls   # K8s Secret with cert
    hosts:
    - short.yourdomain.com

  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true       # redirect HTTP → HTTPS
    hosts:
    - short.yourdomain.com
---
# VirtualService connects to the Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: url-shortener-gateway-vs
  namespace: production
spec:
  hosts:
  - short.yourdomain.com
  gateways:
  - url-shortener-gateway       # attach to the Gateway
  - mesh                        # also apply inside the mesh
  http:
  - route:
    - destination:
        host: url-shortener
        port:
          number: 80
```

**ServiceEntry — control egress to external services**

```
# Without this, egress to external services is blocked in REGISTRY_ONLY mode
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-postgres
  namespace: production
spec:
  hosts:
  - postgres.rds.amazonaws.com
  ports:
  - number: 5432
    name: tcp-postgres
    protocol: TCP
  location: MESH_EXTERNAL
  resolution: DNS
---
# ServiceEntry for external HTTP API
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: stripe-api
spec:
  hosts:
  - api.stripe.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

## 🔐 Part 4: Istio Security — mTLS and Authorization

**SPIFFE identity — how pods authenticate**

*Every pod gets a SPIFFE identity:*

```
spiffe://cluster.local/ns/<namespace>/sa/<serviceaccount>
```

*Citadel (inside istiod) issues short-lived X.509 certificates encoding this identity. Envoy uses these for mTLS. When Service A calls Service B, the connection has:*

- Server cert: spiffe://cluster.local/ns/production/sa/url-shortener-sa
- Client cert: spiffe://cluster.local/ns/production/sa/frontend-sa

B can verify: "this request came from the frontend SA in production namespace." No shared secrets, no API keys — just cryptographic identity.

```
# See the certificate Envoy has
istioctl proxy-config secret \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production

# Decode it
istioctl proxy-config secret \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production \
  -o json | jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' \
  | base64 -d | openssl x509 -noout -text | grep -E "Subject:|URI:"
# URI:spiffe://cluster.local/ns/production/sa/url-shortener-sa
```

**PeerAuthentication — mTLS policy**

```
# Enforce mTLS for entire mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system      # mesh-wide policy
spec:
  mtls:
    mode: STRICT

---
# Namespace-level override
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# Workload-level — allow specific port to skip mTLS (Prometheus scraping)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: url-shortener
  namespace: production
spec:
  selector:
    matchLabels:
      app: url-shortener
  mtls:
    mode: STRICT
  portLevelMtls:
    9090:                      # Prometheus scrapes this port without mTLS
      mode: PERMISSIVE
```

**AuthorizationPolicy — who can talk to whom**

```
# Default: deny all traffic in production
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}   # empty spec = deny everything

---
# Allow frontend to reach url-shortener on specific paths
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: url-shortener
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/production/sa/frontend-sa
    to:
    - operation:
        methods: [GET, POST]
        paths: ["/shorten", "/r/*", "/health", "/metrics"]
    when:
    - key: request.headers[x-api-version]
      values: ["v2"]            # only allow v2 API calls

---
# Allow Prometheus to scrape metrics from any pod
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: [monitoring]
        principals:
        - cluster.local/ns/monitoring/sa/prometheus-kube-prometheus-stack-prometheus
    to:
    - operation:
        paths: ["/metrics"]
        methods: [GET]

---
# JWT validation — require valid JWT for API access
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: url-shortener
  jwtRules:
  - issuer: "https://accounts.google.com"
    jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
    audiences: ["your-client-id"]
    forwardOriginalToken: true    # forward JWT to app for custom claims
```

## 📊 Part 5: Istio Observability

Istio's Envoy sidecars automatically emit metrics, traces, and access logs — without any app instrumentation.

**Golden signals from Istio (no code changes)**

```
# Install Istio addons (Kiali, Jaeger, Prometheus, Grafana)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/addons/kiali.yaml

# Access Kiali — service topology + health
istioctl dashboard kiali

# Access Jaeger — distributed traces
istioctl dashboard jaeger

# Access Grafana — Istio dashboards pre-built
istioctl dashboard grafana
```

**Key Istio metrics**

```
# Request rate per service
sum(rate(istio_requests_total[5m])) by (destination_service_name)

# Error rate (5xx) per service
sum(rate(istio_requests_total{response_code=~"5.*"}[5m])) by (destination_service_name)
/
sum(rate(istio_requests_total[5m])) by (destination_service_name)

# P99 latency per service
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket[5m]))
  by (destination_service_name, le)
)

# TCP bytes sent/received
sum(rate(istio_tcp_sent_bytes_total[5m])) by (destination_service_name)

# Circuit breaker ejections
sum(rate(envoy_cluster_outlier_detection_ejections_active[5m]))
by (cluster_name)
```

**Distributed tracing — automatic without SDK**

```
# Configure trace sampling rate
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-tracing
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 1.0    # sample 1% of requests in production
    # 100% in dev, 1% in prod (cost vs coverage tradeoff)
```

```
# Check tracing works — generate traffic then look in Jaeger
kubectl exec -n production deployment/frontend -- \
  curl -s http://url-shortener/shorten -d '{"url":"https://example.com"}'

istioctl dashboard jaeger
# Select service: url-shortener
# See the trace: frontend → url-shortener → redis
# Each span shows exact latency breakdown
```

## 🔀 Part 6: Advanced Traffic Patterns

**Circuit breaker — real scenario**

```
# DestinationRule with circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: redis-circuit-breaker
spec:
  host: redis-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100    # eject all if all are broken
```

```
# Test circuit breaker
# Generate errors
kubectl exec -n production deployment/url-shortener -- \
  sh -c 'for i in $(seq 1 20); do curl -s http://redis-service:6379/bad; done'

# Watch Envoy eject the endpoint
istioctl proxy-config clusters \
  $(kubectl get pod -n production -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}').production \
  | grep redis
# EJECTED hosts show up here
```

**Fault injection — chaos engineering in the mesh**

```
# Inject failures without changing any app code
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: redis-chaos
  namespace: production
spec:
  hosts:
  - redis-service
  http:
  - fault:
      delay:
        percentage:
          value: 30.0             # 30% of requests get a 2s delay
        fixedDelay: 2s
      abort:
        percentage:
          value: 10.0             # 10% get HTTP 503
        httpStatus: 503
    route:
    - destination:
        host: redis-service
```

```
# Apply chaos to redis
kubectl apply -f redis-chaos-vs.yaml

# Watch your app's error rate in Prometheus
# rate(istio_requests_total{destination_service="redis-service",response_code="503"}[1m])

# Remove chaos
kubectl delete virtualservice redis-chaos -n production
```

**Traffic mirroring — shadow production traffic**

```
# Send 100% to v1 AND copy to v2 (v2 response is discarded)
# Test v2 with real production traffic without any risk
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: url-shortener-mirror
spec:
  hosts:
  - url-shortener
  http:
  - route:
    - destination:
        host: url-shortener
        subset: v1
      weight: 100
    mirror:
      host: url-shortener
      subset: v2                  # v2 gets a copy of all requests
    mirrorPercentage:
      value: 100.0                # mirror 100% — or 10% for sampling
```

**Egress traffic control**

```
# Block all external traffic by default
# Only allowed ServiceEntries can go out

# First: set egress to REGISTRY_ONLY
# In istio ConfigMap:
# mesh:
#   outboundTrafficPolicy:
#     mode: REGISTRY_ONLY   ← or ALLOW_ANY (default, less secure)

# Then explicitly allow external services
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: allow-stripe
spec:
  hosts:
  - api.stripe.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
# Route external traffic through dedicated egress gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: stripe-via-egress
spec:
  hosts:
  - api.stripe.com
  gateways:
  - mesh
  - istio-egressgateway
  http:
  - match:
    - gateways:
      - mesh
      port: 443
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 443
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
    route:
    - destination:
        host: api.stripe.com
        port:
          number: 443
```

## 🌐 Part 7: Gateway API — Istio's Future

*Day 7's note: Ingress is deprecated, Gateway API is the future. Istio now supports Gateway API natively — the recommended approach for new deployments.*

```
# GatewayClass — define Istio as the implementation
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: istio
spec:
  controllerName: istio.io/gateway-controller

---
# Gateway — replaces Istio's own Gateway CRD
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: url-shortener-gateway
  namespace: production
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: url-shortener-tls
    allowedRoutes:
      namespaces:
        from: Same

---
# HTTPRoute — replaces VirtualService for north-south
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: url-shortener
  namespace: production
spec:
  parentRefs:
  - name: url-shortener-gateway
  hostnames:
  - short.yourdomain.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: url-shortener
      port: 80
      weight: 90
    - name: url-shortener-v2
      port: 80
      weight: 10            # 10% canary via Gateway API

  - matches:
    - path:
        type: PathPrefix
        value: /admin
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: x-internal
          value: "true"
    backendRefs:
    - name: admin-service
      port: 80

---
# TCPRoute — for non-HTTP services (replaces ServiceEntry for some cases)
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: postgres-route
  namespace: production
spec:
  parentRefs:
  - name: tcp-gateway
  rules:
  - backendRefs:
    - name: postgres
      port: 5432
```

**Gateway API vs Istio CRDs — comparison**

<img width="907" height="395" alt="image" src="https://github.com/user-attachments/assets/7463f144-ed11-4027-8da3-a130501efe22" />

**Production advice:** 

Use Gateway API for north-south (external traffic). Keep Istio VirtualService/DestinationRule for east-west (mesh traffic) — Gateway API's mesh support (GAMMA) is still experimental.

## 🖥️ Part 8: Hands-On Exercises

**Exercise 1: Install Istio and observe injection**

```
# Step 1 : Download and Extract Istio
# Download the latest Istio release
curl -L https://istio.io/downloadIstio | sh -

# Change into the extracted Istio directory (version name will match current release)
cd istio-*

# Step 2 : Add istioctl to Your System $PATH
# Move the istioctl binary to /usr/local/bin so it is available globally across all terminal sessions:

# Move binary to system PATH
sudo cp bin/istioctl /usr/local/bin/

# Verify istioctl is accessible
istioctl version --remote=false

# Step 3: Install Istio on Your Kubernetes Cluster
# Now that istioctl is available, run your installation command to deploy the demo profile:

istioctl install --set profile=demo -y

# What the demo profile installs:
# istiod: The core control plane (Pilot/Galley/Citadel).
# istio-ingressgateway: Ingress controller for external traffic.
# istio-egressgateway: Egress controller for outbound traffic.

# Step 4: Verify the Installation
# Check that all Istio control plane pods are running in the istio-system namespace:

kubectl get pods -n istio-system

# Expected Output:
NAME                                   READY   STATUS    RESTARTS   AGE
istiod-7d32e92c18-x9k2p               1/1     Running   0          45s
istio-ingressgateway-58273df82-12abc   1/1     Running   0          45s
istio-egressgateway-69d10e82f-xyz90    1/1     Running   0          45s

# Step 5: Enable Sidecar Auto-Injection (Optional)
# To enable automatic Envoy sidecar proxy injection for workloads in any given namespace (e.g., default or cks-16), add the istio-injection=enabled label:

kubectl label namespace default istio-injection=enabled

# Deploy test app — observe sidecar injection
kubectl create deployment httpbin --image=kennethreitz/httpbin
kubectl expose deployment httpbin --port=80

# Verify 2 containers per pod
kubectl get pods
# NAME                       READY   STATUS    RESTARTS
# httpbin-xxx                2/2     Running   0  ← app + istio-proxy

# Inspect the injected sidecar
kubectl describe pod -l app=httpbin \
  | grep -E "Name:|Image:|Port:" | head -20
```

**Exercise 2: Canary with traffic splitting**

```
# Deploy v1 and v2 of your app
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-shortener-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: url-shortener
      version: v1
  template:
    metadata:
      labels:
        app: url-shortener
        version: v1
    spec:
      containers:
      - name: app
        image: nginx:1.24
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-shortener-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: url-shortener
      version: v2
  template:
    metadata:
      labels:
        app: url-shortener
        version: v2
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: url-shortener
spec:
  selector:
    app: url-shortener      # selects BOTH versions
  ports:
  - port: 80
EOF

# Without Istio: ~25% goes to v2 (1 of 4 pods)
# With Istio: we control exactly

cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: url-shortener
spec:
  host: url-shortener
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: url-shortener
spec:
  hosts:
  - url-shortener
  http:
  - route:
    - destination:
        host: url-shortener
        subset: v1
      weight: 90
    - destination:
        host: url-shortener
        subset: v2
      weight: 10
EOF

# Generate traffic and observe distribution
kubectl run traffic-gen --image=busybox --rm -it -- sh
  for i in $(seq 1 20); do
    wget -qO- http://url-shortener | grep -o "nginx/[0-9.]*"
  done
# Should see ~18 times nginx/1.24 and ~2 times nginx/1.25

# Gradually shift — update weights
kubectl patch virtualservice url-shortener \
  --type=merge \
  -p '{"spec":{"http":[{"route":[{"destination":{"host":"url-shortener","subset":"v1"},"weight":50},{"destination":{"host":"url-shortener","subset":"v2"},"weight":50}]}]}}'
```

**Exercise 3: mTLS verification**

```
# Enforce strict mTLS
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF

# Verify mTLS is working
kubectl exec deployment/httpbin -c istio-proxy -- \
  curl -s http://url-shortener/
# Works — goes through mTLS

# Try to bypass Envoy (direct connection without mTLS) — should fail
kubectl run no-sidecar \
  --image=busybox \
  --annotations="sidecar.istio.io/inject=false" \
  --rm -it \
  -- wget -qO- http://url-shortener/
# Connection refused or reset — STRICT mode blocks non-mTLS

# Verify certificate in the sidecar
istioctl proxy-config secret \
  $(kubectl get pod -l app=httpbin -o jsonpath='{.items[0].metadata.name}')
```

**Exercise 4: Fault injection and circuit breaker**

```
# Inject 3 second delay to 100% of requests to httpbin
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin-fault
spec:
  hosts:
  - httpbin
  http:
  - fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 3s
    route:
    - destination:
        host: httpbin
        port:
          number: 80
EOF

# Test — should take 3 seconds
time kubectl exec deployment/url-shortener-v1 -- \
  wget -qO- http://httpbin/delay/0

# Remove fault
kubectl delete virtualservice httpbin-fault

# Add circuit breaker
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: httpbin-cb
spec:
  host: httpbin
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 2
      interval: 10s
      baseEjectionTime: 30s
EOF

# Trigger circuit breaker
kubectl exec deployment/url-shortener-v1 -- sh -c \
  'for i in $(seq 1 10); do wget -qO- http://httpbin/status/500; done'

# Check if endpoint is ejected
istioctl proxy-config endpoints \
  $(kubectl get pod -l app=url-shortener,version=v1 \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep httpbin
# HOST:PORT      STATUS    OUTLIER CHECK
# 10.x.x.x:80   HEALTHY   OK
# (or EJECTED if circuit opened)
```

## 🎯 Part 9: Interview Questions — Day 20

**Q1: Explain how Istio intercepts traffic without modifying application code.**

Istio uses a MutatingAdmissionWebhook to inject an istio-proxy (Envoy) sidecar into every pod in labeled namespaces. An init container istio-init runs first and uses iptables rules to redirect ALL inbound and outbound traffic to Envoy's ports (15006 inbound, 15001 outbound). The app process sees no difference — it still thinks it's connecting directly to other services. Envoy intercepts, applies policies, forwards the traffic, and the response comes back the same way. The only exception is traffic originating from Envoy itself — iptables rules exclude UID 1337 (Envoy's UID) to prevent infinite loops. This transparent interception is why service meshes require no code changes.

**Q2: What is the difference between PeerAuthentication and AuthorizationPolicy in Istio?**

PeerAuthentication controls the transport security — it defines whether mTLS is required, optional, or disabled for a workload or namespace. It answers: "does this connection need to be encrypted and mutually authenticated?" AuthorizationPolicy controls access — it defines which authenticated identities (SPIFFE principals) can perform which actions on which paths. It answers: "even if this connection is authenticated, is this caller allowed to do this?" Both work together: PeerAuthentication ensures every connection is authenticated, AuthorizationPolicy then checks whether the authenticated caller is authorized. You need both for a zero-trust service mesh.

**Q3: A service in the mesh is getting 503s occasionally. How do you diagnose it?**

Start with istioctl proxy-config endpoints <pod>.namespace — check if any endpoints are EJECTED (circuit breaker tripped). Check kubectl describe virtualservice for the service — are there retry policies? Check Envoy's access logs: kubectl logs <pod> -c istio-proxy | grep 503. Check Prometheus: rate(istio_requests_total{response_code="503"}[5m]) by destination and source. Use Jaeger to find a slow/failed trace — it shows exactly which hop returned 503 and the response flags (UF=upstream connection failure, UO=upstream overflow/circuit open, NR=no route). Check the DestinationRule outlierDetection settings — the thresholds might be too aggressive for a noisy service.

**Q4: How does Istio's circuit breaker differ from application-level circuit breaking (like Hystrix)?**

Application-level circuit breakers (Hystrix, Resilience4j) live in the app process, require SDK integration, and only protect the specific calls they wrap. If you have 10 services in 3 languages, you need 3 different library integrations with potentially inconsistent behavior. Istio's circuit breaker is in the Envoy sidecar — uniform behavior across all services regardless of language, zero code changes, consistent configuration via DestinationRule. The tradeoff: Envoy's circuit breaker is connection/request count based (outlierDetection), while app-level breakers can implement more sophisticated logic like sliding window failure rates, half-open states, and fallback methods that return cached data. Production: use Istio for infrastructure-level protection (host ejection), use application-level for business logic fallbacks.

**Q5: What is the GAMMA initiative and why does it matter for Istio?**

GAMMA (Gateway API for Mesh Management and Administration) is a SIG-Network initiative to extend the Gateway API to handle east-west (mesh) traffic — not just north-south (ingress). Currently, Gateway API only standardizes ingress. East-west mesh traffic is handled by Istio-specific CRDs (VirtualService, DestinationRule) or Linkerd-specific CRDs. GAMMA aims to standardize these so a service mesh config written for Istio could work on Linkerd or Cilium with minimal changes. It matters because: organizations can avoid vendor lock-in on mesh choice, platform engineers learn one API instead of three, and the community consolidates around standard primitives. Currently experimental — use Istio CRDs for east-west, Gateway API for north-south.

**Q6: You need to add authentication to 20 microservices. How does Istio solve this at scale?**

Apply a single RequestAuthentication policy at the namespace or mesh level that validates JWT tokens from your IdP. Then apply AuthorizationPolicy to require a valid JWT for all requests — any request without a valid token is rejected by Envoy before it reaches the app. This is O(1) configuration change affecting all 20 services simultaneously — no code changes, no SDK upgrades. The app receives the validated JWT (if forwardOriginalToken: true) and can trust the claims. Contrast with the alternative: integrating a JWT validation library in each of the 20 services in potentially different languages, with different error handling, different claim parsing — weeks of work with inconsistent results. Production nuance: set up a RequestAuthentication without an AuthorizationPolicy first — this validates tokens when present but doesn't require them. Add the AuthorizationPolicy in a second step after verifying token flow works.





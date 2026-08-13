# Day 22 — Kubernetes Internals: The 1% Knowledge

This is the deepest day of the curriculum. The topics here aren't in most tutorials, aren't covered in certification prep, and are exactly what distinguishes a senior K8s engineer from a principal or staff engineer. Every company paying $180k+ for K8s expertise tests this level.

## 🧠 Part 1: The API Server Request Lifecycle — Every Step

*When you run kubectl apply -f deployment.yaml, here is everything that happens inside the API server. Most engineers know 3-4 steps. Know all 12.*

```
Step 1:  TLS Handshake
Step 2:  Authentication
Step 3:  Audit (RequestReceived)
Step 4:  Impersonation check
Step 5:  Authorization (RBAC)
Step 6:  Admission (Mutating webhooks)
Step 7:  Object schema validation
Step 8:  Admission (Validating webhooks)
Step 9:  Audit (ResponseStarted)
Step 10: Persistence to etcd
Step 11: Watch event broadcast
Step 12: Response to client
```

**Step 1-2: TLS + Authentication**

```
# See what auth mechanisms are enabled
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml \
  | grep -E "client-ca|token-auth|oidc"

# Authentication methods tried in order:
# 1. X.509 client certificates (kubeconfig cert)
# 2. Bearer tokens (static, bootstrap, service account)
# 3. OIDC tokens
# 4. Webhook token authentication
# 5. Anonymous (if --anonymous-auth=true, which it is by default)

# What happens on auth failure?
# API server returns 401 Unauthorized
# Audit log records the failure
# No RBAC check — auth must pass first

# Inspect the identity extracted from your cert
kubectl config view --raw \
  -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d \
  | openssl x509 -noout -subject
# subject=O=system:masters, CN=kubernetes-admin
# O=group, CN=username
```

**Step 3: Audit logging — two events per request**

```
# Every request generates TWO audit events:
# 1. RequestReceived — when request arrives (before any processing)
# 2. ResponseComplete — when response is sent (after processing)
# (ResponseStarted for watch/stream responses)

# This means even rejected requests appear in audit logs
# Critical for security: you see FAILED authentication attempts

# Read a raw audit event
docker exec k8s-mastery-control-plane \
  tail -5 /var/log/kubernetes/audit.log 2>/dev/null \
  | python3 -m json.tool | head -40

# Key audit fields:
# auditID      — correlates RequestReceived + ResponseComplete
# user         — who made the request (after auth)
# verb         — get, list, create, update, patch, delete, watch
# objectRef    — what resource was targeted
# responseStatus.code — HTTP status code
# requestReceivedTimestamp
# stageTimestamp
```

**Step 4: Impersonation**

```
# Impersonation lets you act as another user
# Critical for: debugging RBAC, admin doing operations as service

kubectl get pods \
  --as=system:serviceaccount:production:app-sa \
  --as-group=system:authenticated
# API server extracts your real identity, checks if you have
# impersonate verb on users/groups/serviceaccounts resources
# Then proceeds as the impersonated identity for RBAC

# Check who can impersonate
kubectl get clusterrolebindings -o json \
  | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# In audit log: impersonation shows both identities
# user.username: kubernetes-admin (real user)
# user.extra.originalUser: kubernetes-admin
# impersonatedUser.username: system:serviceaccount:production:app-sa
```

**Step 5: Authorization — RBAC internals**

```
# RBAC is not the only authorizer — multiple can chain:
# --authorization-mode=Node,RBAC

# Node authorizer: special-purpose for kubelet
# Kubelet can only read pods/secrets/configmaps for pods on ITS node
# Prevents node compromise from spreading to all nodes

# RBAC evaluation:
# 1. Collect all roles bound to the user/group/SA
# 2. For each rule: check if verb+resource+apiGroup matches request
# 3. If ANY rule matches: ALLOW
# 4. If NO rule matches: DENY
# RBAC is ALLOW-only — no explicit deny rules

# Verbose RBAC debugging
kubectl get pods -v=8 2>&1 | grep -E "Response Status:|RBAC"

# What the API server does internally:
# SubjectAccessReview — you can call this yourself
kubectl create -f - <<EOF
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: chetan
  groups: ["dev-team"]
  resourceAttributes:
    namespace: production
    verb: create
    resource: pods
    group: ""
EOF
# Status: allowed: true/false
```

**Step 10: etcd persistence — optimistic concurrency**

```
# etcd stores objects with a resourceVersion (revision number)
# Every write increments the revision
# Concurrent writes use optimistic locking:

# Controller reads object at revision 1234
# Controller makes changes
# Controller writes: "update this object IF it's still at revision 1234"
# If another write happened in between → 409 Conflict
# Controller must re-read and retry

# This is why you see RetryOnConflict in controller code
# And why controllers must be idempotent

# See the resourceVersion on any object
kubectl get deployment url-shortener -o jsonpath='{.metadata.resourceVersion}'
# 89234

# If you try to update with wrong resourceVersion:
kubectl patch deployment url-shortener \
  --type=merge \
  -p '{"metadata":{"resourceVersion":"1"},"spec":{"replicas":5}}'
# Error: Operation cannot be fulfilled: the object has been modified
# Please apply your changes to the latest version and retry
```

**Step 11: Watch event broadcast — how controllers get instant notifications**

```
// Inside the API server — simplified watch mechanism

// 1. etcd watch — persistent watch on all keys
etcdWatcher := etcdClient.Watch(ctx, "/registry/", clientv3.WithPrefix())

// 2. Caching layer — WatchCache in API server
// Stores a ring buffer of recent events (default 100 per resource type)
// Allows clients to get events without hitting etcd directly

// 3. When an event arrives from etcd:
for event := range etcdWatcher {
    // Decode the etcd value into a K8s object
    obj := decode(event.Kv.Value)
    
    // Update the watch cache
    watchCache.Add(obj)
    
    // Fan out to all active watchers
    for _, watcher := range activeWatchers {
        if watcher.matches(obj) {
            watcher.channel <- WatchEvent{
                Type:   event.Type,   // ADDED, MODIFIED, DELETED
                Object: obj,
            }
        }
    }
}

// 4. Controller receives event → enqueues to work queue
// Work queue rate-limits and deduplicates
// Controller reconciles
```

```
# See watch in action — very verbose but educational
kubectl get pods --watch -v=9 2>&1 | grep -E "Watch|resourceVersion|MODIFIED"

# What you'll see:
# Establishing watch on /api/v1/namespaces/default/pods?watch=true&resourceVersion=89234
# <stream of JSON events>
# {"type":"MODIFIED","object":{"kind":"Pod",...}}
```

## ⚙️ Part 2: Scheduler Framework — Plugins Deep Dive

*The scheduler is a plugin framework. Every decision is made by composable plugins.*

```
// Scheduler extension points — plugins hook into these
type Framework interface {
    // Filter phase — eliminate nodes
    RunFilterPlugins(ctx, state, pod, nodeInfo)
    
    // Score phase — rank remaining nodes
    RunScorePlugins(ctx, state, pod, nodes)
    
    // PostFilter — what to do when no node passes filter
    // (preemption runs here)
    RunPostFilterPlugins(ctx, state, pod, filteredNodeStatusMap)
    
    // Reserve — claim resources before binding
    RunReservePluginsReserve(ctx, state, pod, nodeName)
    
    // Permit — last chance to delay or reject binding
    RunPermitPlugins(ctx, state, pod, nodeName)
    
    // Bind — actually write node assignment to API server
    RunBindPlugins(ctx, state, pod, nodeName)
}
```

**Scheduler performance at scale**

```
# How the scheduler handles 5000 nodes:
# It doesn't score ALL nodes — it uses percentage-based filtering

# --percentage-of-nodes-to-score (default: 0 = auto)
# Cluster < 100 nodes: score all nodes
# Cluster 100-5000 nodes: score 50% of nodes (min 100)
# Cluster > 5000 nodes: score 10% of nodes (min 100)

# This means scheduling is NOT globally optimal at scale
# It's "good enough" optimized for throughput

# See scheduler config
kubectl get configmap -n kube-system \
  $(kubectl get configmap -n kube-system | grep scheduler | awk '{print $1}') \
  -o yaml 2>/dev/null || \
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-scheduler.yaml \
  | grep -E "config|profile"
```

**Write a scheduler plugin**

```
// Custom scheduler plugin — scores nodes by a custom label
package customplugin

import (
    "context"
    "fmt"
    v1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/kubernetes/pkg/scheduler/framework"
)

const Name = "CustomTierPlugin"

type CustomTierPlugin struct {
    handle framework.Handle
}

// Score — called for each node that passes filtering
func (p *CustomTierPlugin) Score(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeName string,
) (int64, *framework.Status) {

    nodeInfo, err := p.handle.SnapshotSharedLister().
        NodeInfos().Get(nodeName)
    if err != nil {
        return 0, framework.NewStatus(
            framework.Error,
            fmt.Sprintf("getting node %s info: %v", nodeName, err),
        )
    }

    node := nodeInfo.Node()
    tier, exists := node.Labels["tier"]

    if !exists {
        return 0, nil    // no tier label = score 0
    }

    // Score based on tier label
    switch tier {
    case "premium":
        return 100, nil  // highest score = preferred
    case "standard":
        return 50, nil
    case "spot":
        return 10, nil   // lowest score = last resort
    default:
        return 0, nil
    }
}

func (p *CustomTierPlugin) ScoreExtensions() framework.ScoreExtensions {
    return nil
}

func (p *CustomTierPlugin) Name() string { return Name }

// Register with scheduler
func New(obj runtime.Object, h framework.Handle) (framework.Plugin, error) {
    return &CustomTierPlugin{handle: h}, nil
}
```

## 🔬 Part 3: etcd Internals — What Every K8s Engineer Must Know

**How K8s uses etcd — the actual data model**

```
# Every K8s resource stored at /registry/<group>/<resource>/<namespace>/<name>
# Let's see what's actually in there

docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry --prefix --keys-only 2>/dev/null | head -50
"

# Output shows the full key structure:
# /registry/apiregistration.k8s.io/apiservices/v1.
# /registry/clusterrolebindings/cluster-admin
# /registry/deployments/production/url-shortener
# /registry/namespaces/default
# /registry/pods/production/url-shortener-xxx
# /registry/secrets/production/app-secrets
```

**etcd compaction and defragmentation**

```
# etcd keeps ALL historical revisions by default
# This causes disk to grow unboundedly
# Compaction removes old revisions — keep only latest

docker exec k8s-mastery-control-plane bash -c "
# Get current revision
REV=\$(ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json 2>/dev/null | python3 -c \"
import json,sys
data=json.load(sys.stdin)
print(data[0]['Status']['header']['revision'])
\")
echo \"Current revision: \$REV\"

# Compact to current revision (removes all historical data)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact \$REV 2>/dev/null

# Defragment — reclaims disk space after compaction
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag 2>/dev/null
echo 'Compaction and defrag complete'
"
```

**etcd performance tuning**

```
# Critical etcd flags for production
# In /etc/kubernetes/manifests/etcd.yaml

- --heartbeat-interval=100        # ms between leader heartbeats (default 100)
- --election-timeout=1000         # ms before follower starts election (default 1000)
                                   # Rule: election-timeout = 10x heartbeat-interval
- --quota-backend-bytes=8589934592  # 8GB max etcd db size (default 2GB)
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h  # auto-compact hourly
- --snapshot-count=10000          # snapshot to disk every 10000 writes

# Disk requirements:
# etcd REQUIRES low latency disk — 10ms write latency budget
# Sequential write: >50 MB/s
# fsync latency: <10ms
# Use dedicated NVMe — never share with other workloads
# Check current latency:
```

```
# Check etcd disk latency
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  check perf 2>/dev/null || echo 'check perf not available in this version'
"

# Check etcd metrics
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table 2>/dev/null
"
```

## 🔄 Part 4: Controller Manager Internals

*The controller-manager runs 30+ controllers in a single binary. Understanding which controller owns what is essential for debugging.*

```
# See all controllers the controller-manager runs
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-controller-manager \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep "Starting controller" | head -30

# Key controllers and what they manage:
```

<img width="810" height="588" alt="image" src="https://github.com/user-attachments/assets/197f352c-ba1a-4f40-bcbc-141c8f9f63c1" />

```
# Leader election — controller-manager uses Lease objects
kubectl get lease -n kube-system
# NAME                          HOLDER                                         AGE
# kube-controller-manager       k8s-mastery-control-plane_xxx                  5d
# kube-scheduler                k8s-mastery-control-plane_xxx                  5d

# See who holds the lease
kubectl get lease kube-controller-manager \
  -n kube-system \
  -o jsonpath='{.spec.holderIdentity}'

# If leader dies, another replica acquires lease within leaseDuration
# Default: leaseDuration=15s, renewDeadline=10s, retryPeriod=2s
```

## 🕸️ Part 5: Kubernetes Networking Internals — Packet Level

**iptables chains for Services — the full path**

```
# A ClusterIP Service creates these iptables rules on every node

# See the KUBE-SERVICES chain (entry point)
docker exec k8s-mastery-control-plane \
  iptables -t nat -L KUBE-SERVICES -n --line-numbers 2>/dev/null | head -20

# For a Service named "url-shortener" in "production":
# KUBE-SERVICES → KUBE-SVC-<hash> (matches ClusterIP:Port)
# KUBE-SVC-<hash> → KUBE-SEP-<hash1> (endpoint 1, probability 0.33)
#                → KUBE-SEP-<hash2> (endpoint 2, probability 0.50 of remainder)
#                → KUBE-SEP-<hash3> (endpoint 3, probability 1.00 of remainder)
# KUBE-SEP-<hash> → DNAT to pod IP:Port

# The probability math is stateless random selection:
# pod1: 1/3 = 0.333 probability
# pod2: 1/2 of remaining = 0.333 probability
# pod3: 1/1 of remaining = 0.333 probability

# Total packets per pod: equal — round-robin via probability

docker exec k8s-mastery-control-plane bash -c "
# Find a Service's KUBE-SVC chain
SVC_CHAIN=\$(iptables -t nat -L KUBE-SERVICES -n 2>/dev/null \
  | grep 'url-shortener\|kubernetes' | head -1 | awk '{print \$2}')
echo \"Chain: \$SVC_CHAIN\"
[ -n \"\$SVC_CHAIN\" ] && iptables -t nat -L \$SVC_CHAIN -n 2>/dev/null
"
```

**IPVS mode — why it's faster**

```
# Check if kube-proxy is in iptables or IPVS mode
kubectl get configmap -n kube-system kube-proxy -o yaml \
  | grep mode

# IPVS uses kernel hash tables — O(1) lookup vs iptables O(n)
# At 10,000 services: iptables has 40,000+ rules to traverse per packet
# IPVS: always one hash lookup regardless of service count

# With IPVS mode — see the virtual servers
docker exec k8s-mastery-worker \
  ipvsadm -Ln 2>/dev/null | head -30 || \
echo "IPVS not enabled on this cluster (using iptables mode)"

# Enable IPVS (requires kube-proxy restart):
kubectl edit configmap kube-proxy -n kube-system
# Change: mode: "" → mode: "ipvs"
kubectl rollout restart daemonset kube-proxy -n kube-system
```

**eBPF — the future beyond iptables**

```
# Cilium replaces kube-proxy entirely with eBPF
# eBPF programs run in kernel — no userspace overhead
# Service routing: O(1) hash map lookup in kernel
# NetworkPolicy: enforced in kernel before packet leaves NIC

# Check if Cilium is installed
kubectl get pods -n kube-system -l k8s-app=cilium 2>/dev/null \
  || echo "Cilium not installed"

# With Cilium — no iptables rules for services
docker exec k8s-mastery-worker \
  iptables -t nat -L KUBE-SERVICES -n 2>/dev/null \
  | wc -l
# Much fewer rules — Cilium handles service routing in eBPF

# Cilium observability — what eBPF enables
# cilium monitor --type drop   # see dropped packets in real time
# cilium endpoint list          # all managed endpoints
# cilium policy get             # all NetworkPolicies in Cilium format
```

## 🔐 Part 6: Security Internals — Token Projection and SPIFFE

**How projected tokens work — cryptographically**

```
# Get the projected token from a pod
kubectl exec -n default \
  $(kubectl get pod -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}') \
  -- cat /var/run/secrets/kubernetes.io/serviceaccount/token \
  2>/dev/null | cut -d. -f2 | base64 -d 2>/dev/null | jq .

# Token claims:
# {
#   "iss": "https://kubernetes.default.svc.cluster.local",
#   "aud": ["https://kubernetes.default.svc.cluster.local"],
#   "exp": 1700000000,           ← expires in 1 hour
#   "iat": 1699996400,           ← issued now
#   "nbf": 1699996400,
#   "sub": "system:serviceaccount:default:my-sa",
#   "kubernetes.io": {
#     "namespace": "default",
#     "pod": {"name":"my-pod-xxx","uid":"abc-123"},
#     "serviceaccount": {"name":"my-sa","uid":"def-456"}
#   }
# }

# Kubelet rotates this token automatically before exp
# New token written to the projected volume
# App reading file on each request gets fresh token automatically
```

**TokenReview — how the API server validates tokens**

```
# When a component sends a token to the API server:
# API server calls TokenReview internally

kubectl create -f - <<EOF
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: $(kubectl exec \
    $(kubectl get pod -l app=url-shortener \
      -o jsonpath='{.items[0].metadata.name}') \
    -- cat /var/run/secrets/kubernetes.io/serviceaccount/token 2>/dev/null)
  audiences: ["https://kubernetes.default.svc.cluster.local"]
EOF

# Response shows:
# status:
#   authenticated: true
#   user:
#     username: system:serviceaccount:default:url-shortener-sa
#     groups: [system:serviceaccounts, system:authenticated]
#   audiences: [https://kubernetes.default.svc.cluster.local]
```

## 📊 Part 7: Kubernetes at Scale — Production Limits and Tuning

**API server tuning**

```
# For large clusters — tune these flags in kube-apiserver.yaml
- --max-requests-inflight=800          # default 400 — concurrent non-mutating requests
- --max-mutating-requests-inflight=400 # default 200 — concurrent mutating requests
- --watch-cache-sizes=pods#1000        # larger watch cache for frequently watched resources
- --default-watch-cache-size=100       # default per resource
- --request-timeout=60s               # default 60s — max time for non-watch request
- --enable-priority-and-fairness=true  # APF — QoS for API requests

# APF (API Priority and Fairness) — prevents any one client from starving others
# FlowSchema — classifies requests into priority levels
# PriorityLevelConfiguration — limits concurrency per level
```

```
# See APF configuration
kubectl get flowschemas
kubectl get prioritylevelconfigurations

# FlowSchemas classify requests:
kubectl describe flowschema catch-all
kubectl describe flowschema exempt           # system:masters bypass APF
kubectl describe flowschema kube-system-service-accounts

# See API server request metrics
kubectl get --raw /metrics | grep apiserver_request_total | head -10
kubectl get --raw /metrics | grep apiserver_current_inflight_requests
```

**Scalability benchmarks — know these numbers cold**

```
Max nodes per cluster:              5,000
Max pods per cluster:               150,000
Max pods per node:                  110 (default)
Max Services:                       10,000
Max namespaces:                     10,000
API server P99 response time:       1 second (SLO)
Pod startup time (no image pull):   5 seconds (SLO)
etcd storage recommendation:        8GB max
etcd max size before degradation:   ~2GB without compaction
Watch cache events per resource:    100 (default ring buffer)
Leader election duration:           15 seconds
```

## 🔭 Part 8: Kubernetes API — Raw Access

**Understanding the raw API makes you dangerous for debugging and automation.**

```
# The K8s API is just REST over HTTPS
# kubectl is a client that calls this API

# Set up proxy for easy raw access
kubectl proxy --port=8001 &

# List all pods in default namespace
curl -s http://localhost:8001/api/v1/namespaces/default/pods \
  | jq '.items[].metadata.name'

# Watch pods (long-lived connection)
curl -s "http://localhost:8001/api/v1/namespaces/default/pods?watch=true" \
  | while read line; do
      echo "$line" | jq -r '"\(.type) \(.object.metadata.name)"'
    done

# Get a specific pod
curl -s http://localhost:8001/api/v1/namespaces/default/pods/my-pod \
  | jq .metadata.resourceVersion

# Patch a deployment
curl -s \
  -X PATCH \
  -H "Content-Type: application/merge-patch+json" \
  http://localhost:8001/apis/apps/v1/namespaces/default/deployments/url-shortener \
  -d '{"spec":{"replicas":5}}'

# List all API groups
curl -s http://localhost:8001/apis | jq '.groups[].name'

# List all resources in the apps group
curl -s http://localhost:8001/apis/apps/v1 \
  | jq '.resources[].name'

# Server-side apply via raw API
curl -s \
  -X PATCH \
  -H "Content-Type: application/apply-patch+yaml" \
  "http://localhost:8001/apis/apps/v1/namespaces/default/deployments/url-shortener?fieldManager=my-tool&force=true" \
  -d @deployment.yaml
```

## 🖥️ Part 9: Hands-On — Deep Diagnostic Exercises

**Exercise 1: Trace a request through the API server**

```
# Enable verbose logging to see every step
kubectl get pods -v=10 2>&1 | head -80

# What you see:
# I0101 ... GET https://localhost:6443/api/v1/namespaces/default/pods
# I0101 ... Response Status: 200 OK in 8ms
# I0101 ... Response Headers: ...
# I0101 ... Response Body: {"kind":"PodList",...}

# Enable audit logging and trace a specific request
kubectl create secret generic trace-test \
  --from-literal=key=value \
  -n default

# Find it in audit log
docker exec k8s-mastery-control-plane \
  grep 'trace-test' /var/log/kubernetes/audit.log 2>/dev/null \
  | python3 -m json.tool \
  | grep -E '"verb"|"user"|"requestObject"|"responseStatus"' \
  | head -20
```

**Exercise 2: Read etcd directly and understand what's stored**

```
# Read every pod in default namespace directly from etcd
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default --prefix --keys-only 2>/dev/null
"

# Read a specific pod (raw binary — use strings to extract readable parts)
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get \$(ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/pods/default --prefix --keys-only 2>/dev/null | head -1) \
  --print-value-only 2>/dev/null | strings | head -30
"
```

**Exercise 3: Scheduler internals — force scheduling failure and read logs**

```
# Create an impossible pod — scheduler must log why
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: scheduler-debug
spec:
  schedulerName: default-scheduler
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: impossible-label
            operator: Exists
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "999"    # impossible amount
        memory: "999Gi"
EOF

# Read what scheduler says
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-scheduler \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep scheduler-debug | tail -10

# Read what API server says about this pod's condition
kubectl get pod scheduler-debug -o yaml \
  | grep -A 10 conditions

kubectl describe pod scheduler-debug | grep -A 5 Events

# Cleanup
kubectl delete pod scheduler-debug
```

**Exercise 4: Watch cache behavior**

```
# Start a watch from current resourceVersion
RV=$(kubectl get pods -o jsonpath='{.metadata.resourceVersion}')
echo "Starting watch from resourceVersion: $RV"

# Start a watch
kubectl get pods --watch \
  -o jsonpath='{.type} {.object.metadata.name}' \
  2>/dev/null &
WATCH_PID=$!

# Create and delete a pod — watch should receive events instantly
kubectl run watch-test --image=nginx:1.25 -- sleep 60
sleep 5
kubectl delete pod watch-test --force --grace-period=0

# Kill watch
sleep 2
kill $WATCH_PID 2>/dev/null

# Try with expired resourceVersion — K8s returns 410 Gone
kubectl get pods \
  --resource-version=1 \
  --watch 2>&1 | head -5
# Error: 410 Gone — cache expired, must relist
```

## 🎯 Part 10: Interview Questions — Day 25

**Q1: Walk me through every step that happens when you kubectl apply a Deployment, from keystroke to running pod.**

kubectl reads your kubeconfig, finds the cluster endpoint and client cert. It serializes the YAML to JSON and sends a PATCH (server-side apply) or PUT/POST request over TLS to the API server. API server authenticates the client cert — extracts username and groups. Audit event recorded (RequestReceived). RBAC checked — does this user have create/patch on deployments in this namespace? Mutating webhooks run — Istio sidecar injector, label injectors, resource defaulters. Schema validation — is the spec valid? Validating webhooks run — Gatekeeper, Kyverno. Persisted to etcd. Watch event broadcast. Deployment controller sees the new object — creates a ReplicaSet. ReplicaSet controller sees the ReplicaSet — creates 3 Pods (Pending, no node assigned). Scheduler sees Pending pods — filters nodes, scores nodes, assigns each pod to a node — writes to API server. kubelet on each assigned node sees its pod via watch — calls containerd to pull image and start container. kubelet updates pod status to Running. readinessProbe passes — Endpoints controller adds pod IPs to Service endpoints. Traffic can now reach the pods.

**Q2: Why does Kubernetes use optimistic concurrency instead of pessimistic locking for etcd writes?**

Pessimistic locking (lock before read, hold until write) in a distributed system causes: lock holder death → cluster hangs, network partition → deadlock, high contention → serialized throughput. Optimistic concurrency trades blocking for retries — read the object, remember its resourceVersion, compute changes, write IF resourceVersion still matches. If another write happened concurrently: detect the conflict via 409 response, re-read, re-apply changes. This is safe because K8s controllers are designed to be idempotent — running the reconcile loop multiple times produces the same result. The tradeoff: under high contention, controllers retry frequently. In practice, conflicts are rare because each controller owns different resources. RetryOnConflict in client-go implements exponential backoff for these retries.

**Q3: What is API Priority and Fairness (APF) and why was it needed?**

Before APF, the API server had two simple queues: one for mutating requests (200 slots) and one for read requests (400 slots). A single misbehaving controller could fill all 400 read slots, starving kubectl, the scheduler, and all other clients. APF replaces this with a work queue system where requests are classified by FlowSchema (who is making the request, what type) into PriorityLevels, each with its own concurrency limit and queue. exempt level: system:masters bypasses all limits — for break-glass access. cluster-admin level: high priority for admin operations. catch-all level: everyone else, strict limits. Within each level, fair queuing via shuffle sharding — each flow (user + resource + verb combination) gets its own virtual queue, so one bad actor can't monopolize a priority level.

**Q4: How does the watch cache prevent the API server from being overwhelmed by controllers?**

Without caching: each of 100 controllers doing List+Watch on pods would all hit etcd directly — 100 concurrent etcd streams, fan-out of all events to 100 clients via etcd. etcd is not designed for high fan-out. With the watch cache: the API server maintains a single watch against etcd per resource type. When an event arrives, it goes into a ring buffer (WatchCache). All controller watches connect to the API server's WatchCache, not etcd directly. The API server fans out one etcd event to 100 clients from its own process. kubectl get pods serves from the WatchCache — zero etcd reads. Only cache misses (resourceVersion too old, requested specific version not in cache) go to etcd. This reduces etcd read pressure by 10-100x in busy clusters.

**Q5: A controller keeps receiving the same reconcile event repeatedly even though nothing has changed. What's likely happening and how do you fix it?**

This is called a reconcile loop — the controller reconciles, makes a change, that change triggers another watch event, which triggers another reconcile, forever. Common causes: the controller updates the object's status or metadata on every reconcile, triggering a MODIFIED event even when nothing meaningful changed. Or: the controller uses Update instead of Patch, overwriting the full object including fields managed by other controllers, causing conflicts and re-reconciliations. Fix: before updating, compare desired vs actual state and only write if there's a real diff. Use Patch with SSA instead of Update. For status updates, use equality.Semantic.DeepEqual to skip the update if nothing changed. Add a Generation check — only reconcile when metadata.generation changes (spec changes), not on every metadata.resourceVersion change (status changes don't increment generation).

**Q6: What happens to running pods during an API server downtime of 5 minutes?**

Running pods keep running — kubelet does not need the API server to maintain running containers. containerd runs independently of K8s. Kubelet caches its pod list locally — if it can't reach the API server, it keeps running the last known set of pods. What stops working: new pod scheduling (scheduler can't write assignments), scaling (HPA and manual scale), config updates (new ConfigMaps/Secrets don't propagate), pod restarts that would require rescheduling (kubelet restarts containers locally but can't get new assignments if the pod dies), kubectl commands, all control plane operations. What keeps working: all currently running containers, Services (kube-proxy uses its own cache and existing iptables rules), CoreDNS (it's already running), load balancer rules. This is why stateless apps tolerate brief API server outages gracefully, but stateful operations requiring K8s API (like StatefulSet ordered startup) are blocked.











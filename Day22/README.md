# 🧠 Kubernetes Internals — A Beginner-Friendly Deep Dive

> **Who this is for:** You already know the basics of Kubernetes (you can deploy a Pod, expose a Service, and read a YAML manifest), but you want to understand *what actually happens under the hood* when you run `kubectl apply`. This guide takes the deepest, most advanced internals of Kubernetes and explains them in plain English, with analogies, diagrams, and hands-on commands you can try on your own cluster.

> **How to read this:** Go top to bottom. Each part builds on the previous one. Every section starts with a simple analogy, then walks through the concept step by step, and ends with real commands you can run to see it yourself.

---

## 📑 Table of Contents

1. [The Big Picture: How Kubernetes Components Talk](#-part-0-the-big-picture)
2. [The API Server Request Lifecycle — Every Step Explained](#-part-1-the-api-server-request-lifecycle)
3. [The Scheduler: How Kubernetes Decides Where Pods Run](#-part-2-the-scheduler)
4. [etcd Internals: Where Kubernetes Stores Everything](#-part-3-etcd-internals)
5. [The Controller Manager: Kubernetes' Automation Engine](#-part-4-the-controller-manager)
6. [Networking Internals: How Packets Reach Your Pods](#-part-5-networking-internals)
7. [Security Internals: How Pods Authenticate to the API](#-part-6-security-internals)
8. [Kubernetes at Scale: Limits and Tuning](#-part-7-kubernetes-at-scale)
9. [The Raw Kubernetes API: Beyond kubectl](#-part-8-the-raw-kubernetes-api)
10. [Hands-On Diagnostic Exercises](#-part-9-hands-on-diagnostic-exercises)
11. [Interview-Style Questions with Full Answers](#-part-10-interview-style-questions)
12. [Glossary of Key Terms](#-glossary-of-key-terms)

---

## 🏗 Part 0: The Big Picture

Before diving into internals, let's recall the main components of a Kubernetes cluster and how they interact. Think of a Kubernetes cluster like a **company**:

| Kubernetes Component | Company Analogy | What It Does |
|---|---|---|
| **API Server** | The reception desk | Every request must pass through it. It checks who you are, whether you're allowed, and records what happens. |
| **etcd** | The filing cabinet | A database that stores the desired state of everything in the cluster (pods, services, secrets, etc.). |
| **Scheduler** | The assignment manager | Decides *which* worker node a new pod should run on. |
| **Controller Manager** | The supervisors | Continuously watches the cluster and fixes differences between "what we want" and "what we have." |
| **kubelet** | The floor worker on each node | Runs on every node, receives pod assignments, and ensures containers are actually running. |
| **kube-proxy** | The mailroom | Handles networking rules so traffic reaches the right pods. |
| **kubelet / container runtime** | The machinery | The actual engine (containerd, CRI-O) that starts and stops containers. |

**The flow of a request** (we'll go deep on this in Part 1):

```
You (kubectl)
   │  sends request
   ▼
API Server  ──authenticates──► checks RBAC ──► runs admission webhooks
   │                                                        │
   ▼                                                        ▼
etcd (stores it) ◄────────────────────────────────── validates & persists
   │
   ▼
Watch event broadcast ──► Scheduler sees new pod ──► assigns node
   │
   ▼
kubelet on that node sees assignment ──► starts container
```

Everything in Kubernetes revolves around the **API Server** talking to **etcd**, and **controllers** watching for changes. Keep this picture in mind throughout.

---

## 🔑 Part 1: The API Server Request Lifecycle

### The Analogy: Airport Security

Think of the API server as **airport security**. When you walk in (send a request), you don't go straight to your gate. You pass through several checkpoints:

1. **Show your passport** → Authentication (who are you?)
2. **Visa check** → Authorization (are you allowed to do this?)
3. **Customs inspection** → Admission (does your request meet our rules? should we modify it?)
4. **Boarding pass stamped** → Persistence (recorded in the system)
5. **Announcement over the PA** → Watch event (notify everyone watching)

### The 12 Steps in Detail

When you run `kubectl apply -f deployment.yaml`, here's everything that happens:

```
Step 1:  TLS Handshake              — secure connection established
Step 2:  Authentication             — "who are you?"
Step 3:  Audit (RequestReceived)    — log that the request arrived
Step 4:  Impersonation check       — are you acting as someone else?
Step 5:  Authorization (RBAC)       — "are you allowed to do this?"
Step 6:  Admission (Mutating)       — webhooks can modify the request
Step 7:  Object schema validation   — is the YAML valid?
Step 8:  Admission (Validating)     — webhooks can reject the request
Step 9:  Audit (ResponseStarted)    — log that we're responding
Step 10: Persistence to etcd        — save to the database
Step 11: Watch event broadcast      — notify everyone watching
Step 12: Response to client         — kubectl gets the result
```

Let's walk through the important ones.

---

#### Step 1 & 2: TLS + Authentication — "Who Are You?"

**What happens:** Your `kubectl` reads your `kubeconfig` file to find your cluster's address and your client certificate. It opens an encrypted (TLS) connection to the API server and presents your certificate. The API server verifies the certificate and extracts your identity.

Your certificate contains two key pieces of information:
- **CN (Common Name)** → your username (e.g., `kubernetes-admin`)
- **O (Organization)** → your group (e.g., `system:masters`)

Think of it like an ID card: the CN is your name, the O is your department.

**Authentication methods (tried in this order):**
1. X.509 client certificates (this is what your kubeconfig uses)
2. Bearer tokens (service accounts use these)
3. OIDC tokens (single sign-on, like Google login)
4. Webhook token authentication (custom token validation)
5. Anonymous (if enabled — usually a bad idea in production)

**What happens on failure?** The API server returns `401 Unauthorized`. The audit log records the failure. Authorization (RBAC) is *never* checked if authentication fails — you must prove who you are first.

**Try it yourself:**

```bash
# See what authentication methods your API server has enabled
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml \
  | grep -E "client-ca|token-auth|oidc"

# Inspect the identity stored in your kubeconfig certificate
kubectl config view --raw \
  -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d \
  | openssl x509 -noout -subject
# Output: subject=O=system:masters, CN=kubernetes-admin
#         O = group, CN = username
```

---

#### Step 3: Audit Logging — "Recording Everything"

**The analogy:** A security camera that records everyone who enters the building, even people who get turned away.

Every single request generates **two** audit events:
1. **RequestReceived** — the moment the request arrives (before any processing)
2. **ResponseComplete** — when the response is sent back (after processing)

This means even *rejected* requests show up in the audit logs — which is critical for security, because you can see failed login attempts.

**Key fields in an audit log entry:**

| Field | Meaning | Example |
|---|---|---|
| `auditID` | Links the RequestReceived and ResponseComplete events | `a1b2-c3d4-...` |
| `user` | Who made the request (after authentication) | `kubernetes-admin` |
| `verb` | What they tried to do | `create`, `get`, `list`, `delete`, `watch` |
| `objectRef` | What resource was targeted | `pods/default/my-pod` |
| `responseStatus.code` | HTTP status code returned | `200`, `401`, `403` |
| `requestReceivedTimestamp` | When the request arrived | `2024-01-01T10:00:00Z` |
| `stageTimestamp` | When this stage completed | `2024-01-01T10:00:00.5Z` |

**Try it yourself:**

```bash
# Read the last few raw audit events (if audit logging is enabled)
docker exec k8s-mastery-control-plane \
  tail -5 /var/log/kubernetes/audit.log 2>/dev/null \
  | python3 -m json.tool | head -40
```

---

#### Step 4: Impersonation — "Acting As Someone Else"

**The analogy:** A manager who can temporarily wear an employee's badge to test whether that employee can access a certain room.

Impersonation lets you make a request *as if* you were another user or service account. This is incredibly useful for:
- **Debugging RBAC:** "Can this service account actually read ConfigMaps?"
- **Admin testing:** Performing an operation as a specific service to verify it works.

When you impersonate, the API server:
1. Extracts your *real* identity first
2. Checks whether you have the `impersonate` permission on that user/group
3. If allowed, proceeds with the *impersonated* identity for all RBAC checks

**Try it yourself:**

```bash
# Make a request as if you were a specific service account
kubectl get pods \
  --as=system:serviceaccount:production:app-sa \
  --as-group=system:authenticated

# Check who has the power to impersonate others
kubectl get clusterrolebindings -o json \
  | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
```

In the audit log, impersonation shows **both** identities:
- `user.username`: your real identity (`kubernetes-admin`)
- `impersonatedUser.username`: who you're pretending to be (`system:serviceaccount:production:app-sa`)

---

#### Step 5: Authorization (RBAC) — "Are You Allowed?"

**The analogy:** A building with rooms. Even after you're identified at the front desk (authentication), you need a keycard that specifically grants access to the room you want to enter.

RBAC stands for **Role-Based Access Control**. After authentication confirms *who* you are, RBAC decides *what you're allowed to do*.

**Important:** RBAC is not the only authorizer. Multiple authorizers can chain together. A typical setup is `--authorization-mode=Node,RBAC`:
- **Node authorizer:** Special rules for the kubelet. A kubelet can only read pods/secrets/configmaps for pods running on *its own node*. This prevents a compromised node from accessing data for pods on *other* nodes.
- **RBAC authorizer:** The general-purpose rules for users and service accounts.

**How RBAC evaluation works (step by step):**
1. Collect all Roles/ClusterRoles bound to the user's identity (user, groups, service account)
2. For each rule in those roles, check: does the `verb` + `resource` + `apiGroup` match the request?
3. If **any** rule matches → **ALLOW**
4. If **no** rule matches → **DENY**

**Key concept:** RBAC is **allow-only**. There are no explicit "deny" rules. If you have no matching allow rule, you're denied by default. This is safer — you can't accidentally lock yourself out with a conflicting deny.

**Try it yourself:**

```bash
# See the detailed RBAC decision-making (very verbose)
kubectl get pods -v=8 2>&1 | grep -E "Response Status:|RBAC"

# Ask the API server directly: "would user 'chetan' be allowed to create pods?"
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
# Response: status.allowed: true  (or false)
```

The `SubjectAccessReview` is the same mechanism the API server uses internally — you're just calling it yourself. It's a fantastic debugging tool when you're trying to figure out *why* a user can or can't do something.

---

#### Step 10: Persistence to etcd — "Saving to the Database"

**The analogy:** A shared Google Doc where multiple people might be editing at the same time. To prevent overwriting each other, the system tracks a "version number" and only saves your changes if the document hasn't been modified since you last read it.

etcd stores every Kubernetes object with a **resourceVersion** — a number that increments every time *anything* is written. When a controller reads an object, modifies it, and writes it back, it says: *"update this object, but only if it's still at revision 1234."*

- If nothing else changed it in between → success ✓
- If something *did* change it → `409 Conflict`, and the controller must re-read and retry

This is called **optimistic concurrency** (as opposed to pessimistic locking, where you'd lock the object and block everyone else). It works because:
- Kubernetes controllers are designed to be **idempotent** — running the same operation twice produces the same result
- Conflicts are rare in practice because different controllers manage different resources
- Retries use exponential backoff (via `RetryOnConflict` in the code)

**Why this matters:** If two controllers tried to update the same object at the same time without this protection, one would silently overwrite the other's changes. The resourceVersion prevents that.

**Try it yourself:**

```bash
# See the resourceVersion on any object
kubectl get deployment url-shortener -o jsonpath='{.metadata.resourceVersion}'
# Output: 89234

# Try to update with a stale (wrong) resourceVersion — it will fail
kubectl patch deployment url-shortener \
  --type=merge \
  -p '{"metadata":{"resourceVersion":"1"},"spec":{"replicas":5}}'
# Error: Operation cannot be fulfilled on deployments.apps "url-shortener":
# the object has been modified; please apply your changes to the latest
# version and try again
```

That error is *optimistic concurrency working correctly* — it prevented a conflicting write.

---

#### Step 11: Watch Event Broadcast — "Notifying Everyone"

**The analogy:** A stock ticker. Instead of investors repeatedly calling the exchange to ask "did the price change?", they subscribe once and get notified instantly when it does.

When an object is saved to etcd, the API server doesn't just sit there. It broadcasts a **watch event** to every controller that's "watching" for changes to that resource type. This is how controllers react instantly.

**How the watch mechanism works (under the hood):**

1. **etcd watch:** The API server maintains a persistent watch on all keys in etcd (watching `/registry/` with a prefix)
2. **WatchCache:** The API server keeps a ring buffer (default: 100 recent events per resource type) so it can serve events without hitting etcd directly
3. **Event arrives from etcd →** the API server decodes it, updates its cache, and fans it out to every active watcher
4. **Controller receives event →** puts it on a work queue (which deduplicates and rate-limits), then reconciles

```
etcd ──► API Server (WatchCache) ──► Controller A
                                 ──► Controller B
                                 ──► Controller C
```

**Why this design?** etcd is not designed for high fan-out. Without the cache, 100 controllers watching pods would mean 100 concurrent etcd streams. With the cache, there's *one* stream to etcd, and the API server fans events out from its own memory. This reduces etcd read pressure by 10–100x in busy clusters.

**Try it yourself:**

```bash
# See watch in action (very verbose, but educational)
kubectl get pods --watch -v=9 2>&1 | grep -E "Watch|resourceVersion|MODIFIED"

# What you'll see:
# Establishing watch on /api/v1/namespaces/default/pods?watch=true&resourceVersion=89234
# {"type":"MODIFIED","object":{"kind":"Pod",...}}
```

---

## 🎯 Part 2: The Scheduler

### The Analogy: A Restaurant Seating Host

Imagine a busy restaurant. The **scheduler** is the host who decides where each new party (pod) sits:
- **Filtering:** "This party of 8 can't fit at a 2-person table" (eliminate nodes that can't fit the pod)
- **Scoring:** "Party prefers a window seat, so rank window tables higher" (rank the remaining nodes)
- **Binding:** "Seat them at table 12" (assign the pod to a node)

### The Scheduler Is a Plugin Framework

The Kubernetes scheduler is not one monolithic block of logic. It's a **plugin framework** — every decision is made by composable plugins that hook into specific "extension points":

```
┌──────────────────────────────────────────────────────────┐
│                    SCHEDULER PIPELINE                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. FILTER (eliminate nodes)                             │
│     "Does this node have enough CPU? Does it match       │
│     the node selector? Is it tainted?"                   │
│                                                          │
│  2. SCORE (rank remaining nodes)                          │
│     "Which node is the BEST fit? Give each a score."      │
│                                                          │
│  3. POSTFILTER (what if no node passed?)                  │
│     "Should we evict lower-priority pods to make room?"  │
│     (this is called preemption)                          │
│                                                          │
│  4. RESERVE (claim resources before binding)              │
│     "Hold this node's resources so nobody else takes it"  │
│                                                          │
│  5. PERMIT (last chance to delay or reject)              │
│     "Wait — is there a reason to hold off on binding?"    │
│                                                          │
│  6. BIND (write the assignment)                           │
│     "Tell the API server: this pod goes on node X"        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

Each of these phases runs plugins. You can enable, disable, or write your own.

### Scheduler Performance at Scale

**The problem:** In a cluster with 5,000 nodes, scoring *every* node for every pod would be slow.

**The solution:** The scheduler doesn't score all nodes. It uses **percentage-based filtering**:

| Cluster Size | Nodes Scored |
|---|---|
| Fewer than 100 nodes | All nodes (100%) |
| 100–5,000 nodes | 50% of nodes (minimum 100) |
| More than 5,000 nodes | 10% of nodes (minimum 100) |

**What this means:** At scale, scheduling is **not globally optimal** — it's "good enough," optimized for throughput rather than perfect placement. This is a deliberate trade-off: faster scheduling decisions at the cost of occasionally picking a slightly-less-than-perfect node.

You can tune this with the `--percentage-of-nodes-to-score` flag.

**Try it yourself:**

```bash
# See your scheduler's configuration
kubectl get configmap -n kube-system \
  $(kubectl get configmap -n kube-system | grep scheduler | awk '{print $1}') \
  -o yaml 2>/dev/null \
  || docker exec k8s-mastery-control-plane \
       cat /etc/kubernetes/manifests/kube-scheduler.yaml \
       | grep -E "config|profile"
```

### Writing Your Own Scheduler Plugin (Conceptual Example)

You can write a custom plugin that scores nodes based on a custom label. For example, a plugin that prefers nodes labeled `tier=premium`:

```go
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

// Score is called for each node that passed filtering.
// Return a higher number = more preferred.
func (p *CustomTierPlugin) Score(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeName string,
) (int64, *framework.Status) {

    nodeInfo, err := p.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
    if err != nil {
        return 0, framework.NewStatus(
            framework.Error,
            fmt.Sprintf("getting node %s info: %v", nodeName, err),
        )
    }

    node := nodeInfo.Node()
    tier, exists := node.Labels["tier"]

    if !exists {
        return 0, nil // no tier label = score 0 (neutral)
    }

    // Score based on the tier label
    switch tier {
    case "premium":
        return 100, nil // highest score = most preferred
    case "standard":
        return 50, nil
    case "spot":
        return 10, nil // lowest score = last resort
    default:
        return 0, nil
    }
}

func (p *CustomTierPlugin) ScoreExtensions() framework.ScoreExtensions { return nil }
func (p *CustomTierPlugin) Name() string                                { return Name }

// New registers the plugin with the scheduler framework
func New(obj runtime.Object, h framework.Handle) (framework.Plugin, error) {
    return &CustomTierPlugin{handle: h}, nil
}
```

**Don't worry if the Go code looks intimidating.** The key takeaway is: *every scheduling decision is a pluggable function*. If Kubernetes' default behavior doesn't suit you, you can write your own logic.

---

## 🗄 Part 3: etcd Internals

### The Analogy: The Cluster's Filing Cabinet

**etcd** is a distributed key-value store — it's the "source of truth" for the entire cluster. Everything Kubernetes knows (pods, services, secrets, deployments, configmaps) lives in etcd. If etcd is lost and there's no backup, your cluster's state is gone.

### How Kubernetes Uses etcd: The Data Model

Every Kubernetes object is stored at a predictable key path:

```
/registry/<resource-type>/<namespace>/<name>
```

For example:

| Kubernetes Object | etcd Key |
|---|---|
| Namespace "default" | `/registry/namespaces/default` |
| Pod "my-app" in "production" | `/registry/pods/production/my-app-xyz` |
| Secret "db-password" in "production" | `/registry/secrets/production/db-password` |
| Deployment "url-shortener" | `/registry/deployments/production/url-shortener` |
| ClusterRoleBinding "cluster-admin" | `/registry/clusterrolebindings/cluster-admin` |

**Try it yourself** (see all the keys stored in etcd):

```bash
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry --prefix --keys-only 2>/dev/null | head -50
"

# Output shows keys like:
# /registry/apiregistration.k8s.io/apiservices/v1.
# /registry/clusterrolebindings/cluster-admin
# /registry/deployments/production/url-shortener
# /registry/namespaces/default
# /registry/pods/production/url-shortener-xxx
# /registry/secrets/production/app-secrets
```

### Why This Matters

Understanding the key structure helps you:
- **Debug** by reading raw data from etcd when the API server is misbehaving
- **Back up** by snapshotting etcd (you know exactly what you're saving)
- **Understand performance** — each key is one entry, and etcd's performance depends on the total number of keys

### Compaction and Defragmentation

**The problem:** By default, etcd keeps **every historical revision** of every object. Every time a pod's status changes, a new revision is saved. Over time, this causes the etcd database to grow unboundedly.

**The solution:** Two maintenance operations:

1. **Compaction:** Removes old revisions, keeping only the latest version of each key. Think of it like emptying the "trash" in your filing cabinet — you keep the current documents but throw away old drafts.
2. **Defragmentation:** Reclaims the disk space freed by compaction. After compacting, the freed space isn't automatically returned to the OS until you defrag.

**Try it yourself:**

```bash
docker exec k8s-mastery-control-plane bash -c "
# 1. Get the current revision
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

# 2. Compact to current revision (removes all historical data)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact \$REV 2>/dev/null

# 3. Defragment — reclaim disk space after compaction
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag 2>/dev/null
echo 'Compaction and defrag complete'
"
```

### etcd Performance Tuning (Production Flags)

For production clusters, these flags (set in `/etc/kubernetes/manifests/etcd.yaml`) are critical:

| Flag | Default | Recommended | Why |
|---|---|---|---|
| `--heartbeat-interval` | 100ms | 100ms | Time between leader heartbeats |
| `--election-timeout` | 1000ms | 1000ms | Time before a follower calls a new election. **Rule of thumb:** should be 10× the heartbeat interval |
| `--quota-backend-bytes` | 2GB | 8GB | Maximum etcd database size |
| `--auto-compaction-mode` | (disabled) | `periodic` | Automatically compact old data |
| `--auto-compaction-retention` | — | `1h` | Compact hourly |
| `--snapshot-count` | 10000 | 10000 | Snapshot to disk every 10,000 writes |

**Disk requirements (critical!):** etcd is extremely sensitive to disk latency because every write must be synced to disk before it's acknowledged.
- Write latency budget: **under 10ms**
- Sequential write speed: **over 50 MB/s**
- **Always use dedicated NVMe disks** — never share etcd's disk with other workloads

**Try it yourself** (check etcd performance):

```bash
# Check etcd disk performance
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  check perf 2>/dev/null || echo 'check perf not available in this version'
"

# See etcd's current status
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table 2>/dev/null
"
```

---

## 🤖 Part 4: The Controller Manager

### The Analogy: A Team of Supervisors

The **kube-controller-manager** is a single binary that runs **30+ different controllers** inside it. Think of it as a building with many supervisors, each watching a different area:

| Controller | What It Watches | What It Does |
|---|---|---|
| **Deployment** | Deployments | Creates/deletes ReplicaSets to match desired replica count |
| **ReplicaSet** | ReplicaSets | Creates/deletes Pods to match desired count |
| **Node** | Node health | Marks nodes as unhealthy if they stop reporting |
| **Endpoint** | Services & their pods | Updates Service endpoints when pods come/go |
| **ServiceAccount** | Namespaces | Auto-creates a default service account per namespace |
| **Garbage Collection** | Object ownership | Deletes child objects when parents are deleted |
| **Job** | Jobs | Tracks completion of batch workloads |

**Why one binary?** It simplifies deployment and reduces resource usage. All controllers share the same connection to the API server.

### Leader Election: Only One Boss at a Time

**The problem:** If you run multiple controller-manager replicas (for high availability), they'd all try to do the same work simultaneously — creating duplicate pods, fighting over changes.

**The solution:** **Leader election.** Only one replica is "active" at a time. The others stand by. If the leader dies, another takes over within seconds.

Kubernetes uses **Lease objects** for this. Think of it like a baton in a relay race — whoever holds the baton is the leader.

```
kubectl get lease -n kube-system
# NAME                       HOLDER                                AGE
# kube-controller-manager    k8s-mastery-control-plane_xxx        5d
# kube-scheduler             k8s-mastery-control-plane_xxx        5d
```

**How it works (the timing):**
- `leaseDuration` (15s): How long the lease is valid
- `renewDeadline` (10s): The leader must renew before this time or it's considered dead
- `retryPeriod` (2s): How often followers check / the leader renews

If the leader stops renewing (e.g., it crashed), a follower acquires the lease after the `leaseDuration` expires, and takes over.

**Try it yourself:**

```bash
# See all controllers the controller-manager is running
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-controller-manager \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep "Starting controller" | head -30

# See who currently holds the leader lease
kubectl get lease -n kube-system
kubectl get lease kube-controller-manager \
  -n kube-system \
  -o jsonpath='{.spec.holderIdentity}'
```

---

## 🕸 Part 5: Networking Internals

### The Analogy: A Mail Sorting System

When traffic comes into a Kubernetes Service, it needs to reach one of the pods behind that service. This is like a post office sorting mail:
- The **Service** is the building's main address (a single IP that everyone sends to)
- The **pods** are individual apartments behind that address
- **kube-proxy** is the mail sorter — it reads the destination and routes each "letter" (packet) to the right apartment

### How Services Work: iptables Chains

By default, kube-proxy uses **iptables** rules to implement Services. When you create a ClusterIP Service, kube-proxy creates a chain of iptables rules on **every node**.

**The packet's journey** (for a Service called "url-shortener" in the "production" namespace):

```
Packet arrives destined for Service IP
         │
         ▼
┌─────────────────────┐
│  KUBE-SERVICES       │  Entry point — matches the Service's ClusterIP:Port
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  KUBE-SVC-<hash>     │  Service chain — random load balancing
└─────────┬───────────┘
          │
    ┌─────┴─────┬─────────────┐
    ▼           ▼             ▼
┌────────┐ ┌────────┐   ┌────────┐
│ KUBE-  │ │ KUBE-  │   │ KUBE-  │
│ SEP-1  │ │ SEP-2  │   │ SEP-3  │
└───┬────┘ └───┬────┘   └───┬────┘
    │          │             │
    ▼          ▼             ▼
  Pod 1      Pod 2         Pod 3
  (DNAT)     (DNAT)        (DNAT)
```

**The probability math** (how load balancing works):

iptables uses **stateless random selection** with decreasing probabilities:
- Endpoint 1: `1/3 = 0.333` probability
- Endpoint 2: `1/2 of the remaining = 0.333` probability
- Endpoint 3: `1/1 of the remaining = 0.333` probability

Each pod ends up with equal probability — it's round-robin via probability.

**Try it yourself:**

```bash
# See the KUBE-SERVICES chain (the entry point)
docker exec k8s-mastery-control-plane \
  iptables -t nat -L KUBE-SERVICES -n --line-numbers 2>/dev/null | head -20

# Find a specific Service's chain
docker exec k8s-mastery-control-plane bash -c "
SVC_CHAIN=\$(iptables -t nat -L KUBE-SERVICES -n 2>/dev/null \
  | grep 'url-shortener\|kubernetes' | head -1 | awk '{print \$2}')
echo \"Chain: \$SVC_CHAIN\"
[ -n \"\$SVC_CHAIN\" ] && iptables -t nat -L \$SVC_CHAIN -n 2>/dev/null
"
```

### IPVS Mode: Why It's Faster

**The problem with iptables:** At scale, iptables rules are linear — every packet must traverse the rules top to bottom. With 10,000 Services, there are 40,000+ rules to check per packet. That's slow.

**The IPVS solution:** IPVS (IP Virtual Server) uses **kernel hash tables** — lookup is O(1) (constant time) regardless of how many services exist.

| Mode | Lookup Time | At 10,000 Services |
|---|---|---|
| iptables | O(n) — linear traversal | 40,000+ rules per packet |
| IPVS | O(1) — hash table lookup | Same as 1 service |

**Try it yourself:**

```bash
# Check which mode kube-proxy is using
kubectl get configmap -n kube-system kube-proxy -o yaml | grep mode

# See IPVS virtual servers (if IPVS mode is enabled)
docker exec k8s-mastery-worker \
  ipvsadm -Ln 2>/dev/null | head -30 \
  || echo "IPVS not enabled on this cluster (using iptables mode)"

# To enable IPVS (requires kube-proxy restart):
kubectl edit configmap kube-proxy -n kube-system
# Change: mode: ""  →  mode: "ipvs"
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### eBPF: The Future Beyond iptables

**eBPF** (extended Berkeley Packet Filter) is the next generation. Tools like **Cilium** replace kube-proxy entirely by running programs directly in the Linux kernel.

**Why eBPF is a game-changer:**
- **No userspace overhead** — everything happens in the kernel
- **O(1) service routing** — hash map lookups, like IPVS but more flexible
- **NetworkPolicy enforced in the kernel** — packets are dropped *before* they leave the network card
- **Rich observability** — you can see exactly which packets are dropped and why

**Try it yourself:**

```bash
# Check if Cilium (eBPF) is installed
kubectl get pods -n kube-system -l k8s-app=cilium 2>/dev/null \
  || echo "Cilium not installed"

# With Cilium, there are far fewer iptables rules for services
docker exec k8s-mastery-worker \
  iptables -t nat -L KUBE-SERVICES -n 2>/dev/null | wc -l
# Much fewer rules — Cilium handles service routing in eBPF instead

# Cilium observability commands (if installed):
# cilium monitor --type drop   # see dropped packets in real time
# cilium endpoint list          # all managed endpoints
# cilium policy get             # all NetworkPolicies in Cilium format
```

---

## 🔐 Part 6: Security Internals

### How Service Account Tokens Work

**The analogy: A Temporary Visitor Badge**

In the old days, Kubernetes gave each service account a permanent, static token — like a visitor badge that never expired. If that badge was stolen, the attacker had access forever.

Modern Kubernetes uses **projected service account tokens** — temporary, automatically-rotating JWT tokens. Think of it like a visitor badge that expires every hour and is silently renewed before it does.

### What's Inside a Token

A service account token is a **JWT** (JSON Web Token) with these claims:

| Claim | Meaning | Example |
|---|---|---|
| `iss` | Issuer (who created it) | `https://kubernetes.default.svc.cluster.local` |
| `aud` | Audience (who it's for) | `https://kubernetes.default.svc.cluster.local` |
| `exp` | Expiration time | (1 hour from issue) |
| `iat` | Issued at | (now) |
| `sub` | Subject (who the token represents) | `system:serviceaccount:default:my-sa` |
| `kubernetes.io` | K8s-specific metadata | namespace, pod name, SA UID |

**How rotation works:**
1. Kubelet mounts the token into the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`
2. Before the token expires, kubelet writes a **fresh token** to the same file
3. Applications that read the file on each request automatically get the fresh token — no restart needed

**Try it yourself:**

```bash
# Read the projected token from inside a pod and decode it
kubectl exec -n default \
  $(kubectl get pod -l app=url-shortener \
    -o jsonpath='{.items[0].metadata.name}') \
  -- cat /var/run/secrets/kubernetes.io/serviceaccount/token \
  2>/dev/null | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

### TokenReview: How the API Server Validates Tokens

When a component (like an ingress controller or a webhook) receives a token, it can ask the API server: "Is this token valid, and who does it belong to?"

This is done via a **TokenReview** object — the same mechanism the API server uses internally.

**Try it yourself:**

```bash
# Ask the API server to validate a token
kubectl create -f - <<EOF
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: \$(kubectl exec \
    \$(kubectl get pod -l app=url-shortener \
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

---

## 📊 Part 7: Kubernetes at Scale

### The Analogy: A City's Infrastructure

As a city grows, the systems that worked for a small town break down. Traffic lights that manage 10 intersections fine become gridlocked at 1,000 intersections. Similarly, Kubernetes clusters at scale need tuning that small clusters don't.

### API Server Tuning

For large clusters, tune these flags in `kube-apiserver.yaml`:

| Flag | Default | Large Cluster | Purpose |
|---|---|---|---|
| `--max-requests-inflight` | 400 | 800 | Concurrent non-mutating (read) requests |
| `--max-mutating-requests-inflight` | 200 | 400 | Concurrent mutating (write) requests |
| `--watch-cache-sizes` | (auto) | `pods#1000` | Larger cache for frequently-watched resources |
| `--default-watch-cache-size` | 100 | 100 | Default cache size per resource type |
| `--request-timeout` | 60s | 60s | Max time for a non-watch request |
| `--enable-priority-and-fairness` | true | true | Enable APF (see below) |

### API Priority and Fairness (APF)

**The problem APF solves:** Before APF, the API server had two simple queues — one for reads (400 slots) and one for writes (200 slots). A single misbehaving controller could fill all 400 read slots, **starving** everyone else — `kubectl` would hang, the scheduler couldn't get pod info, etc.

**How APF fixes it:** APF replaces the two queues with a sophisticated system:
1. **FlowSchema** — classifies each request (who is making it, what type?)
2. **PriorityLevelConfiguration** — each level has its own concurrency limit and queue

```
Request arrives
      │
      ▼
FlowSchema classifies it
      │
      ├──► "exempt" level          (system:masters — bypass all limits)
      ├──► "cluster-admin" level   (high priority for admin operations)
      └──► "catch-all" level       (everyone else — strict limits)
```

Within each level, **fair queuing** (via "shuffle sharding") ensures each "flow" (a user + resource + verb combination) gets its own virtual queue — so one bad actor can't monopolize a priority level.

**Try it yourself:**

```bash
# See APF configuration
kubectl get flowschemas
kubectl get prioritylevelconfigurations

# Inspect specific flow schemas
kubectl describe flowschema catch-all
kubectl describe flowschema exempt                         # system:masters bypass APF
kubectl describe flowschema kube-system-service-accounts

# See API server request metrics
kubectl get --raw /metrics | grep apiserver_request_total | head -10
kubectl get --raw /metrics | grep apiserver_current_inflight_requests
```

### Scalability Benchmarks — Know These Numbers

These are the **tested limits** Kubernetes is designed for. Memorize them:

| Metric | Limit |
|---|---|
| Max nodes per cluster | 5,000 |
| Max pods per cluster | 150,000 |
| Max pods per node | 110 (default) |
| Max Services | 10,000 |
| Max namespaces | 10,000 |
| API server P99 response time (SLO) | 1 second |
| Pod startup time without image pull (SLO) | 5 seconds |
| etcd recommended max size | 8 GB |
| etcd size before degradation | ~2 GB (without compaction) |
| Watch cache events per resource | 100 (default ring buffer) |
| Leader election duration | 15 seconds |

---

## 🌐 Part 8: The Raw Kubernetes API

### The Analogy: Driving Manual vs. Automatic

Using `kubectl` is like driving an automatic transmission — it's convenient and handles a lot for you. But sometimes you need to understand what's happening under the hood (manual transmission) for debugging or automation.

**Key insight:** The Kubernetes API is just **REST over HTTPS**. `kubectl` is simply a client that calls this API. You can call it directly with `curl` if you set up a proxy.

### Setting Up Direct Access

```bash
# Start a proxy that handles authentication for you
kubectl proxy --port=8001 &
```

Now `http://localhost:8001` forwards to your API server, with your credentials automatically attached.

### Common Operations via Raw API

```bash
# List all pods in the default namespace
curl -s http://localhost:8001/api/v1/namespaces/default/pods \
  | jq '.items[].metadata.name'

# Get a specific pod's resourceVersion
curl -s http://localhost:8001/api/v1/namespaces/default/pods/my-pod \
  | jq .metadata.resourceVersion

# Watch pods (this opens a long-lived streaming connection)
curl -s "http://localhost:8001/api/v1/namespaces/default/pods?watch=true" \
  | while read line; do
      echo "$line" | jq -r '"\(.type) \(.object.metadata.name)"'
    done

# Patch a deployment (change replica count)
curl -s \
  -X PATCH \
  -H "Content-Type: application/merge-patch+json" \
  http://localhost:8001/apis/apps/v1/namespaces/default/deployments/url-shortener \
  -d '{"spec":{"replicas":5}}'

# List all API groups
curl -s http://localhost:8001/apis | jq '.groups[].name'

# List all resources in the "apps" group
curl -s http://localhost:8001/apis/apps/v1 \
  | jq '.resources[].name'

# Server-side apply via raw API
curl -s \
  -X PATCH \
  -H "Content-Type: application/apply-patch+yaml" \
  "http://localhost:8001/apis/apps/v1/namespaces/default/deployments/url-shortener?fieldManager=my-tool&force=true" \
  -d @deployment.yaml
```

**Why learn this?** Understanding the raw API makes you powerful for:
- **Debugging** when `kubectl` behaves unexpectedly (you can see exactly what's sent/received)
- **Automation** in languages where the Kubernetes client library isn't available
- **Understanding** how controllers and operators actually interact with the cluster

---

## 🛠 Part 9: Hands-On Diagnostic Exercises

The best way to internalize these concepts is to try them. Here are four guided exercises.

### Exercise 1: Trace a Request Through the API Server

**Goal:** See every step that happens when you run a `kubectl` command.

```bash
# Enable verbose logging (level 10 = extremely detailed)
kubectl get pods -v=10 2>&1 | head -80

# What you'll see:
# I0101 ... GET https://localhost:6443/api/v1/namespaces/default/pods
# I0101 ... Response Status: 200 OK in 8ms
# I0101 ... Response Headers: ...
# I0101 ... Response Body: {"kind":"PodList",...}

# Now trace a specific request through the audit log
kubectl create secret generic trace-test \
  --from-literal=key=value \
  -n default

# Find it in the audit log
docker exec k8s-mastery-control-plane \
  grep 'trace-test' /var/log/kubernetes/audit.log 2>/dev/null \
  | python3 -m json.tool \
  | grep -E '"verb"|"user"|"requestObject"|"responseStatus"' \
  | head -20
```

### Exercise 2: Read etcd Directly

**Goal:** See what Kubernetes actually stores in its database.

```bash
# List all pods in the default namespace, directly from etcd
docker exec k8s-mastery-control-plane bash -c "
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default --prefix --keys-only 2>/dev/null
"

# Read a specific pod's raw data (it's binary protobuf — use 'strings' to extract readable parts)
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

### Exercise 3: Force a Scheduling Failure and Read the Logs

**Goal:** See how the scheduler logs its decisions when a pod can't be scheduled.

```bash
# Create a pod with impossible requirements
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
        cpu: "999"     # impossible amount
        memory: "999Gi"
EOF

# Read what the scheduler says about it
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-scheduler \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep scheduler-debug | tail -10

# Read the pod's condition (why it's stuck)
kubectl get pod scheduler-debug -o yaml | grep -A 10 conditions
kubectl describe pod scheduler-debug | grep -A 5 Events

# Clean up
kubectl delete pod scheduler-debug
```

### Exercise 4: Observe Watch Cache Behavior

**Goal:** See how watch events are delivered in real time, and what happens when the cache is too old.

```bash
# Start a watch on pods
kubectl get pods --watch \
  -o jsonpath='{.type} {.object.metadata.name}' \
  2>/dev/null &
WATCH_PID=$!

# Create and delete a pod — the watch should receive events instantly
kubectl run watch-test --image=nginx:1.25 -- sleep 60
sleep 5
kubectl delete pod watch-test --force --grace-period=0

# Stop the watch
sleep 2
kill $WATCH_PID 2>/dev/null

# Now try with an EXPIRED resourceVersion — Kubernetes returns "410 Gone"
kubectl get pods \
  --resource-version=1 \
  --watch 2>&1 | head -5
# Error: 410 Gone — the cache has expired, you must relist from the current state
```

---

## 💬 Part 10: Interview-Style Questions

These are the kind of questions that distinguish someone who *uses* Kubernetes from someone who *understands* it. Try to answer each one before reading the answer.

---

### Q1: Walk me through every step when you `kubectl apply` a Deployment, from keystroke to running pod.

**Answer:**

1. `kubectl` reads your `kubeconfig` to find the cluster endpoint and your client certificate.
2. It serializes the YAML to JSON and sends a PATCH (server-side apply) or PUT/POST request over TLS to the API server.
3. **API server authenticates** the client cert — extracts your username and groups.
4. **Audit event** recorded (`RequestReceived`).
5. **RBAC is checked** — does this user have `create`/`patch` permission on deployments in this namespace?
6. **Mutating webhooks run** — e.g., Istio sidecar injection, label injection, resource defaulters.
7. **Schema validation** — is the spec valid?
8. **Validating webhooks run** — e.g., Gatekeeper, Kyverno policy checks.
9. **Persisted to etcd.**
10. **Watch event broadcast** — anyone watching deployments gets notified.
11. **Deployment controller** sees the new object → creates a **ReplicaSet**.
12. **ReplicaSet controller** sees the ReplicaSet → creates Pods (they start as `Pending`, no node assigned).
13. **Scheduler** sees the Pending pods → filters nodes, scores nodes, assigns each pod to a node → writes the assignment to the API server.
14. **kubelet** on each assigned node sees its pod via watch → calls containerd to pull the image and start the container.
15. **kubelet** updates pod status to `Running`.
16. **readinessProbe passes** → **Endpoints controller** adds pod IPs to the Service's endpoints.
17. **Traffic can now reach the pods.**

---

### Q2: Why does Kubernetes use optimistic concurrency instead of pessimistic locking for etcd writes?

**Answer:**

**Pessimistic locking** (lock before reading, hold until writing) in a distributed system causes problems:
- If the lock holder dies → the whole cluster hangs
- Network partitions cause deadlocks
- High contention serializes throughput (everything waits in line)

**Optimistic concurrency** trades blocking for retries:
1. Read the object, remember its `resourceVersion`
2. Compute your changes
3. Write back: "update this *only if* the resourceVersion still matches"
4. If another write happened concurrently → you get a `409 Conflict`, re-read, re-apply, retry

**Why this works for Kubernetes:**
- Controllers are designed to be **idempotent** — running the reconcile loop multiple times produces the same result
- Conflicts are rare because each controller owns different resources
- `RetryOnConflict` in client-go implements exponential backoff for these retries

**The tradeoff:** Under extremely high contention, controllers retry frequently. But in practice, this is rare and the system stays responsive.

---

### Q3: What is API Priority and Fairness (APF) and why was it needed?

**Answer:**

**Before APF:** The API server had two simple queues — one for mutating requests (200 slots) and one for read requests (400 slots). A single misbehaving controller could fill all 400 read slots, **starving** `kubectl`, the scheduler, and every other client. The whole cluster would appear unresponsive.

**APF** replaces this with a work-queue system where:
1. Requests are classified by **FlowSchema** (who is making the request, what type?)
2. Each FlowSchema routes to a **PriorityLevel**, which has its own concurrency limit and queue

Key priority levels:
- **`exempt`:** `system:masters` bypass all limits — for emergency "break-glass" access
- **`cluster-admin`:** high priority for admin operations
- **`catch-all`:** everyone else, strict limits

Within each level, **fair queuing** (via "shuffle sharding") gives each flow (a user + resource + verb combination) its own virtual queue — so one bad actor can't monopolize a priority level.

---

### Q4: How does the watch cache prevent the API server from being overwhelmed?

**Answer:**

**Without caching:** Each of 100 controllers doing List+Watch on pods would hit etcd directly — 100 concurrent etcd streams, with etcd fanning out every event to 100 clients. etcd is not designed for high fan-out.

**With the watch cache:**
1. The API server maintains a **single watch** against etcd per resource type
2. When an event arrives, it goes into a **ring buffer** (the WatchCache, default 100 events per resource type)
3. All controller watches connect to the API server's WatchCache, **not** etcd directly
4. The API server fans out one etcd event to 100 clients from its own process memory
5. `kubectl get pods` is served from the WatchCache — **zero etcd reads**
6. Only cache misses (resourceVersion too old) go to etcd

**Result:** This reduces etcd read pressure by **10–100x** in busy clusters.

---

### Q5: A controller keeps receiving the same reconcile event repeatedly even though nothing has changed. What's likely happening and how do you fix it?

**Answer:**

This is a **reconcile loop** — the controller reconciles, makes a change, that change triggers another watch event, which triggers another reconcile, forever.

**Common causes:**
1. The controller updates the object's `status` or `metadata` on *every* reconcile, triggering a `MODIFIED` event even when nothing meaningful changed
2. The controller uses `Update` instead of `Patch`, overwriting the full object (including fields managed by other controllers), causing conflicts and re-reconciliations

**Fixes:**
1. **Before updating, compare desired vs. actual state** — only write if there's a real diff
2. **Use Patch with Server-Side Apply (SSA)** instead of `Update`
3. For status updates, use `equality.Semantic.DeepEqual` to skip the update if nothing changed
4. **Add a Generation check** — only reconcile when `metadata.generation` changes (which happens on spec changes), not on every `metadata.resourceVersion` change (which happens on status changes and doesn't increment generation)

---

### Q6: What happens to running pods during an API server downtime of 5 minutes?

**Answer:**

**What keeps working:**
- **All currently running containers** — the kubelet doesn't need the API server to maintain running containers. containerd runs independently of Kubernetes.
- **kubelet keeps running its last known set of pods** — it caches the pod list locally
- **Services** — kube-proxy uses its own cache and existing iptables rules, so traffic keeps flowing
- **CoreDNS** — it's already running as a pod, so name resolution continues
- **Load balancer rules** — existing rules stay in place

**What stops working:**
- **New pod scheduling** — the scheduler can't write assignments without the API server
- **Scaling** — HPA and manual scale operations can't proceed
- **Config updates** — new ConfigMaps/Secrets don't propagate to pods
- **Pod restarts that require rescheduling** — kubelet can restart containers locally, but if a pod dies and needs to be rescheduled, it can't get a new assignment
- **`kubectl` commands** — all CLI operations fail
- **All control plane operations** — no new deployments, no rolling updates, no autoscaling

**The takeaway:** Stateless applications tolerate brief API server outages gracefully. But stateful operations that require the Kubernetes API (like StatefulSet ordered startup) are blocked. This is why running multiple API server replicas is important for production availability.

---

## 📚 Glossary of Key Terms

| Term | Plain-English Meaning |
|---|---|
| **API Server** | The single entry point for all Kubernetes operations — every request goes through it. |
| **etcd** | The database that stores the cluster's desired state and actual state. |
| **kubelet** | An agent on every node that ensures containers are running as expected. |
| **kube-proxy** | A network component on every node that implements Service routing rules. |
| **Scheduler** | Decides which node a pod should run on, based on resources and constraints. |
| **Controller Manager** | A collection of controllers that watch the cluster and fix drift between desired and actual state. |
| **RBAC** | Role-Based Access Control — defines who can do what. |
| **Admission Webhook** | Custom logic that can modify (mutating) or reject (validating) requests before they're saved. |
| **resourceVersion** | A version number on every object, used for optimistic concurrency. |
| **Watch** | A streaming subscription that delivers change events in real time. |
| **WatchCache** | The API server's in-memory cache of recent events, reducing load on etcd. |
| **Optimistic Concurrency** | A strategy where writes include a version check, and conflicts trigger retries. |
| **APF** | API Priority and Fairness — prevents any single client from starving others. |
| **FlowSchema** | Classifies API requests into priority levels (part of APF). |
| **Impersonation** | Making a request as if you were another user — useful for RBAC debugging. |
| **iptables** | Linux firewall rules used by kube-proxy to route Service traffic. |
| **IPVS** | A faster kernel-level load balancer that kube-proxy can use instead of iptables. |
| **eBPF** | A technology for running programs in the Linux kernel — the basis for tools like Cilium. |
| **JWT** | JSON Web Token — the format of service account tokens. |
| **Projected Token** | A service account token that's automatically rotated by the kubelet. |
| **Leader Election** | A mechanism ensuring only one instance of a controller is active at a time. |
| **Compaction** | Removing old revisions from etcd to reclaim space. |
| **Defragmentation** | Returning freed disk space to the OS after etcd compaction. |
| **SubjectAccessReview** | An API object that asks "is this user allowed to do X?" |
| **TokenReview** | An API object that asks "is this token valid, and who does it belong to?" |

---

## 🙏 Final Notes

This guide covers the internals that most Kubernetes tutorials skip. You don't need to memorize every detail — but understanding *that these mechanisms exist* and *roughly how they work* will make you a far more effective Kubernetes engineer.

**The best way to learn:** Run the commands yourself. Spin up a cluster (Kubeadm, kind, minikube — whatever you prefer), and try each "Try it yourself" block. Seeing the real output makes these concepts click far better than reading about them.

If you found this helpful, consider starring the repository and sharing it with others learning Kubernetes.

---

*This guide is based on a "Kubernetes Internals" deep-dive curriculum, rewritten with beginner-friendly explanations, analogies, and hands-on examples.*

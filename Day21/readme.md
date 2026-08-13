# Day 21 — Multi-Tenancy, Admission Webhooks & API Machinery

Today covers the platform engineering layer — the infrastructure that teams build on top of Kubernetes to serve multiple teams safely. This is what Staff Engineer and Platform Engineer interviews at top companies test almost exclusively.

## 🧠 Part 1: Multi-Tenancy Threat Model

Before building anything, define what you're protecting against:

```
Noisy neighbour    — team A's workload starves team B of CPU/memory
Blast radius       — team A's bug cascades to team B's namespace
Data leakage       — team A's pod reads team B's secrets
Privilege escape   — team A gains cluster-admin via misconfigured RBAC
Supply chain       — team A deploys a malicious image that affects shared nodes
```

**Two tenancy models:**

```
Soft multi-tenancy   — namespaces + RBAC + quota
                       trusted internal teams
                       single cluster
                       cost-efficient

Hard multi-tenancy   — separate clusters per tenant
                       untrusted external users (SaaS)
                       kernel-level isolation (Kata Containers)
                       expensive but safe
```

*Production reality: most companies use soft multi-tenancy internally and hard multi-tenancy for customer-facing SaaS.*

## 🏗️ Part 2: Hierarchical Namespaces

Flat namespaces don't model team structures. A team has sub-teams, projects have environments, environments have services. Hierarchical Namespace Controller (HNC) makes namespaces tree-structured.

```
platform-team/
├── platform-dev/
│   ├── platform-dev-frontend/
│   └── platform-dev-backend/
└── platform-prod/
    ├── platform-prod-frontend/
    └── platform-prod-backend/
```

```
# Install HNC
kubectl apply -f \
  https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/latest/download/default.yaml

# Install HNC kubectl plugin
HNC_VERSION=v1.1.0
curl -L https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/${HNC_VERSION}/kubectl-hns_linux_amd64 \
  -o /usr/local/bin/kubectl-hns
chmod +x /usr/local/bin/kubectl-hns

# Create namespace hierarchy
kubectl create namespace platform-team
kubectl hns create platform-dev --parent platform-team
kubectl hns create platform-prod --parent platform-team
kubectl hns create platform-dev-frontend --parent platform-dev

# See the tree
kubectl hns tree platform-team
# platform-team
# ├── platform-dev
# │   └── platform-dev-frontend
# └── platform-prod

# Key feature: propagate RBAC and NetworkPolicy down the tree
# A RoleBinding in platform-team propagates to all children
kubectl create rolebinding team-lead \
  --clusterrole=admin \
  --user=alice \
  -n platform-team

# Alice now has admin in ALL child namespaces automatically
kubectl auth can-i create pods \
  --as=alice \
  -n platform-dev-frontend    # yes — propagated from parent

# Propagate a NetworkPolicy to all children
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: platform-team
  annotations:
    hnc.x-k8s.io/propagate: "true"    # propagate to all children
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

# Verify it propagated
kubectl get networkpolicy -n platform-dev-frontend
# NAME           POD-SELECTOR   AGE
# default-deny   <none>         5s  ← propagated from parent
```

## 🎛️ Part 3: Admission Webhooks — Deep Internals

Day 16 used Gatekeeper (a ValidatingAdmissionWebhook). Today you understand how webhooks work and build one.

**The admission chain**

```
kubectl apply →
  API Server →
    Authentication →
      Authorization (RBAC) →
        Mutating Admission Webhooks   ← run first, can modify
          (multiple, run in parallel, then serialized) →
        Object Schema Validation →
        Validating Admission Webhooks  ← run second, can only approve/reject
          (run in parallel) →
        Persisted to etcd
```

**Critical order:** 

Mutating webhooks run before validating. A mutating webhook can add a label, then a validating webhook checks for that label. This is the injection pattern — Istio's sidecar injector is a mutating webhook.

**Webhook configuration**

```
# MutatingWebhookConfiguration
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: label-injector
webhooks:
- name: inject-labels.example.com
  admissionReviewVersions: [v1, v1beta1]
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-system
      path: /mutate                  # your webhook endpoint
    caBundle: <base64-ca-cert>       # TLS required — webhooks must be HTTPS
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    operations: [CREATE, UPDATE]
  namespaceSelector:                 # only for specific namespaces
    matchLabels:
      webhook: enabled
  objectSelector:                    # only for specific objects
    matchExpressions:
    - key: skip-webhook
      operator: DoesNotExist
  failurePolicy: Fail                # Fail | Ignore
  # Fail: webhook unavailable → request rejected (safe but fragile)
  # Ignore: webhook unavailable → request proceeds (available but unsafe)
  sideEffects: None                  # None | NoneOnDryRun | Some | Unknown
  timeoutSeconds: 5                  # webhook must respond within 5s
  reinvocationPolicy: IfNeeded       # re-run if another webhook modified object
```

**Build a webhook in Python**

```
# webhook.py — a real admission webhook
from flask import Flask, request, jsonify
import base64
import json
import jsonpatch

app = Flask(__name__)

@app.route('/mutate', methods=['POST'])
def mutate():
    admission_review = request.get_json()
    request_obj = admission_review['request']
    
    uid = request_obj['uid']
    obj = request_obj['object']
    
    # Build patches to apply
    patches = []
    
    # Patch 1: Add team label if missing
    if 'team' not in obj.get('metadata', {}).get('labels', {}):
        if 'labels' not in obj['metadata']:
            patches.append({
                "op": "add",
                "path": "/metadata/labels",
                "value": {}
            })
        patches.append({
            "op": "add",
            "path": "/metadata/labels/team",
            "value": "platform"         # default team
        })
    
    # Patch 2: Add cost-center annotation
    namespace = request_obj.get('namespace', 'default')
    cost_center = get_cost_center(namespace)
    patches.append({
        "op": "add",
        "path": "/metadata/annotations/cost-center",
        "value": cost_center
    })
    
    # Patch 3: Inject default resource limits if missing
    containers = obj.get('spec', {}).get(
        'template', {}).get('spec', {}).get('containers', [])
    
    for i, container in enumerate(containers):
        if 'resources' not in container:
            patches.append({
                "op": "add",
                "path": f"/spec/template/spec/containers/{i}/resources",
                "value": {
                    "requests": {"cpu": "100m", "memory": "128Mi"},
                    "limits":   {"cpu": "500m", "memory": "256Mi"}
                }
            })
    
    # Encode patches as base64 JSON Patch
    patch_bytes = json.dumps(patches).encode()
    patch_b64 = base64.b64encode(patch_bytes).decode()
    
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": uid,
            "allowed": True,
            "patchType": "JSONPatch",
            "patch": patch_b64
        }
    })

@app.route('/validate', methods=['POST'])
def validate():
    admission_review = request.get_json()
    request_obj = admission_review['request']
    uid = request_obj['uid']
    obj = request_obj['object']
    
    errors = []
    
    # Rule 1: Must have team label
    labels = obj.get('metadata', {}).get('labels', {})
    if 'team' not in labels:
        errors.append("missing required label: team")
    
    # Rule 2: Must not use :latest tag
    containers = obj.get('spec', {}).get(
        'template', {}).get('spec', {}).get('containers', [])
    for container in containers:
        image = container.get('image', '')
        if image.endswith(':latest') or ':' not in image:
            errors.append(
                f"container '{container['name']}' "
                f"must not use :latest or untagged image"
            )
    
    # Rule 3: Must have resource limits
    for container in containers:
        resources = container.get('resources', {})
        if not resources.get('limits', {}).get('memory'):
            errors.append(
                f"container '{container['name']}' "
                f"missing memory limit"
            )
    
    if errors:
        return jsonify({
            "apiVersion": "admission.k8s.io/v1",
            "kind": "AdmissionReview",
            "response": {
                "uid": uid,
                "allowed": False,
                "status": {
                    "code": 403,
                    "message": "; ".join(errors)
                }
            }
        })
    
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": uid,
            "allowed": True
        }
    })

def get_cost_center(namespace):
    mapping = {
        "platform-prod": "CC-001",
        "platform-dev": "CC-002",
        "data-team": "CC-003"
    }
    return mapping.get(namespace, "CC-000")

if __name__ == '__main__':
    app.run(
        host='0.0.0.0',
        port=8443,
        ssl_context=('tls.crt', 'tls.key')    # TLS required
    )
```

**Deploy the webhook**

```
# Step 1: Generate self-signed cert for the webhook service
openssl req -x509 -newkey rsa:4096 \
  -keyout tls.key -out tls.crt \
  -days 365 -nodes \
  -subj "/CN=webhook-service.webhook-system.svc" \
  -addext "subjectAltName=DNS:webhook-service.webhook-system.svc,DNS:webhook-service.webhook-system.svc.cluster.local"

# Step 2: Create namespace and TLS secret
kubectl create namespace webhook-system
kubectl create secret tls webhook-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n webhook-system

# Step 3: Deploy webhook server
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
  namespace: webhook-system
spec:
  replicas: 2                      # HA — webhook must be highly available
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      containers:
      - name: webhook
        image: ghcr.io/youruser/admission-webhook:v1
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: tls
          mountPath: /certs
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: webhook-tls
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  namespace: webhook-system
spec:
  selector:
    app: admission-webhook
  ports:
  - port: 443
    targetPort: 8443
EOF

# Step 4: Register the webhook with K8s
CA_BUNDLE=$(cat tls.crt | base64 | tr -d '\n')

cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: platform-mutator
webhooks:
- name: mutate.platform.example.com
  admissionReviewVersions: [v1]
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-system
      path: /mutate
    caBundle: ${CA_BUNDLE}
  rules:
  - apiGroups: [apps]
    apiVersions: [v1]
    resources: [deployments]
    operations: [CREATE, UPDATE]
  failurePolicy: Ignore    # don't block if webhook is down
  sideEffects: None
  timeoutSeconds: 5
EOF

# Test it
kubectl create deployment test-webhook \
  --image=nginx:1.25 \
  -n platform-dev

# Check if labels were injected
kubectl get deployment test-webhook \
  -n platform-dev \
  -o jsonpath='{.metadata.labels}' | jq .
# {"team": "platform", "app": "test-webhook"}  ← injected
```

## 🔧 Part 4: Kubernetes API Machinery Deep Dive

Understanding the API machinery is what makes you dangerous in interviews.

**API groups and versioning**

```
# See all API groups
kubectl api-versions
# admissionregistration.k8s.io/v1
# apiextensions.k8s.io/v1
# apps/v1
# autoscaling/v1
# autoscaling/v2
# batch/v1
# ...

# See all resources and their API group
kubectl api-resources
# NAME                    SHORTNAMES   APIVERSION                    NAMESPACED   KIND
# pods                    po           v1                            true         Pod
# deployments             deploy       apps/v1                       true         Deployment
# customresourcedefs      crd,crds     apiextensions.k8s.io/v1       false        CustomResourceDefinition

# See what verbs are available per resource
kubectl api-resources --verbs=list -o wide | head -20
```

**Server-side apply — the future of kubectl apply**

```
# Traditional apply: client computes diff, sends full object
# Server-side apply: server computes diff, tracks field ownership

# Apply with SSA
kubectl apply \
  --server-side \
  --field-manager=my-controller \
  -f deployment.yaml

# Why it matters:
# Multiple controllers can manage different fields of the same object
# Field ownership tracked by fieldManager name
# Conflicts detected when two managers claim the same field
# Kubectl apply uses "kubectl" as field manager
# Your operator uses its own name

# See field ownership
kubectl get deployment my-app \
  -o yaml | grep -A 100 managedFields
# managedFields:
# - manager: kubectl         → owns metadata.labels
# - manager: my-controller   → owns spec.replicas
# - manager: hpa-controller  → owns spec.replicas (CONFLICT → rejected)
```

**Strategic Merge Patch vs JSON Patch vs JSON Merge Patch**

```
# Three patch types — know the difference

# 1. Strategic Merge Patch (kubectl apply default)
# Understands K8s list semantics — merges containers by name
kubectl patch deployment my-app \
  --type=strategic \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","image":"nginx:1.26"}]}}}}'
# Merges the container named "app" — doesn't replace the whole list

# 2. JSON Merge Patch
# Replaces entire arrays — careful with lists
kubectl patch deployment my-app \
  --type=merge \
  -p '{"spec":{"replicas":5}}'
# Safe for scalars. Dangerous for arrays.

# 3. JSON Patch (RFC 6902) — surgical precision
kubectl patch deployment my-app \
  --type=json \
  -p '[
    {"op":"replace","path":"/spec/replicas","value":5},
    {"op":"add","path":"/metadata/labels/version","value":"v2"},
    {"op":"remove","path":"/metadata/labels/deprecated"}
  ]'
# Explicit operations: add, remove, replace, move, copy, test
```

**Watch mechanism — how controllers get events**

```
# Manually watch resources (what controllers do programmatically)
kubectl get pods --watch -v=8 2>&1 | grep -E "GET|WATCH|resourceVersion"

# You see:
# GET /api/v1/namespaces/default/pods → returns list + resourceVersion: 12345
# GET /api/v1/namespaces/default/pods?watch=true&resourceVersion=12345
#   → long-lived connection
#   → server pushes events: ADDED, MODIFIED, DELETED

# ResourceVersion is the etcd revision — tells K8s where to resume watching
# If watch is interrupted: reconnect from last known resourceVersion
# If resourceVersion expired (too old): must relist from scratch

# See events your controller would receive
kubectl get events -w --output-watch-events=true
# EVENT-TYPE   REASON   OBJECT   MESSAGE
# ADDED        ...
# MODIFIED     ...
```

**Rate limiting and client-go patterns**

```
// Production controller patterns

// 1. Work queue with rate limiting — prevents thundering herd
workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
// DefaultControllerRateLimiter uses:
// - Exponential backoff per item (5ms → 1000s)
// - Bucket rate limiter (10 qps, 100 burst) for total throughput

// 2. Informer — cached watch with event handlers
informer := factory.Apps().V1().Deployments().Informer()
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { enqueue(obj) },
    UpdateFunc: func(old, new interface{}) { enqueue(new) },
    DeleteFunc: func(obj interface{}) { enqueue(obj) },
})

// 3. Lister — reads from informer cache (not API server)
lister := factory.Apps().V1().Deployments().Lister()
deployment, err := lister.Deployments(namespace).Get(name)
// This is a local cache read — zero API server traffic

// 4. Retry with exponential backoff
if err := retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    // Get latest version
    current, err := client.AppsV1().Deployments(ns).Get(ctx, name, metav1.GetOptions{})
    if err != nil {
        return err
    }
    // Modify
    current.Spec.Replicas = &desiredReplicas
    // Update — may fail with conflict if another controller updated first
    _, err = client.AppsV1().Deployments(ns).Update(ctx, current, metav1.UpdateOptions{})
    return err
}); err != nil {
    return err
}
```

## 🏛️ Part 5: Namespace-as-a-Service Platform

The full pattern production platform teams implement.

**Namespace provisioning via operator**

```
# Team requests a namespace via a CRD
apiVersion: platform.company.com/v1
kind: NamespaceRequest
metadata:
  name: data-science-prod
spec:
  team: data-science
  environment: production
  costCenter: CC-042
  quotaTier: large             # small | medium | large | xlarge
  allowedRegistries:
  - ghcr.io/yourcompany
  - registry.k8s.io
  networkAccess:
  - database-team              # can reach database-team services
  owners:
  - alice@company.com
  - bob@company.com
```

**The Namespace Operator watches NamespaceRequest and creates:**

```
# 1. Namespace with labels
apiVersion: v1
kind: Namespace
metadata:
  name: data-science-prod
  labels:
    team: data-science
    environment: production
    cost-center: CC-042
    pod-security.kubernetes.io/enforce: restricted

# 2. ResourceQuota based on tier
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: data-science-prod
spec:
  hard:
    requests.cpu: "20"          # large tier
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"

# 3. LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: data-science-prod
spec:
  limits:
  - type: Container
    default: {cpu: 500m, memory: 512Mi}
    defaultRequest: {cpu: 100m, memory: 128Mi}
    max: {cpu: "8", memory: 16Gi}

# 4. NetworkPolicy — default deny + allowed peers
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: data-science-prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-team
  namespace: data-science-prod
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          team: database-team
    ports:
    - port: 5432

# 5. RBAC — owners get admin, members get developer role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-owners
  namespace: data-science-prod
subjects:
- kind: User
  name: alice@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-admin           # custom ClusterRole — full ns access
  apiGroup: rbac.authorization.k8s.io

# 6. ServiceAccount for CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: data-science-prod
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/data-science-cicd"
```

## 🔍 Part 6: Kyverno — Alternative to Gatekeeper

Kyverno is a Kubernetes-native policy engine — policies are written in YAML, not Rego. Easier to adopt for teams unfamiliar with OPA.

```
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace

kubectl get pods -n kyverno -w
```

**Kyverno policies — YAML not Rego**

```
# Policy 1: Mutate — add default labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-team-label
    match:
      any:
      - resources:
          kinds: [Deployment, StatefulSet, DaemonSet]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(team): "unknown"        # + means "add only if missing"
            +(environment): "dev"

---
# Policy 2: Validate — require labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce   # Enforce | Audit
  rules:
  - name: require-team-label
    match:
      any:
      - resources:
          kinds: [Deployment]
          namespaces: [production, staging]
    validate:
      message: "Deployment must have 'team' and 'environment' labels"
      pattern:
        metadata:
          labels:
            team: "?*"               # any non-empty value
            environment: "?*"

---
# Policy 3: Generate — auto-create resources
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-netpol
spec:
  rules:
  - name: default-deny
    match:
      any:
      - resources:
          kinds: [Namespace]
          selector:
            matchLabels:
              generate-netpol: "true"
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny
      namespace: "{{request.object.metadata.name}}"
      synchronize: true              # keep in sync — if deleted, recreate
      data:
        spec:
          podSelector: {}
          policyTypes: [Ingress, Egress]

---
# Policy 4: Verify images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-cosign-signature
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production]
    verifyImages:
    - imageReferences:
      - "ghcr.io/yourcompany/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <your cosign public key>
              -----END PUBLIC KEY-----
```

**Kyverno vs Gatekeeper — choose wisely**

<img width="921" height="403" alt="image" src="https://github.com/user-attachments/assets/91ded25c-f1b1-4e40-9802-fd2dec1ff886" />

## 🖥️ Part 7: Hands-On Exercises

**Exercise 1: Build and deploy your admission webhook**

```
# Create a simple webhook in Python that:
# 1. Adds "managed-by: platform" label to all Deployments
# 2. Rejects Deployments with :latest image tag

# Write the webhook
cat <<'EOF' > webhook.py
from flask import Flask, request, jsonify
import base64
import json

app = Flask(__name__)

@app.route('/mutate', methods=['POST'])
def mutate():
    review = request.get_json()
    uid = review['request']['uid']
    obj = review['request']['object']
    
    patches = []
    
    # Add managed-by label
    labels = obj.get('metadata', {}).get('labels', {})
    if 'managed-by' not in labels:
        if not obj['metadata'].get('labels'):
            patches.append({"op":"add","path":"/metadata/labels","value":{}})
        patches.append({"op":"add","path":"/metadata/labels/managed-by","value":"platform"})
    
    patch_b64 = base64.b64encode(json.dumps(patches).encode()).decode()
    
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": uid,
            "allowed": True,
            "patchType": "JSONPatch",
            "patch": patch_b64
        }
    })

@app.route('/validate', methods=['POST'])
def validate():
    review = request.get_json()
    uid = review['request']['uid']
    obj = review['request']['object']
    
    containers = obj.get('spec',{}).get('template',{}).get('spec',{}).get('containers',[])
    errors = []
    
    for c in containers:
        img = c.get('image','')
        if img.endswith(':latest') or ':' not in img:
            errors.append(f"container '{c['name']}' uses :latest or untagged image")
    
    if errors:
        return jsonify({
            "apiVersion":"admission.k8s.io/v1",
            "kind":"AdmissionReview",
            "response":{
                "uid":uid,
                "allowed":False,
                "status":{"code":403,"message":" | ".join(errors)}
            }
        })
    
    return jsonify({
        "apiVersion":"admission.k8s.io/v1",
        "kind":"AdmissionReview",
        "response":{"uid":uid,"allowed":True}
    })

@app.route('/healthz')
def health():
    return 'ok'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF

# For local testing — use a NodePort service + socat to avoid TLS complexity
# In production: use cert-manager to get TLS automatically

# Test your webhook logic locally
# Install pip and required Python packages
sudo apt update && sudo apt install -y python3-pip python3-flask
pip3 install flask jsonpatch --break-system-packages 2>/dev/null || pip3 install flask jsonpatch

# Start the Webhook Server in the Background
python3 webhook.py &

# Confirm the process is running on port 8080:
curl http://localhost:8080/healthz

# Test Mutate and Validate Endpoints
# Test /mutate (Injects managed-by: platform label):
curl -s -X POST http://localhost:8080/mutate \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "uid": "test-123",
      "object": {
        "metadata": {"name": "test", "labels": {}},
        "spec": {"template": {"spec": {"containers": [{"name":"app","image":"nginx:1.25"}]}}}
      }
    }
  }' | jq .

# Expected Output:
127.0.0.1 - - [13/Aug/2026 05:58:08] "POST /mutate HTTP/1.1" 200 -
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "allowed": true,
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL21ldGFkYXRhL2xhYmVscyIsICJ2YWx1ZSI6IHt9fSwgeyJvcCI6ICJhZGQiLCAicGF0aCI6ICIvbWV0YWRhdGEvbGFiZWxzL21hbmFnZWQtYnkiLCAidmFsdWUiOiAicGxhdGZvcm0ifV0=",
    "patchType": "JSONPatch",
    "uid": "test-123"
  }
}

# Test /validate (Blocks :latest tag):
curl -s -X POST http://localhost:8080/validate \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "uid": "test-456",
      "object": {
        "metadata": {"name":"bad"},
        "spec": {"template": {"spec": {"containers": [{"name":"app","image":"nginx:latest"}]}}}
      }
    }
  }' | jq .

# Expected Validation Response:
127.0.0.1 - - [13/Aug/2026 05:58:24] "POST /validate HTTP/1.1" 200 -
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "allowed": false,
    "status": {
      "code": 403,
      "message": "container 'app' uses :latest or untagged image"
    },
    "uid": "test-456"
  }
}
```

**Exercise 2: Kyverno policy enforcement**

```
# Add the official Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/

# Update your local Helm chart repository cache
helm repo update

# Install Kyverno into the kyverno namespace
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace

kubectl get pods -n kyverno -w

# Apply require-labels policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: check-team-label
    match:
      any:
      - resources:
          kinds: [Deployment]
    validate:
      message: "Deployment must have a 'team' label"
      pattern:
        metadata:
          labels:
            team: "?*"
EOF

# Test enforcement
kubectl create deployment no-label --image=nginx:1.25
# Error: admission webhook denied: Deployment must have a 'team' label

kubectl create deployment with-label \
  --image=nginx:1.25 \
  --dry-run=client -o yaml \
  | kubectl label -f - --local team=platform -o yaml \
  | kubectl apply -f -
# Succeeds

# Apply mutation policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-managed-by
spec:
  rules:
  - name: add-label
    match:
      any:
      - resources:
          kinds: [Deployment]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(managed-by): "platform"
EOF

# Prove mutation works
kubectl create deployment auto-label --image=nginx:1.25 \
  --dry-run=server -o yaml | grep managed-by
# managed-by: platform  ← injected by Kyverno
```

**Exercise 3: Namespace-as-a-Service simulation**

```
# Create a namespace provisioning script that implements the platform pattern
cat <<'SCRIPT' > provision-namespace.sh
#!/bin/bash
# Usage: ./provision-namespace.sh <namespace> <team> <tier>
NS=$1 TEAM=$2 TIER=$3

# Tier quotas
case $TIER in
  small)  CPU=4; MEM=8Gi; PODS=20 ;;
  medium) CPU=8; MEM=16Gi; PODS=50 ;;
  large)  CPU=20; MEM=40Gi; PODS=100 ;;
esac

echo "Provisioning namespace: $NS for team: $TEAM (tier: $TIER)"

# 1. Create namespace with PSA labels
kubectl create namespace $NS
kubectl label namespace $NS \
  team=$TEAM \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# 2. ResourceQuota
kubectl create resourcequota quota \
  --hard="requests.cpu=${CPU},requests.memory=${MEM},pods=${PODS}" \
  -n $NS

# 3. Default NetworkPolicy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: $NS
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: $NS
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - port: 53
      protocol: UDP
EOF

# 4. RBAC
kubectl create role developer \
  --verb=get,list,watch,create,update,patch \
  --resource=pods,deployments,services,configmaps \
  -n $NS

kubectl create clusterrolebinding ${NS}-team-admin \
  --clusterrole=admin \
  --group=${TEAM}-admins

echo "Namespace $NS provisioned successfully"
SCRIPT

chmod +x provision-namespace.sh
./provision-namespace.sh data-science-dev data-science medium

# Verify everything was created
kubectl get all,quota,networkpolicy,limitrange,rolebinding -n data-science-dev
```

## 🎯 Part 8: Interview Questions — Day 24

**Q1: What is the difference between a MutatingAdmissionWebhook and a ValidatingAdmissionWebhook?**

Mutating runs first and can modify the incoming object by returning JSON Patch operations — used for injection (Istio sidecars, default labels, resource limits). Validating runs after all mutations are complete and can only approve or reject — it cannot modify. Both run in parallel within their phase. A common pattern: a mutating webhook adds default values, then a validating webhook enforces that all required fields are present. The critical implication: your validating webhook sees the post-mutation object, not the original — so it can rely on mutations having already run.

**Q2: failurePolicy: Fail vs failurePolicy: Ignore — how do you choose?**

Fail means if the webhook server is unreachable or returns an error, the admission request is rejected — nothing gets created. This is safe but means a broken webhook breaks your entire cluster's ability to create resources. Ignore means webhook failures are treated as approved — your cluster keeps working but policies aren't enforced during the outage. The right answer depends on the webhook's purpose: security-critical webhooks (Gatekeeper, image signature verification) should use Fail. Operational webhooks that add convenience labels can use Ignore. Always run at least 2 webhook replicas across nodes with a PodDisruptionBudget, and set timeoutSeconds: 5 to fail fast.

**Q3: How does server-side apply (SSA) change how controllers should be written?**

With traditional client-side apply, the client owns the entire object — updating one field means sending the full spec. With SSA, each controller declares ownership of specific fields via fieldManager. When two controllers both try to own the same field, K8s detects a conflict and rejects one. This enables safe multi-controller scenarios: HPA owns spec.replicas, your operator owns spec.template.spec.containers[app].image, kubectl owns metadata.labels. Each can update independently without clobbering the other. Controllers should use Apply with their own fieldManager name instead of Update — this shifts conflict detection to the API server where it can be resolved cleanly.

**Q4: A namespace is stuck in Terminating. Walk through the fix.**

When a namespace enters Terminating, K8s sends deletion to all resources inside it. If any resource has a finalizer whose controller is not running (CRD deleted before its resources, broken operator), the finalizer never clears and the namespace hangs. Fix: identify the stuck resources with kubectl api-resources --namespaced | xargs -I{} kubectl get {} -n <ns> 2>/dev/null. For each stuck resource, patch the finalizer to empty: kubectl patch <resource> <name> -p '{"metadata":{"finalizers":[]}}' --type=merge. If the namespace itself has a finalizer: use kubectl replace --raw /api/v1/namespaces/<ns>/finalize with a version of the namespace spec that has empty finalizers. This is the nuclear option — always check why the finalizer wasn't cleared first.

**Q5: How do you implement cost allocation in a shared Kubernetes cluster?**

Multiple layers: label everything with team, environment, cost-center via a MutatingAdmissionWebhook — enforced so nothing runs without these labels. Use namespace-level ResourceQuota to prevent runaway spending. Instrument Prometheus with kube_pod_labels metric — join it with container_cpu_usage_seconds_total and container_memory_working_set_bytes to compute cost per team per namespace per day. Use Kubecost or OpenCost (CNCF project) for automated cost reporting — they understand spot instances, on-demand pricing, and PV costs. Export daily cost reports per cost-center label to your finance system. Production nuance: shared infrastructure (ingress controller, monitoring stack, DNS) needs to be either excluded from allocation or split proportionally by namespace count or resource usage.

**Q6: What is the Kubernetes garbage collector and how does owner references work?**

Every K8s object can have ownerReferences — a list of objects that own it. When an owner is deleted, the garbage collector automatically deletes all objects that reference it. Cascading delete modes: Foreground (owner stays until all dependents are deleted — you can watch the cleanup), Background (owner deleted immediately, dependents deleted asynchronously — default), Orphan (dependents are not deleted, ownerReferences cleared). Controllers use controllerutil.SetControllerReference() to set ownership. Example: Deployment → owns ReplicaSet → owns Pods. Delete Deployment with default (Background): ReplicaSet and Pods are deleted asynchronously. This is why deleting a Deployment removes all its pods — owner references, not label selectors.


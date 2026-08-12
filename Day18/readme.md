# Day 18 — Custom Controllers, Operators & Multi-Cluster

This is where you go from "knows Kubernetes" to "understands Kubernetes deeply enough to extend it." Every senior K8s interview at top companies touches this material. Operators are how production teams manage complex stateful systems at scale.

## 🧠 Part 1: The Controller Pattern — Revisited Deeply

Day 3 introduced the reconciliation loop. Today you understand it at the implementation level.
Every controller in K8s — including yours — does exactly this:

```
for {
    desired  := getFromAPI()      // what Git/user wants
    actual   := getFromCluster()  // what actually exists
    diff     := compare(desired, actual)
    if diff != nil {
        reconcile(diff)           // make actual match desired
    }
    sleep(or wait for watch event)
}
```

This is called the operator pattern. The controller watches a resource (built-in or custom) and reconciles the cluster toward the desired state defined in that resource.

**The Kubernetes extension points**

```
Custom Resource Definitions (CRDs)    — extend the K8s API with your own types
Custom Controllers                    — watch CRDs and reconcile
Admission Webhooks                    — mutate or validate resources at admission
API Aggregation                       — add entirely new API servers
```

Operators = CRDs + Custom Controllers working together.

## 📐 Part 2: Custom Resource Definitions

A CRD lets you define your own Kubernetes object. Once created, kubectl get, kubectl apply, kubectl describe all work on it — it feels native.

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: urlshorteners.apps.yourcompany.com   # must be plural.group
spec:
  group: apps.yourcompany.com
  versions:
  - name: v1
    served: true                    # this version is served by the API
    storage: true                   # this version is stored in etcd
    schema:
      openAPIV3Schema:              # validation schema
        type: object
        properties:
          spec:
            type: object
            required: [targetUrl, slug]
            properties:
              targetUrl:
                type: string
                format: uri
                description: "The URL to redirect to"
              slug:
                type: string
                pattern: '^[a-z0-9-]+$'
                maxLength: 50
                description: "Short URL slug"
              ttlDays:
                type: integer
                minimum: 1
                maximum: 365
                default: 30
          status:
            type: object
            properties:
              shortUrl:
                type: string
              hitCount:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: {type: string}
                    status: {type: string}
                    message: {type: string}
    subresources:
      status: {}                    # enables /status subresource
      scale:                        # enables kubectl scale
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.replicas
    additionalPrinterColumns:       # what kubectl get shows
    - name: Target URL
      type: string
      jsonPath: .spec.targetUrl
    - name: Slug
      type: string
      jsonPath: .spec.slug
    - name: Hits
      type: integer
      jsonPath: .status.hitCount
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  scope: Namespaced                 # Namespaced or Cluster
  names:
    plural: urlshorteners
    singular: urlshortener
    kind: URLShortener
    shortNames: [us, urls]          # kubectl get us works
```

```
# Apply the CRD
kubectl apply -f urlshortener-crd.yaml

# Now you can use it like any native resource
kubectl get urlshorteners
kubectl explain urlshortener.spec
kubectl explain urlshortener.spec.targetUrl

# Create a custom resource
cat <<EOF | kubectl apply -f -
apiVersion: apps.yourcompany.com/v1
kind: URLShortener
metadata:
  name: github-short
  namespace: production
spec:
  targetUrl: "https://github.com/youruser/url-shortener"
  slug: "gh"
  ttlDays: 90
EOF

kubectl get urlshorteners
# NAME           TARGET URL                                    SLUG   HITS   AGE
# github-short   https://github.com/youruser/url-shortener   gh     0      5s

# Status subresource — updated by controller
kubectl get urlshortener github-short -o yaml | grep -A 5 status
```

**CRD versioning and conversion**

```
# Multiple versions with conversion webhook
spec:
  versions:
  - name: v1
    served: true
    storage: true        # only one version can be storage version
  - name: v1beta1
    served: true         # still served for backward compat
    storage: false
  conversion:
    strategy: Webhook    # webhook converts between versions
    webhook:
      conversionReviewVersions: [v1, v1beta1]
      clientConfig:
        service:
          name: urlshortener-conversion
          namespace: default
          path: /convert
```

## ⚙️ Part 3: Building a Controller with controller-runtime

*controller-runtime is the library behind most production operators (used by kubebuilder and operator-sdk).*

```
// main.go — operator entry point
package main

import (
    "context"
    "fmt"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    apiv1 "github.com/youruser/url-shortener-operator/api/v1"
)

// URLShortenerReconciler is your controller
type URLShortenerReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// Reconcile is called when a URLShortener object changes
// OR when a resource it owns changes
func (r *URLShortenerReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,    // contains namespace/name of the changed object
) (ctrl.Result, error) {

    // Step 1: Get the URLShortener resource
    urlShortener := &apiv1.URLShortener{}
    if err := r.Get(ctx, req.NamespacedName, urlShortener); err != nil {
        // Object deleted — nothing to do
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Step 2: Define desired state — what should exist?
    desired := r.buildDeployment(urlShortener)

    // Step 3: Get actual state — what exists now?
    actual := &appsv1.Deployment{}
    err := r.Get(ctx, req.NamespacedName, actual)

    if err != nil {
        // Deployment doesn't exist yet — create it
        if err := r.Create(ctx, desired); err != nil {
            return ctrl.Result{}, fmt.Errorf("creating deployment: %w", err)
        }
    } else {
        // Deployment exists — update if needed
        actual.Spec = desired.Spec
        if err := r.Update(ctx, actual); err != nil {
            return ctrl.Result{}, fmt.Errorf("updating deployment: %w", err)
        }
    }

    // Step 4: Update status
    urlShortener.Status.ShortUrl = fmt.Sprintf(
        "https://short.ly/%s", urlShortener.Spec.Slug)
    if err := r.Status().Update(ctx, urlShortener); err != nil {
        return ctrl.Result{}, err
    }

    // Requeue after 5 minutes to ensure drift correction
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

// SetupWithManager registers the controller
func (r *URLShortenerReconciler) SetupWithManager(
    mgr ctrl.Manager,
) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&apiv1.URLShortener{}).   // watch URLShortener objects
        Owns(&appsv1.Deployment{}).   // also watch Deployments this controller creates
        Owns(&corev1.Service{}).      // and Services
        Complete(r)
}

func main() {
    mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
    })

    (&URLShortenerReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)

    mgr.Start(ctrl.SetupSignalHandler())
}
```

**Key controller-runtime concepts**

```
// Watches — what triggers reconciliation
ctrl.NewControllerManagedBy(mgr).
    For(&apiv1.URLShortener{}).        // primary resource
    Owns(&appsv1.Deployment{}).        // secondary — reconcile owner when this changes
    Watches(                           // arbitrary watches
        &corev1.ConfigMap{},
        handler.EnqueueRequestsFromMapFunc(mapConfigMapToURLShortener),
    ).
    Complete(r)

// Finalizers — run cleanup before object is deleted
const finalizer = "urlshorteners.apps.yourcompany.com/finalizer"

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    obj := &apiv1.URLShortener{}
    r.Get(ctx, req.NamespacedName, obj)

    if obj.DeletionTimestamp.IsZero() {
        // Object not being deleted — add finalizer
        if !contains(obj.Finalizers, finalizer) {
            obj.Finalizers = append(obj.Finalizers, finalizer)
            r.Update(ctx, obj)
        }
    } else {
        // Object being deleted — run cleanup
        if contains(obj.Finalizers, finalizer) {
            r.cleanupExternalResources(obj)   // delete DNS record, revoke token etc

            // Remove finalizer — allows K8s to complete deletion
            obj.Finalizers = remove(obj.Finalizers, finalizer)
            r.Update(ctx, obj)
        }
    }
    // ... rest of reconcile
}
```

## 🏭 Part 4: Operator SDK — Scaffold Fast

In practice you use operator-sdk or kubebuilder instead of writing everything from scratch.

```
# Install operator-sdk
curl -LO https://github.com/operator-framework/operator-sdk/releases/latest/download/operator-sdk_linux_amd64
chmod +x operator-sdk_linux_amd64
sudo mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

# Scaffold a new operator
mkdir url-shortener-operator && cd url-shortener-operator
operator-sdk init \
  --domain yourcompany.com \
  --repo github.com/youruser/url-shortener-operator

# Create API + controller
operator-sdk create api \
  --group apps \
  --version v1 \
  --kind URLShortener \
  --resource \
  --controller

# What got generated:
tree .
# ├── api/v1/
# │   ├── urlshortener_types.go     ← define your spec/status structs here
# │   └── zz_generated.deepcopy.go  ← auto-generated
# ├── controllers/
# │   └── urlshortener_controller.go ← write your reconcile logic here
# ├── config/
# │   ├── crd/                       ← generated CRD YAML
# │   ├── rbac/                      ← RBAC for operator SA
# │   └── manager/                   ← Deployment for operator
# └── main.go

# Generate CRD from Go types
make generate manifests

# Run operator locally (watches cluster via kubeconfig)
make run

# Build and deploy to cluster
make docker-build IMG=ghcr.io/youruser/url-shortener-operator:v1
make docker-push IMG=ghcr.io/youruser/url-shortener-operator:v1
make deploy IMG=ghcr.io/youruser/url-shortener-operator:v1
```

## 🌐 Part 5: Real-World Operators — Study These

Understanding how production operators work gives you instant interview credibility.

**Prometheus Operator**

```
# Instead of managing Prometheus config files manually:
apiVersion: monitoring.coreos.com/v1
kind: Prometheus               # CRD
metadata:
  name: main
spec:
  replicas: 2
  serviceMonitorSelector:      # auto-discovers ServiceMonitors
    matchLabels:
      team: platform
  retention: 15d
  storage:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 50Gi
---
# The operator watches Prometheus CRs and:
# - Creates StatefulSet with correct config
# - Mounts generated prometheus.yml ConfigMap
# - Updates config when ServiceMonitors change
# - Handles rolling restarts on config change
```

**cert-manager**

```
# cert-manager watches Certificate CRs and:
# - Creates CertificateSigningRequests
# - Calls Let's Encrypt or internal CA
# - Stores result in a Secret
# - Renews 30 days before expiry automatically
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: url-shortener-tls
spec:
  secretName: url-shortener-tls-secret
  duration: 2160h       # 90 days
  renewBefore: 720h     # renew 30 days before
  dnsNames:
  - short.yourdomain.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**Strimzi (Kafka Operator)**

```
# Manages Kafka clusters as K8s-native objects
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
spec:
  kafka:
    replicas: 3
    listeners:
    - name: plain
      port: 9092
      type: internal
      tls: false
    storage:
      type: persistent-claim
      size: 100Gi
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
```

## 🌍 Part 6: Multi-Cluster Kubernetes

Single clusters hit limits at scale. Multi-cluster is how large organizations run K8s.

**Why multi-cluster**

```
Isolation        — prod/staging on separate clusters, blast radius bounded
Geography        — cluster per region, data residency compliance
Scale            — single cluster limits (~5000 nodes, ~150k pods)
Team autonomy    — each team owns their cluster
DR/HA            — active-active across clusters
```

**Cluster API — provision clusters as K8s objects**

```
# Cluster API (CAPI) lets you manage clusters like pods
# One management cluster provisions and manages workload clusters

apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-eu-west-1
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster              # AWS provider
    name: production-eu-west-1
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-eu-west-1-cp
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: production-eu-west-1
spec:
  region: eu-west-1
  sshKeyName: cluster-key
  networkSpec:
    vpc:
      cidrBlock: "10.0.0.0/16"
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: production-eu-west-1-cp
spec:
  replicas: 3                    # HA control plane
  version: v1.30.0
  machineTemplate:
    infrastructureRef:
      kind: AWSMachineTemplate
      name: production-eu-west-1-cp
```

**ArgoCD multi-cluster — deploy to many clusters**

```
# Register additional clusters in ArgoCD
argocd cluster add production-eu-west-1 \
  --kubeconfig ~/.kube/production-eu-west-1.yaml

argocd cluster add staging-us-east-1 \
  --kubeconfig ~/.kube/staging-us-east-1.yaml

argocd cluster list
# SERVER                          NAME                  STATUS
# https://eu-west-1.api.k8s       production-eu-west-1  Successful
# https://us-east-1.api.k8s       staging-us-east-1     Successful
# https://kubernetes.default.svc  in-cluster            Successful
```

```
# ApplicationSet — deploy same app to multiple clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: url-shortener-all-clusters
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production      # only production clusters

  - list:                              # or explicit list
      elements:
      - cluster: production-eu-west-1
        url: https://eu-west-1.api.k8s
        values:
          region: eu-west-1
          replicaCount: "5"
      - cluster: production-us-east-1
        url: https://us-east-1.api.k8s
        values:
          region: us-east-1
          replicaCount: "3"

  template:
    metadata:
      name: 'url-shortener-{{cluster}}'
    spec:
      project: production
      source:
        repoURL: https://github.com/youruser/gitops-repo
        targetRevision: HEAD
        path: charts/url-shortener
        helm:
          parameters:
          - name: replicaCount
            value: '{{values.replicaCount}}'
          - name: region
            value: '{{values.region}}'
      destination:
        server: '{{url}}'
        namespace: url-shortener
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Submariner — cross-cluster networking**

```
# Submariner connects pod networks across clusters
# Pods in cluster A can reach pods in cluster B directly

# Install on broker cluster
subctl deploy-broker --kubeconfig broker.yaml

# Join cluster A
subctl join broker-info.subm \
  --kubeconfig cluster-a.yaml \
  --clusterid cluster-a

# Join cluster B
subctl join broker-info.subm \
  --kubeconfig cluster-b.yaml \
  --clusterid cluster-b

# Export a service from cluster A — makes it reachable in cluster B
kubectl apply -f - <<EOF
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: redis
  namespace: production
EOF

# In cluster B — reach cluster A's redis
# redis.production.svc.clusterset.local
```

## 🔀 Part 7: Service Mesh — Istio Fundamentals

Istio intercepts all pod-to-pod traffic via sidecar proxies (Envoy). This gives you: mTLS, traffic shaping, observability, and circuit breaking — without changing app code.

```
Without Istio:  Pod A → Pod B  (plain TCP, no encryption, no metrics)
With Istio:     Pod A → Envoy sidecar → encrypted mTLS → Envoy sidecar → Pod B
                              ↓                                  ↓
                        metrics/traces                    metrics/traces
```

**Install Istio**

```
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -
sudo mv istio-*/bin/istioctl /usr/local/bin/

# Install Istio on cluster
istioctl install --set profile=demo -y

# Enable sidecar injection for a namespace
kubectl label namespace production istio-injection=enabled

# Verify — new pods get 2/2 containers (app + istio-proxy)
kubectl rollout restart deployment -n production
kubectl get pods -n production
# url-shortener-xxx   2/2   Running  ← app + istio-proxy sidecar
```

**Traffic management**

```
# VirtualService — advanced routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: url-shortener
spec:
  hosts:
  - url-shortener
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: url-shortener
        subset: v2              # canary users → v2
  - route:
    - destination:
        host: url-shortener
        subset: v1              # everyone else → v1
      weight: 90
    - destination:
        host: url-shortener
        subset: v2
      weight: 10                # 10% traffic split
---
# DestinationRule — define subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: url-shortener
spec:
  host: url-shortener
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:           # circuit breaker
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**mTLS — automatic mutual TLS**

```
# Enforce strict mTLS for a namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT    # STRICT | PERMISSIVE | DISABLE

# AuthorizationPolicy — who can talk to whom
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: url-shortener-authz
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
        - cluster.local/ns/production/sa/frontend-sa  # only frontend SA allowed
    to:
    - operation:
        methods: [GET, POST]
        paths: ["/shorten", "/r/*"]
```

## 🖥️ Part 8: Hands-On Exercises

**Exercise 1: Build and use your first CRD**

```
# Apply the URLShortener CRD
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: urlshorteners.apps.example.com
spec:
  group: apps.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [targetUrl, slug]
            properties:
              targetUrl:
                type: string
              slug:
                type: string
                maxLength: 20
              ttlDays:
                type: integer
                default: 30
          status:
            type: object
            properties:
              shortUrl:
                type: string
              ready:
                type: boolean
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Target
      type: string
      jsonPath: .spec.targetUrl
    - name: Slug
      type: string
      jsonPath: .spec.slug
    - name: TTL
      type: integer
      jsonPath: .spec.ttlDays
  scope: Namespaced
  names:
    plural: urlshorteners
    singular: urlshortener
    kind: URLShortener
    shortNames: [us]
EOF

# Use it
kubectl get urlshorteners
kubectl explain urlshortener.spec

cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: URLShortener
metadata:
  name: my-first-cr
spec:
  targetUrl: "https://kubernetes.io/docs"
  slug: "k8sdocs"
  ttlDays: 60
EOF

kubectl get us
# NAME           TARGET                        SLUG      TTL
# my-first-cr   https://kubernetes.io/docs   k8sdocs   60

# Prove validation works
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: URLShortener
metadata:
  name: invalid-cr
spec:
  targetUrl: "https://example.com"
  slug: "this-slug-is-way-too-long-exceeds-twenty-chars"
EOF
# Error: spec.slug: Too long: max length is 20

# Update status subresource (controller would do this)
kubectl patch urlshortener my-first-cr \
  --subresource=status \
  --type=merge \
  -p '{"status":{"shortUrl":"https://short.ly/k8sdocs","ready":true}}'

kubectl get us my-first-cr -o yaml | grep -A 5 status
```

**Exercise 2: Write a simple controller in bash (concept)**

```
# A controller in bash — proves the reconcile loop concept
# Real controllers use Go + controller-runtime

cat <<'CONTROLLER' > /tmp/bash-controller.sh
#!/bin/bash
# Watch URLShortener resources and create ConfigMaps for each

echo "Starting URLShortener controller..."

while true; do
  # Get all URLShortener resources
  RESOURCES=$(kubectl get urlshorteners -A -o json 2>/dev/null)

  echo "$RESOURCES" | jq -c '.items[]' | while read -r item; do
    NAME=$(echo "$item" | jq -r '.metadata.name')
    NS=$(echo "$item" | jq -r '.metadata.namespace')
    SLUG=$(echo "$item" | jq -r '.spec.slug')
    TARGET=$(echo "$item" | jq -r '.spec.targetUrl')

    # Desired state: ConfigMap with slug → target mapping
    CM_NAME="urlshortener-${NAME}"

    # Check if ConfigMap exists
    kubectl get configmap "$CM_NAME" -n "$NS" &>/dev/null

    if [ $? -ne 0 ]; then
      echo "Creating ConfigMap for $NAME in $NS"
      kubectl create configmap "$CM_NAME" \
        --from-literal="slug=$SLUG" \
        --from-literal="target=$TARGET" \
        -n "$NS"
    fi

    # Update status
    kubectl patch urlshortener "$NAME" -n "$NS" \
      --subresource=status \
      --type=merge \
      -p "{\"status\":{\"shortUrl\":\"https://short.ly/$SLUG\",\"ready\":true}}" \
      &>/dev/null
  done

  sleep 10    # reconcile every 10 seconds
done
CONTROLLER

chmod +x /tmp/bash-controller.sh

# Run the controller
kubectl create namespace controller-demo
/tmp/bash-controller.sh &
CONTROLLER_PID=$!

# Create a URLShortener — controller should create ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: URLShortener
metadata:
  name: demo-url
  namespace: controller-demo
spec:
  targetUrl: "https://example.com"
  slug: "demo"
EOF

sleep 12

# Verify controller created the ConfigMap
kubectl get configmap -n controller-demo
kubectl get configmap urlshortener-demo-url -n controller-demo -o yaml

# Verify status was updated
kubectl get urlshortener demo-url -n controller-demo -o yaml | grep -A 5 status

# Stop controller
kill $CONTROLLER_PID
```

**Exercise 3: ArgoCD ApplicationSet multi-namespace deploy**

```
# Deploy the same app to multiple namespaces using ApplicationSet
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: url-shortener-envs
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        replicas: "1"
        namespace: url-shortener-dev
      - env: staging
        replicas: "2"
        namespace: url-shortener-staging
  template:
    metadata:
      name: 'url-shortener-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: guestbook
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
EOF

# Watch all apps created at once
argocd app list

# Syncing each environment independently
argocd app sync url-shortener-dev
argocd app sync url-shortener-staging
```

## 🎯 Part 9: Interview Questions — Day 18

**Q1: What is an Operator and how is it different from a Helm chart?**

A Helm chart is a packaging tool — it templates and installs K8s manifests but has no ongoing awareness of the cluster. Once installed, Helm is done. An Operator is a running controller that continuously watches custom resources and reconciles cluster state toward what those resources describe. It encodes operational knowledge — how to upgrade, backup, heal, and scale a specific application. Helm installs software; an Operator runs it. For simple stateless apps, Helm is fine. For stateful systems like databases or message queues, Operators encode the operational playbook that would otherwise require human expertise.

**Q2: Walk me through the lifecycle of a custom resource when its controller is down.**

The resource itself (stored in etcd) is unaffected — it exists and can be read/updated via kubectl normally. But nothing reconciles against it. If you create a new URLShortener while the controller is down, no ConfigMap gets created, no status is updated. When the controller restarts, it does a full reconciliation — it lists all URLShortener resources and compares desired vs actual state for each one, creating any missing resources. This is why controllers must be idempotent — they're designed to run repeatedly and produce the same result.

**Q3: What is a finalizer and when do you need one?**

A finalizer is a key in metadata.finalizers that blocks object deletion until a controller removes it. When you delete a resource with finalizers, K8s sets deletionTimestamp but doesn't delete it — the object enters a terminating state. Your controller sees the deletionTimestamp, runs cleanup logic (delete DNS records, revoke API tokens, remove from external load balancer), then removes the finalizer. Only then does K8s complete the deletion. Use finalizers whenever your operator creates external resources that won't be garbage-collected by K8s on their own.

**Q4: How does Istio's mTLS differ from TLS at the Ingress layer?**

Ingress TLS terminates at the Ingress controller — traffic inside the cluster between services is plain text. Istio mTLS encrypts pod-to-pod traffic inside the cluster using mutual TLS — both sides authenticate with certificates issued by Istio's CA (Citadel). Every pod has a SPIFFE identity, and the Envoy sidecar handles cert rotation transparently. In STRICT mode, only authenticated services can communicate — a compromised pod can't spoof another service's identity. Ingress TLS protects north-south traffic; Istio mTLS protects east-west.

**Q5: What are the limits of a single Kubernetes cluster and when do you go multi-cluster?**

Kubernetes supports up to 5,000 nodes and 150,000 pods per cluster per the official scalability SLOs. Beyond that, go multi-cluster. But scale isn't the only reason: regulatory requirements may demand separate clusters per region for data residency. Blast radius isolation — a misconfigured namespace in a shared cluster can impact everyone. Upgrade risk — you can upgrade clusters one at a time. Team autonomy — platform teams can give teams their own cluster with full admin access without affecting others. Most organizations go multi-cluster before hitting node limits, purely for isolation.

**Q6: What is the controller watch cache and why does it matter for performance?**

controller-runtime maintains an in-memory cache of all watched resources, populated by a long-lived List+Watch against the API server. When your reconciler calls r.Get(ctx, key, obj), it reads from this local cache — not from the API server. This is critical: without the cache, every reconciliation loop would hit the API server, causing thundering herd problems at scale. The tradeoff: cache reads can be slightly stale (eventually consistent). For writes (Create, Update, Delete) you always hit the API server directly. This is why you should never write business logic that assumes immediate consistency after a write — always re-read from cache on the next reconcile.
















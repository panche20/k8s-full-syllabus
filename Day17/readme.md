# Day 17 — Advanced Scheduling + Cluster Internals

*Week 3 final day. Today covers the scheduling layer that most engineers never touch but every senior interview tests.*

## 🧠 Part 1: The Scheduler Deep Dive

*The default scheduler (kube-scheduler) assigns pods to nodes in two phases:*

```
Phase 1: Filtering    — eliminate nodes that CANNOT run the pod
Phase 2: Scoring      — rank remaining nodes, pick the highest score
```

Understanding both phases is what lets you control exactly where workloads land.

**Filtering plugins (eliminators)**

```
NodeResourcesFit      — node has enough CPU/memory
NodeAffinity          — pod's nodeAffinity rules match
TaintToleration       — pod tolerates node's taints
VolumeBinding         — required PVCs can be bound on this node
NodePorts             — hostPort not already in use
PodTopologySpread     — topology spread constraints satisfied
InterPodAffinity      — pod affinity/anti-affinity rules
NodeUnschedulable     — node is not cordoned
```

**Scoring plugins (rankers)**

```
LeastAllocated        — prefer nodes with most free resources
BalancedAllocation    — balance CPU and memory usage evenly
NodeAffinity          — nodes matching preferred affinity score higher
ImageLocality         — prefer nodes that already have the image pulled
InterPodAffinity      — prefer nodes where affinity pods run
TaintToleration       — fewer tolerations needed = higher score
```

## 📌 Part 2: Node Selectors and Node Affinity

**nodeSelector — simple, blunt**

```
spec:
  nodeSelector:
    disk: ssd                 # must match exactly — AND logic
    region: eu-west-1         # both labels required
```

**nodeAffinity — expressive, powerful**

```
spec:
  affinity:
    nodeAffinity:

      # HARD rule — pod won't schedule if not satisfied
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:             # terms are OR'd
          - key: kubernetes.io/arch
            operator: In
            values: [amd64, arm64]
          - key: node-type
            operator: NotIn
            values: [spot]             # don't schedule on spot instances
        - matchExpressions:            # this entire block is OR'd with above
          - key: cloud.google.com/gke-nodepool
            operator: Exists

      # SOFT rule — prefer but don't require
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                     # higher weight = stronger preference
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [eu-west-1a]
      - weight: 20
        preference:
          matchExpressions:
          - key: tier
            operator: In
            values: [premium]
```

**Operators: In, NotIn, Exists, DoesNotExist, Gt, Lt**

**Interview trap:**

IgnoredDuringExecution means if a node label changes AFTER the pod is scheduled, the pod keeps running there. RequiredDuringExecution (not yet implemented) would evict it. Know this distinction.

## 🤝 Part 3: Pod Affinity and Anti-Affinity

Whereas node affinity attracts pods to nodes based on node labels, pod affinity attracts pods to nodes based on what other pods are already running there.

**Pod affinity — co-locate pods**

```
spec:
  affinity:
    podAffinity:
      # HARD: must run on same node as pods with app=cache
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: redis-cache
        topologyKey: kubernetes.io/hostname   # same node
        namespaceSelector: {}                 # search all namespaces

      # SOFT: prefer same zone as app=frontend
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: frontend
          topologyKey: topology.kubernetes.io/zone   # same AZ
```

**Pod anti-affinity — spread pods apart**

```
spec:
  affinity:
    podAntiAffinity:
      # HARD: no two pods of this app on same node
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: url-shortener
        topologyKey: kubernetes.io/hostname

      # SOFT: prefer different AZs
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: url-shortener
          topologyKey: topology.kubernetes.io/zone
```

**Production pattern:** 

Anti-affinity with topologyKey: kubernetes.io/hostname guarantees no two replicas of your app share a node. This means a node failure takes down at most one replica. Essential for high-availability deployments.

## 🌐 Part 4: Topology Spread Constraints

Anti-affinity is binary — it either allows or blocks. Topology Spread Constraints are smarter — they distribute pods evenly across failure domains with fine-grained control.

```
spec:
  topologySpreadConstraints:
  - maxSkew: 1                          # max difference between most and least loaded zone
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule    # hard — or ScheduleAnyway (soft)
    labelSelector:
      matchLabels:
        app: url-shortener
    minDomains: 3                       # require at least 3 zones to exist

  - maxSkew: 2
    topologyKey: kubernetes.io/hostname  # also spread across nodes
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: url-shortener
```

*Example with 3 zones and 6 replicas:*

```
maxSkew: 1 means:
  zone-a: 2 pods
  zone-b: 2 pods
  zone-c: 2 pods   ← perfect spread

  zone-a: 3, zone-b: 2, zone-c: 1 → skew=2 → violates maxSkew:1
  zone-a: 2, zone-b: 2, zone-c: 2 → skew=0 → ✓
```

## ⚖️ Part 5: Priority Classes

When the cluster is under resource pressure, the scheduler needs to know which pods to evict and which to protect. PriorityClass defines this.

```
# System-critical — protect cluster components
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000              # higher value = higher priority
globalDefault: false
preemptionPolicy: PreemptLowerPriority   # can evict lower priority pods
description: "For critical system components"
---
# High — production workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Production workloads"
---
# Default — standard workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 1000
globalDefault: true         # applied to pods with no priorityClassName
description: "Standard workloads"
---
# Low — batch jobs, can be evicted
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
preemptionPolicy: Never     # won't preempt others even if higher priority pod is pending
description: "Batch and background jobs"
```

```
# Apply to a pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:1.25
```

**How preemption works**

```
Cluster is full. High-priority pod is Pending.
Scheduler finds: if low-priority pod is evicted from node X,
                 high-priority pod fits on node X.
Scheduler evicts low-priority pod → schedules high-priority pod.
Low-priority pod goes back to Pending.
```

```
# See all priority classes
kubectl get priorityclass

# Built-in system classes
kubectl get priorityclass | grep system
# system-cluster-critical   2000000000
# system-node-critical      2000001000
# These protect CoreDNS, kube-proxy etc from preemption
```

## 🔧 Part 6: Custom Scheduler

You can run multiple schedulers and assign specific pods to a non-default one. Used for: ML workloads (custom GPU packing), batch (gang scheduling), specific hardware.

```
# Deploy a second scheduler
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: custom-scheduler
  template:
    metadata:
      labels:
        component: custom-scheduler
    spec:
      serviceAccountName: custom-scheduler-sa
      containers:
      - name: scheduler
        image: registry.k8s.io/kube-scheduler:v1.30.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/custom-scheduler-config.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/kubernetes/
      volumes:
      - name: config
        configMap:
          name: custom-scheduler-config
---
# KubeSchedulerConfiguration
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-scheduler-config
  namespace: kube-system
data:
  custom-scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    schedulerName: custom-scheduler    # unique name
    profiles:
    - schedulerName: custom-scheduler
      plugins:
        score:
          disabled:
          - name: NodeResourcesBalancedAllocation
          enabled:
          - name: NodeResourcesFit
            weight: 2
```

```
# Assign pod to custom scheduler
spec:
  schedulerName: custom-scheduler    # default is "default-scheduler"
  containers:
  - name: app
    image: nginx:1.25
```

## 📊 Part 7: Resource Management Deep Dive

**Request vs Limit mechanics**

```
Requests: used by scheduler for node selection
          used by kubelet for QoS class
          guaranteed to the container

Limits:   enforced by cgroups at runtime
          CPU: throttled when exceeded (not killed)
          Memory: OOMKilled when exceeded
```

**CPU throttling — the silent performance killer**

```
# Check if your containers are being CPU throttled
# This is the most common performance issue nobody notices

kubectl exec -it <pod> -- cat \
  /sys/fs/cgroup/cpu/cpu.stat | grep throttled

# Or via Prometheus:
# rate(container_cpu_throttled_seconds_total[5m]) /
# rate(container_cpu_usage_seconds_total[5m])
# > 0.25 means 25% of time is throttled — bad

# Fix: increase CPU limit or reduce CPU request ratio
# Rule of thumb: limit should be 2-4x request for bursty apps
```

**VPA — Vertical Pod Autoscaler**

HPA scales horizontally (more pods). VPA scales vertically (bigger pods).

```
# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-install.sh
```

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: url-shortener-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: url-shortener
  updatePolicy:
    updateMode: "Auto"          # Auto | Recreate | Initial | Off
    # Auto: evicts and restarts pods with new resource recommendations
    # Off: only shows recommendations, doesn't apply
  resourcePolicy:
    containerPolicies:
    - containerName: fastapi
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: "2"
        memory: 1Gi
      controlledResources: [cpu, memory]
```

```
# See VPA recommendations (use Off mode first to understand before Auto)
kubectl describe vpa url-shortener-vpa
# Recommendation:
#   Container Recommendations:
#     Container Name: fastapi
#     Lower Bound:    cpu: 150m, memory: 200Mi
#     Target:         cpu: 300m, memory: 350Mi   ← apply this
#     Upper Bound:    cpu: 800m, memory: 900Mi
```

**Important:** 

Don't use HPA and VPA on CPU/memory simultaneously — they fight each other. Use HPA on CPU, VPA on memory. Or use HPA only with custom metrics and VPA for CPU/memory.

## 🎯 Part 8: Descheduler — Fix Scheduling Drift

The scheduler only runs when pods are created. If nodes are added later, existing pods don't rebalance. The Descheduler evicts pods so they get rescheduled to better nodes.

```
helm repo add descheduler \
  https://kubernetes-sigs.github.io/descheduler/
helm repo update

helm install descheduler \
  descheduler/descheduler \
  --namespace kube-system \
  --set schedule="*/5 * * * *"    # run every 5 minutes as CronJob
```

```
# Descheduler policy
apiVersion: descheduler/v1alpha2
kind: DeschedulerPolicy
profiles:
- name: default
  pluginConfig:
  - name: RemoveDuplicates          # ensure no two replicas on same node
    args:
      namespaces:
        include: [production]

  - name: RemovePodsViolatingNodeAffinity
    args:
      nodeAffinityType: [requiredDuringSchedulingIgnoredDuringExecution]

  - name: LowNodeUtilization        # move pods from hot nodes to cold
    args:
      thresholds:
        cpu: 20                     # nodes below 20% are underutilized
        memory: 20
        pods: 20
      targetThresholds:
        cpu: 50                     # don't move to nodes above 50%
        memory: 50
        pods: 50

  - name: RemovePodsViolatingTopologySpreadConstraint
```

## 🖥️ Part 9: Hands-On Exercises

**Exercise 1: Anti-affinity high availability**

```
# Create 3-replica deployment with hard anti-affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: ha-app
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests: {cpu: 50m, memory: 32Mi}
EOF

# Verify pods spread across nodes
kubectl get pods -l app=ha-app -o wide
# Each pod on a different node

# Try scaling to 4 — 4th should stay Pending
# (only 3 nodes in kind cluster, hard anti-affinity)
kubectl scale deployment ha-app --replicas=4
kubectl get pods -l app=ha-app -o wide
kubectl describe pod -l app=ha-app | grep -A 5 "Events:"
# 0/3 nodes available: 3 node(s) didn't match pod anti-affinity rules
```

**Exercise 2: Topology spread constraints**

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: spread-app
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests: {cpu: 50m, memory: 32Mi}
EOF

# Verify even spread
kubectl get pods -l app=spread-app -o wide \
  | awk '{print $7}' | sort | uniq -c
# 2 k8s-mastery-control-plane
# 2 k8s-mastery-worker
# 2 k8s-mastery-worker2
# Perfect spread — 2 per node
```

**Exercise 3: Priority classes and preemption**

```
# Create priority classes
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
preemptionPolicy: Never
EOF

# Fill the cluster with low-priority pods
kubectl create deployment low-filler \
  --image=nginx:1.25 \
  --replicas=20
kubectl set resources deployment low-filler \
  --requests=cpu=200m,memory=128Mi
kubectl patch deployment low-filler \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"low-priority"}}}}'

# Wait for pods to fill nodes
kubectl get pods -l app=low-filler -o wide

# Now create a high-priority pod that needs resources
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: urgent-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
EOF

# Watch preemption happen
kubectl get pods -w
# urgent-pod goes Pending briefly
# then a low-priority pod gets evicted
# urgent-pod becomes Running
kubectl describe pod urgent-pod | grep -A 5 "Events:"
```

**Exercise 4: VPA recommendations**

```
# Deploy with intentionally wrong resource settings
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo
  template:
    metadata:
      labels:
        app: vpa-demo
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: 1000m       # way too high
            memory: 1Gi      # way too high
          limits:
            cpu: 2000m
            memory: 2Gi
EOF

# Create VPA in Off mode — just recommendations
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 10m
        memory: 32Mi
      maxAllowed:
        cpu: "1"
        memory: 512Mi
EOF

# Wait a few minutes for VPA to observe
sleep 120

# See what VPA recommends
kubectl describe vpa vpa-demo
# Target recommendation will be much lower than 1000m CPU
```

**Exercise 5: Scheduler debugging**

```
# Create a pod that can't be scheduled and diagnose why
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unschedulable-pod
spec:
  nodeSelector:
    gpu: "true"                 # no node has this
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: memory-type
            operator: In
            values: [HBM2]     # also doesn't exist
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
EOF

kubectl get pod unschedulable-pod
kubectl describe pod unschedulable-pod | grep -A 10 Events

# Scheduler logs
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system \
    -l component=kube-scheduler \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep unschedulable-pod | tail -10
```

## 🎯 Part 10: Interview Questions — Day 20

**Q1: What's the difference between pod affinity and node affinity?**

Node affinity attracts pods to nodes based on node labels — the node itself has properties (disk type, zone, hardware). Pod affinity attracts pods to nodes based on what other pods are already running there — useful for co-locating an app with its cache for low latency. Pod anti-affinity repels pods from nodes where certain pods run — used to spread replicas across nodes for HA. Node affinity is about node properties; pod affinity is about pod co-location.

**Q2: A Deployment has 5 replicas but 3 pods are on one node. How do you fix this without deleting pods?**

Add a topologySpreadConstraints with maxSkew: 1 and topologyKey: kubernetes.io/hostname. Then either wait for the Descheduler to evict and rebalance (if installed), or do a rolling restart: kubectl rollout restart deployment/<name>. During the rolling restart, the scheduler places new pods according to the spread constraint, achieving even distribution. The ScheduleAnyway option for whenUnsatisfiable makes it a soft constraint.

**Q3: What happens to a running pod when its node's label changes and the pod has requiredDuringSchedulingIgnoredDuringExecution node affinity?**

Nothing — the pod keeps running. The IgnoredDuringExecution part means the rule is only evaluated at scheduling time, not continuously. The pod was placed there when the label existed, and it stays. RequiredDuringExecution (not yet stable in K8s) would evict the pod when the label is removed. This is a common interview trap — the naming is counter-intuitive.

**Q4: How does CPU throttling differ from OOMKill and why does it matter for performance?**

OOMKill is violent — when a container exceeds its memory limit, the kernel kills it immediately (exit code 137). CPU throttling is silent — when a container exceeds its CPU limit, the kernel simply doesn't give it CPU time slices. The container stays running but slows down. This is why CPU throttling is dangerous: your pod looks healthy (no restarts, status Running) but latency is spiking and requests are timing out. Always monitor container_cpu_throttled_seconds_total in Prometheus — if throttling exceeds 25%, increase your CPU limit.

**Q5: A high-priority pod is Pending even though there are lower-priority pods running. Why might preemption not occur?**

Several reasons: the lower-priority pods might have preemptionPolicy: Never. A PodDisruptionBudget might prevent evicting enough pods to free up resources. The high-priority pod might have affinity/anti-affinity rules that rule out all nodes even after eviction. The resource request might be too large for any single node even if fully emptied. Or the lower-priority pods are in namespaces with preemption-policy: Never on their PriorityClass. Check kubectl describe pod <pending-pod> — the scheduler records exactly why preemption didn't succeed.

**Q6: When would you use a custom scheduler instead of configuring the default one?**

When the default scheduler's plugins fundamentally don't fit your workload. Examples: GPU cluster packing (want to fill GPUs completely, not balance across nodes — opposite of default behavior), gang scheduling for ML training (all pods of a job must schedule simultaneously or none do — default scheduler doesn't support this), latency-sensitive workloads needing NUMA-aware scheduling, or multi-cluster federation. For most cases, configure the default scheduler via affinity, taints, topology constraints, and priority classes — custom schedulers add significant operational complexity.

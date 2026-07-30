# Day 12 — Kubernetes Troubleshooting Deep Dive + CKA Mock Review

## Advanced Troubleshooting + Cluster Internals

Today goes deeper than Day 11 — real multi-component failures, control plane internals, and the kind of scenarios that appear in senior engineer interviews and the hardest CKA tasks.

## 🧠 Part 1: Troubleshooting Mental Stack — Advanced

*Day 11 gave you the 5-layer system. Today adds two more layers for when those five don't solve it:*

```
1. Symptom          kubectl get
2. Events           kubectl describe
3. Logs             kubectl logs
4. Config           kubectl get -o yaml
5. Node             journalctl, systemctl
─────────────────────────────────────────── Day 11 stops here
6. Network          tcpdump, netstat, iptables, DNS
7. Control plane    etcd keys, API server audit log, controller watches
```

- Layers 6 and 7 are where production incidents live.

## 🔬 Part 2: Network-Level Debugging

*When kubectl exec + curl isn't enough.*

```
# ── DNS deep dive ────────────────────────────────────────────

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Common CoreDNS issues:
# - forward . /etc/resolv.conf  pointing to a broken upstream
# - custom domain not in hosts block
# - CoreDNS pod crashed (check restarts)
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Debug DNS from inside a pod
kubectl run dns-debug --image=busybox:1.35 --rm -it -- sh
  nslookup kubernetes.default.svc.cluster.local
  nslookup google.com
  cat /etc/resolv.conf
  # nameserver 10.96.0.10  ← CoreDNS ClusterIP
  # search default.svc.cluster.local svc.cluster.local cluster.local

# Check CoreDNS ClusterIP
kubectl get svc kube-dns -n kube-system
# should match the nameserver in /etc/resolv.conf

# ── iptables — what kube-proxy actually wrote ────────────────

# On a node — see all KUBE-SERVICES chains
docker exec k8s-mastery-control-plane \
  iptables -t nat -L KUBE-SERVICES -n --line-numbers | head -40

# Trace a specific Service ClusterIP → pod IP mapping
docker exec k8s-mastery-control-plane \
  iptables -t nat -L KUBE-SVC-XXXX -n

# Check if kube-proxy is syncing rules
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -20

# ── Pod-to-pod connectivity test ─────────────────────────────

# Deploy two pods on different nodes
kubectl run pod-a --image=busybox:1.35 \
  --overrides='{"spec":{"nodeName":"k8s-mastery-worker"}}' \
  -- sleep 3600
kubectl run pod-b --image=busybox:1.35 \
  --overrides='{"spec":{"nodeName":"k8s-mastery-worker2"}}' \
  -- sleep 3600

# Get pod IPs
kubectl get pods -o wide

# Test cross-node pod networking
kubectl exec pod-a -- ping -c 3 <pod-b-ip>

# If ping fails — CNI issue
kubectl get pods -n kube-system | grep -E 'calico|cilium|flannel|weave'
kubectl describe pod -n kube-system <cni-pod>

# ── tcpdump on a node (when you need to see raw packets) ──────

docker exec k8s-mastery-worker bash -c \
  "apt-get install -y tcpdump -q && \
   tcpdump -i eth0 -n port 8000 -c 20"
```

## 🔬 Part 3: Control Plane Internal Debugging

```
# ── Watch what the controller-manager is doing ───────────────

kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-controller-manager \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep -E 'error|Error|warning' | tail -30

# ── Watch what the scheduler is doing ────────────────────────

kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-scheduler \
    -o jsonpath='{.items[0].metadata.name}') \
  | tail -30

# Filter for scheduling failures
kubectl logs -n kube-system \
  -l component=kube-scheduler | grep -i "failed\|error\|skip"

# ── API server audit log ──────────────────────────────────────

# If audit logging is enabled (Day 10):
docker exec k8s-mastery-control-plane \
  tail -20 /var/log/kubernetes/audit.log \
  | python3 -m json.tool \
  | grep -E '"verb"|"user"|"resource"|"namespace"'

# Find who deleted a resource
docker exec k8s-mastery-control-plane \
  grep '"verb":"delete"' /var/log/kubernetes/audit.log \
  | python3 -m json.tool | grep -E 'user|resource|name' | head -30

# ── etcd — see what's stored for a broken resource ───────────

docker exec k8s-mastery-control-plane bash -c "
  ETCDCTL_API=3 etcdctl get \
    /registry/deployments/production/url-shortener \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --print-value-only | strings"

# List all keys for a namespace — see everything K8s has stored
docker exec k8s-mastery-control-plane bash -c "
  ETCDCTL_API=3 etcdctl get /registry \
    --prefix --keys-only \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    | grep '/production/' | head -30"
```
## 🩺 Advanced Scenario 1: Pod Stuck in Terminating

*The most common "nothing works" situation.*

```
# Symptom
kubectl get pods
# NAME      READY   STATUS        RESTARTS   AGE
# stuck     1/1     Terminating   0          48m    ← never finishes

# Why this happens:
# 1. Finalizer on the pod that never clears
# 2. Node is down — kubelet can't send container stopped signal
# 3. Volume unmount hanging

# Diagnose
kubectl get pod stuck -o yaml | grep -A 10 finalizers
# metadata:
#   finalizers:
#   - some.io/finalizer    ← blocking deletion

# Fix option 1: remove the finalizer
kubectl patch pod stuck \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge

# Fix option 2: force delete (node is down)
kubectl delete pod stuck --force --grace-period=0

# Fix option 3: if node NotReady and pod must go
kubectl delete pod stuck $now

# Verify
kubectl get pods   # stuck is gone
```

## 🩺 Advanced Scenario 2: Namespace Stuck in Terminating

```
kubectl get ns dying-namespace
# NAME             STATUS        AGE
# dying-namespace  Terminating   2h    ← stuck

# Why: namespace has resources with finalizers that never cleared
# Most common cause: CRDs deleted before their resources

# Find what's stuck
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -I{} kubectl get {} \
  --ignore-not-found \
  -n dying-namespace 2>/dev/null

# Nuclear fix: patch the namespace finalizer directly
kubectl get namespace dying-namespace -o json \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
d['spec']['finalizers']=[]
print(json.dumps(d))" \
  | kubectl replace --raw \
    /api/v1/namespaces/dying-namespace/finalize -f -

# Namespace deletes immediately
kubectl get ns dying-namespace   # gone
```

## 🩺 Advanced Scenario 3: Certificate Has Expired

```
# Symptom: kubectl stops working
kubectl get nodes
# Unable to connect to the server: x509: certificate has expired

# On the control plane node
docker exec k8s-mastery-control-plane bash

# Check all certs
kubeadm certs check-expiration
# CERTIFICATE    EXPIRES                    RESIDUAL TIME
# apiserver      Nov 15, 2024 10:00 UTC     -2d         ← expired

# Renew everything
kubeadm certs renew all

# Restart control plane — must re-read certs
cd /etc/kubernetes/manifests
mkdir /tmp/manifests-bak
mv *.yaml /tmp/manifests-bak/
sleep 10
mv /tmp/manifests-bak/*.yaml .

# Update local kubeconfig (it also has an embedded cert)
cp /etc/kubernetes/admin.conf $HOME/.kube/config

exit

# Test
kubectl get nodes   # works again
```

## 🩺 Advanced Scenario 4: etcd Has No Leader

```
# Symptom: all kubectl commands hang or return:
# etcdserver: no leader

# Check etcd pod
kubectl get pods -n kube-system -l component=etcd
# etcd-k8s-mastery-control-plane   0/1   CrashLoopBackOff

# Get logs directly via crictl
docker exec k8s-mastery-control-plane bash -c \
  "crictl logs \$(crictl ps -a | grep etcd | head -1 | awk '{print \$1}')"

# Common causes:
# 1. Quorum lost (majority of etcd nodes down in HA cluster)
# 2. Data directory corrupted
# 3. Clock skew between nodes (etcd is time-sensitive)

# Check time sync
docker exec k8s-mastery-control-plane date
date   # compare with your machine

# Fix clock skew
docker exec k8s-mastery-control-plane bash -c \
  "apt-get install -y ntpdate && ntpdate pool.ntp.org"

# If data corruption — restore from backup (Day 10 procedure)
# If quorum lost — force new cluster from last good member:
docker exec k8s-mastery-control-plane bash -c \
  "sed -i 's|--initial-cluster-state=existing|--initial-cluster-state=new|' \
   /etc/kubernetes/manifests/etcd.yaml"
```

## 🩺 Advanced Scenario 5: Webhook Blocking Everything

```
# Symptom: ALL resource creation fails with:
# Error from server (InternalError): Internal error occurred:
# failed calling webhook "validate.example.com"

kubectl apply -f anything.yaml
# Error: failed calling webhook

# Find the webhook
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Describe it — find which webhook is failing
kubectl describe validatingwebhookconfiguration my-webhook
# FailurePolicy: Fail    ← this means webhook failure blocks ALL requests

# Quick fix: delete the broken webhook (if it's a dev environment)
kubectl delete validatingwebhookconfiguration my-webhook

# Production fix: change FailurePolicy to Ignore temporarily
kubectl patch validatingwebhookconfiguration my-webhook \
  --type=json \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

# Then fix the webhook service itself
kubectl get pods -n webhook-system   # find broken webhook pod
kubectl logs -n webhook-system <webhook-pod>
```

## 🩺 Advanced Scenario 6: HPA Not Scaling

```
# Symptom: CPU is high, HPA exists, but no scaling
kubectl get hpa
# NAME   REFERENCE         TARGETS         MINPODS   MAXPODS   REPLICAS
# web    Deployment/web    <unknown>/70%   2         10        2
#                           ↑
#                    <unknown> = metrics not available

# Diagnosis tree:

# 1. Is metrics-server running?
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes   # if this fails, metrics-server is broken

# Fix metrics-server on kind (needs insecure TLS)
kubectl patch deployment metrics-server \
  -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-",
        "value":"--kubelet-insecure-tls"}]'

# 2. Do pods have resource requests set?
kubectl get deployment web -o yaml | grep -A 5 resources
# If no requests — HPA can't compute utilization

# Fix: add requests
kubectl set resources deployment web \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi

# 3. Check HPA events
kubectl describe hpa web
# Events:
#   Warning  FailedGetScale  unable to fetch metrics

# 4. Verify metrics API is available
kubectl get apiservices | grep metrics
# v1beta1.metrics.k8s.io   True   ← must be True
```

## 🩺 Advanced Scenario 7: PodDisruptionBudget Blocking Drain

```
# Symptom: kubectl drain hangs or fails
kubectl drain worker-node \
  --ignore-daemonsets \
  --delete-emptydir-data
# error: cannot evict pod as it would violate the pod's disruption budget

# Find the PDB
kubectl get pdb -A
# NAMESPACE     NAME         MIN-AVAILABLE   MAX-UNAVAILABLE   ALLOWED-DISRUPTIONS
# production    api-pdb      3               N/A               0
#                                                               ↑ zero allowed

# Why: you have 3 replicas, minAvailable=3
# Evicting one pod would take you to 2 — violates PDB

kubectl describe pdb api-pdb -n production
# Status:
#   Allowed Disruptions: 0    ← drain blocked

# Fix options:

# Option 1: scale up first, then drain
kubectl scale deployment api -n production --replicas=4
# Now Allowed Disruptions = 1 → drain can proceed

# Option 2: temporarily delete the PDB (planned maintenance)
kubectl delete pdb api-pdb -n production
kubectl drain worker-node --ignore-daemonsets --delete-emptydir-data
# Recreate PDB after

# Option 3: bypass PDB (emergency only — risky)
kubectl drain worker-node \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --disable-eviction    # skips PDB check, directly deletes pods
```

## 🖥️ Part 4: Hands-On — Advanced Break/Fix Scenarios

**Break 1: Multi-component failure**

```
# Break the kube-controller-manager AND create a deployment simultaneously
docker exec k8s-mastery-control-plane bash -c \
  "sed -i 's/kube-controller-manager/kube-controller-managerBROKEN/' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml"

# Create a deployment while controller is down
kubectl create deployment orphan --image=nginx:1.25 --replicas=3
kubectl get pods   # ReplicaSet created but pods may not be created

# Your tasks:
# 1. Identify what's broken (controller-manager)
# 2. Fix it
# 3. Verify the deployment reconciles to 3 pods after fix
```

**Break 2: Broken CNI causing pod networking failure**

```
# Simulate CNI failure on kind
docker exec k8s-mastery-worker bash -c \
  "mv /etc/cni/net.d/10-kindnet.conflist /tmp/"

# Create a pod — should go to worker
kubectl run cni-test --image=nginx:1.25 \
  --overrides='{"spec":{"nodeName":"k8s-mastery-worker"}}'

# Watch it fail to get network
kubectl get pod cni-test -w
# ContainerCreating... stuck

kubectl describe pod cni-test | grep -A 10 Events
# failed to setup network for sandbox: ...

# Your task: restore CNI and verify pod gets an IP
```

**Break 3: Full diagnosis drill**

```
# Apply 5 broken scenarios simultaneously
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: adv-broken-1
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:1.25
    livenessProbe:
      exec:
        command: ["cat", "/tmp/doesnotexist"]
      initialDelaySeconds: 3
      periodSeconds: 3
      failureThreshold: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: adv-broken-2
  namespace: default
spec:
  volumes:
  - name: pvc-vol
    persistentVolumeClaim:
      claimName: nonexistent-pvc
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: pvc-vol
      mountPath: /data
---
apiVersion: v1
kind: Pod
metadata:
  name: adv-broken-3
  namespace: default
spec:
  serviceAccountName: doesnotexist
  containers:
  - name: app
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: adv-broken-4
  namespace: default
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["sh","-c","exit 1"]
  containers:
  - name: app
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: adv-broken-5
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      limits:
        memory: "1Mi"     # impossibly small limit
EOF

# Your task: for each pod write:
# 1. The status you see
# 2. The root cause
# 3. The fix
# Use ONLY: kubectl get, describe, logs, events
```

## 🎯 Part 5: Advanced Interview Questions

**Q1: A Deployment shows 3/3 desired but pods are being replaced constantly. What's happening?**

Liveness probe is too aggressive — it's killing pods before they're stable, and the Deployment keeps replacing them. This creates a churn loop: pod starts, liveness fires, pod killed, new pod starts. Check kubectl describe pod — look for Liveness probe failed events and high restart counts with short ages. Fix by adding a startupProbe with generous failureThreshold, or increasing initialDelaySeconds on liveness. Also check if the probe endpoint is actually reachable within the pod.

**Q2: You notice pods scheduled on a node aren't starting. kubectl shows ContainerCreating for 10+ minutes.**

This is almost always a CNI or container runtime issue. Check kubectl describe pod — events show failed to create containerd task or network setup failed. On the node check systemctl status containerd and journalctl -u kubelet. For CNI: check CNI plugin pods in kube-system, verify /etc/cni/net.d/ has a valid config, check the CNI binary exists in /opt/cni/bin/. The specific error in the events points to exactly which layer failed.

**Q3: Your cluster was working. After a weekend nobody touched it, Monday morning kubectl returns 403 Forbidden for everything. What happened?**

Bootstrap token expired, or the service account token used by your kubeconfig expired. Check cert expiry: kubeadm certs check-expiration. If certs are fine, check the token in your kubeconfig: kubectl config view --raw | grep token — if using a static token it may have been rotated. Also check if a new admin changed RBAC over the weekend. The 403 pattern (not 401, not connection refused) specifically means authentication succeeded but authorization failed — focus on RBAC changes first, cert expiry second.

**Q4: How does Kubernetes handle a node that becomes unreachable? Walk through the full sequence.**

Node stops sending heartbeats to the API server. After node-monitor-grace-period (default 40s) the node condition goes to Unknown. After pod-eviction-timeout (default 5 minutes) the node controller starts evicting pods — adding a NoExecute taint node.kubernetes.io/unreachable. Pods without tolerations for this taint are evicted and rescheduled elsewhere. Pods with tolerationSeconds stay for that duration. StatefulSet pods are NOT rescheduled until the node comes back or is forcibly deleted, to prevent split-brain on stateful workloads.

**Q5: You have a memory leak in production. Pods keep getting OOMKilled but you can't increase limits right now. What do you do?**

Short term: add a liveness probe with a memory threshold check, or configure the pod to restart itself when memory crosses a threshold. Set restartPolicy: Always (default) so OOMKills are followed by automatic restarts. Increase replicaCount so the blast radius of any single OOMKill is smaller. Long term: profile the application — kubectl exec into a pod and check /sys/fs/cgroup/memory/memory.usage_in_bytes and memory.stat. Use kubectl top pods --containers to track which container is leaking. Enable a Prometheus alert on memory approaching limits so you catch it before OOMKill.

**Q6: A critical production pod has been in Terminating state for 2 hours. How do you handle it?**

First understand why before forcing deletion. Check kubectl get pod -o yaml for finalizers — if a finalizer is blocking, find which controller owns it and whether that controller is healthy. Check if the node is up — if the node is dead, --force --grace-period=0 is the right call since the kubelet can't signal the container to stop. Check if a volume is stuck unmounting with kubectl describe pod events. In production: patch finalizers to empty array first (clean), then force delete if patching doesn't resolve within a minute. Document the force deletion — it bypasses graceful shutdown, which may leave resources in a bad state in the underlying infrastructure.















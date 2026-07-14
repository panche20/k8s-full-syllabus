# Day 11 — Troubleshooting Like a Pro

Troubleshooting is 30% of the CKA exam by weight. It's also what separates a $80k K8s engineer from a $160k one. 
Today is pure diagnostic methodology — not theory, but a repeatable system you apply under pressure.

## 🧠 Part 1: The Troubleshooting System

*Never random-guess in K8s. Always follow this hierarchy:*

```
1. What is the SYMPTOM?          kubectl get <resource> — find the bad state
2. What are the EVENTS?          kubectl describe — read the Events section
3. What are the LOGS?            kubectl logs — container stdout/stderr
4. What is the CONFIG?           kubectl get -o yaml — full spec vs actual
5. What does the NODE say?       kubectl describe node, systemctl, journalctl
```

*Stop at whichever layer gives you the answer. Most problems are solved at layer 2 or 3.*

## 🔧 Part 2: Essential Diagnostic Commands

*Burn these into muscle memory:*

```
# ── Pod triage ──────────────────────────────────────────────
kubectl get pods -A                          # all pods, all namespaces
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get pod <name> -o wide              # which node, pod IP
kubectl get pod <name> -o yaml              # full spec + status
kubectl describe pod <name>                 # human-readable + Events

# ── Logs ────────────────────────────────────────────────────
kubectl logs <pod>                           # current container
kubectl logs <pod> --previous               # previous (crashed) container
kubectl logs <pod> -c <container>           # specific container
kubectl logs <pod> --tail=50 -f             # stream last 50 lines

# ── Exec ────────────────────────────────────────────────────
kubectl exec -it <pod> -- bash              # shell in pod
kubectl exec -it <pod> -- sh               # if no bash
kubectl exec -it <pod> -c <container> -- sh # specific container

# ── Events ─────────────────────────────────────────────────
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events -A --field-selector=type=Warning

# ── Node triage ─────────────────────────────────────────────
kubectl get nodes
kubectl describe node <name>                # conditions, capacity, pods
kubectl top nodes                           # CPU + memory usage
kubectl top pods -A --sort-by=memory       # most memory hungry pods

# ── Control plane ───────────────────────────────────────────
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system kube-controller-manager-<node>

# ── On the node (SSH or docker exec) ────────────────────────
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager
systemctl status containerd
crictl pods                                  # container runtime pods
crictl ps -a                                 # all containers including crashed
crictl logs <container-id>
```

## 🩺 Scenario 1: CrashLoopBackOff

*Symptom: Pod status is CrashLoopBackOff, restarts incrementing.*

```
bash# Step 1 — see the state
kubectl get pod broken-app
# NAME         READY   STATUS             RESTARTS   AGE
# broken-app   0/1     CrashLoopBackOff   5          3m

# Step 2 — events
kubectl describe pod broken-app | tail -20
# Events:
#   Warning  BackOff  kubelet  Back-off restarting failed container

# Step 3 — THIS is the key for CrashLoopBackOff
kubectl logs broken-app --previous     # logs from the LAST crashed run
# Error: REDIS_HOST environment variable not set
# Traceback...

# Root cause: missing env var
# Fix: add the env var to the pod/deployment spec
kubectl set env deployment/broken-app REDIS_HOST=redis-service
```

**CrashLoopBackOff causes and fixes:**

<img width="875" height="357" alt="image" src="https://github.com/user-attachments/assets/0995feca-35a9-4f0e-b329-e7db667a4b2b" />


## 🩺 Scenario 2: ImagePullBackOff / ErrImagePull

Symptom: Pod stuck in ImagePullBackOff or ErrImagePull.

```
bashkubectl describe pod bad-image-pod
# Events:
#   Warning  Failed  kubelet  Failed to pull image "ghcr.io/user/app:v99":
#            rpc error: code = NotFound

# Three root causes — events tell you which:
```

<img width="866" height="202" alt="image" src="https://github.com/user-attachments/assets/cbd1270d-4a4a-45f9-8924-105e5595b137" />

```
# Fix 1: wrong tag — patch the deployment
kubectl set image deployment/my-app app=ghcr.io/user/app:v2

# Fix 2: private registry — create pull secret and attach
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=youruser \
  --docker-password=ghp_token

kubectl patch deployment my-app -p \
  '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}}}'

# Verify image exists before deploying
docker pull ghcr.io/user/app:v2
```

## 🩺 Scenario 3: Pending Pod — Never Scheduled

**Symptom:** 

Pod stays Pending indefinitely.

```
kubectl describe pod pending-pod | grep -A 20 Events
# Events:
#   Warning  FailedScheduling  default-scheduler
#            0/3 nodes are available:
#            1 node(s) had untolerated taint,
#            2 Insufficient memory
```

**Read the scheduler message — it tells you exactly why:**

<img width="875" height="476" alt="image" src="https://github.com/user-attachments/assets/92e88121-42da-4bef-aa45-d16ad81d024d" />


```
# Diagnose resource pressure
kubectl describe nodes | grep -A 5 "Allocated resources"

# See what's eating resources on each node
kubectl top pods -A --sort-by=cpu

# Check if PVC is the problem
kubectl get pvc                   # look for Pending PVCs
kubectl describe pvc <name>       # see why it's not bound
```

## 🩺 Scenario 4: OOMKilled

**Symptom:** 

Container keeps dying, exit code 137, reason OOMKilled.

```
kubectl describe pod oom-pod
# Last State: Terminated
#   Reason:   OOMKilled
#   Exit Code: 137

kubectl top pod oom-pod --containers
# NAME       CPU(cores)   MEMORY(bytes)
# app        22m          248Mi          ← hitting the 256Mi limit
```

```
# Fix: increase memory limit
kubectl set resources deployment oom-deploy \
  --limits=memory=512Mi \
  --requests=memory=256Mi

# If you can't increase limit — find the memory leak
kubectl exec -it oom-pod -- sh
  cat /sys/fs/cgroup/memory/memory.usage_in_bytes
  # see current memory usage from inside

# Check if it's a memory leak vs legitimate growth
kubectl top pod oom-pod -w    # watch memory climb over time
```

## 🩺 Scenario 5: Node NotReady

**Symptom:** 

kubectl get nodes shows a node as NotReady.

```
kubectl get nodes
# NAME         STATUS     ROLES    AGE
# worker-1     NotReady   <none>   5d

kubectl describe node worker-1 | grep -A 10 Conditions
# Conditions:
#   Type              Status  Reason
#   MemoryPressure    False
#   DiskPressure      False
#   PIDPressure       False
#   Ready             False   KubeletNotReady   # ← kubelet not talking to API

# SSH to the node (or docker exec for kind)
docker exec -it k8s-mastery-worker bash

# Check kubelet
systemctl status kubelet
# ● kubelet.service - kubelet: The Kubernetes Node Agent
#    Loaded: loaded
#    Active: failed  ← bad

journalctl -u kubelet -n 50 --no-pager
# Common log messages and meanings:
```

<img width="847" height="382" alt="image" src="https://github.com/user-attachments/assets/834580e5-d5d0-4db0-96d4-2635d8e856d8" />

```
# Most common fix — restart kubelet
systemctl restart kubelet
systemctl status kubelet

# If disk pressure:
crictl rmi --prune                   # remove unused images
df -h                                # check disk usage
du -sh /var/log/pods/* | sort -rh | head -20   # find big log dirs
```

## 🩺 Scenario 6: Service Not Routing Traffic

**Symptom:** 

App is running, Service exists, but requests time out or get connection refused.

```
# Step 1 — verify pods are actually Running and Ready
kubectl get pods -l app=my-app
# 0/3 pods show READY 0/1 → readinessProbe failing

# Step 2 — check endpoints (if empty, Service routes to nothing)
kubectl get endpoints my-service
# NAME         ENDPOINTS         AGE
# my-service   <none>            5m   ← problem is here

# Step 3 — compare Service selector with pod labels
kubectl get svc my-service -o yaml | grep selector -A 5
# selector:
#   app: myapp            ← note: "myapp"

kubectl get pods --show-labels
# LABELS
# app=my-app              ← note: "my-app" — MISMATCH

# Fix: align labels
kubectl patch svc my-service -p '{"spec":{"selector":{"app":"my-app"}}}'

# Step 4 — verify endpoints populated
kubectl get endpoints my-service
# NAME         ENDPOINTS                           AGE
# my-service   10.244.1.5:8000,10.244.2.3:8000    5m  ← fixed
```

**Service troubleshooting checklist:**

```
# Does the Service exist?
kubectl get svc <name>

# Are there endpoints?
kubectl get endpoints <name>

# Do pod labels match Service selector?
kubectl get pods --show-labels | grep <expected-label>

# Is the targetPort correct?
kubectl describe svc <name> | grep TargetPort
kubectl describe pod <name> | grep -A 5 Ports

# Can you reach the pod directly (bypassing Service)?
kubectl exec test-pod -- wget -O- <pod-ip>:<port>

# Can you reach the Service ClusterIP?
kubectl exec test-pod -- wget -O- <cluster-ip>:<port>

# Is kube-proxy running?
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-<xxx>
```

## 🩺 Scenario 7: Ingress / Gateway API Not Working

**Symptom:** 

External traffic not reaching the app. Ingress exists, pods are running.

```
# Check Ingress controller is running
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | tail -30

# Check Ingress resource itself
kubectl describe ingress my-ingress
# Events:
#   Warning  Sync  nginx-controller  Error obtaining endpoints for service...

# Common Ingress problems:
# 1. ingressClassName doesn't match controller
kubectl get ingressclass                    # what classes exist?
kubectl get ingress -o yaml | grep ingressClassName

# 2. Backend service not found
kubectl get svc <backend-service-name>      # does it exist?

# 3. TLS secret missing
kubectl get secret <tls-secret-name>        # exists?

# ── Gateway API equivalent ───────────────────────────────────
# Check Gateway is Accepted and Programmed
kubectl get gateway -A
kubectl describe gateway my-gateway
# Conditions:
#   Type: Accepted — True
#   Type: Programmed — True   ← must be True for traffic to flow

# Check HTTPRoute is Accepted and has ResolvedRefs
kubectl get httproute -A
kubectl describe httproute my-route
# Conditions:
#   Type: Accepted — True
#   Type: ResolvedRefs — True  ← False means backend service not found

# Check GatewayClass
kubectl get gatewayclass                    # controller must be installed
kubectl describe gatewayclass nginx         # check Accepted condition
```


## 🩺 Scenario 8: PVC Stuck in Pending

**Symptom:**

PVC stays Pending, pod stuck in ContainerCreating.

```
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed
#            storageclass.storage.k8s.io "fast-ssd" not found

# Root causes:
# 1. StorageClass doesn't exist
kubectl get storageclass                    # check what exists

# 2. No PV matches (static provisioning)
kubectl get pv                              # is there a matching PV?
# PV must have: matching storageClassName, sufficient capacity, matching accessMode

# 3. PV is Released (still bound to old claim reference)
kubectl describe pv my-pv | grep Status
# Status: Released              ← needs manual cleanup

# Fix Released PV:
kubectl patch pv my-pv \
  -p '{"spec":{"claimRef":null}}'          # clear the claim reference
# PV goes back to Available

# 4. volumeBindingMode: WaitForFirstConsumer
# PVC stays Pending until pod is scheduled — this is NORMAL behaviour
kubectl describe pvc | grep "WaitForFirstConsumer"
```


## 🩺 Scenario 9: RBAC — Forbidden Errors

**Symptom:** 

Pod or user gets 403 Forbidden or Error from server (Forbidden).

```
# Identify exactly what's forbidden
kubectl get pods --as=chetan -n production
# Error from server (Forbidden): pods is forbidden:
# User "chetan" cannot list resource "pods" in API group ""
# in the namespace "production"

# Check what chetan CAN do
kubectl auth can-i --list --as=chetan -n production

# Check what RoleBindings exist for chetan
kubectl get rolebindings -n production -o yaml | grep -A 5 chetan
kubectl get clusterrolebindings -o yaml | grep -A 5 chetan

# For a ServiceAccount
kubectl auth can-i list pods \
  --as=system:serviceaccount:production:my-sa \
  -n production

# Fix: create missing binding
kubectl create rolebinding fix-chetan \
  --clusterrole=view \
  --user=chetan \
  -n production

# Verify immediately
kubectl auth can-i list pods --as=chetan -n production  # yes
```

## 🩺 Scenario 10: Control Plane Component Down

**Symptom:** 

kubectl commands hang or return errors like etcdserver: request timed out.

```
# Step 1 — check control plane pods
kubectl get pods -n kube-system
# kube-apiserver-...          CrashLoopBackOff  ← broken
# kube-scheduler-...          Running
# etcd-...                    Running

# Step 2 — check static pod logs
kubectl logs -n kube-system kube-apiserver-k8s-mastery-control-plane
# if kubectl is broken, go directly to the node:

docker exec -it k8s-mastery-control-plane bash
crictl logs $(crictl ps -a | grep apiserver | awk '{print $1}')

# Step 3 — check the static pod manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Look for: wrong flags, bad cert paths, missing files

# Common control plane break causes:
```

<img width="840" height="287" alt="image" src="https://github.com/user-attachments/assets/df9eade7-e9de-44fc-b3f3-beed3df04b5f" />

```
# If API server manifest has a typo — fix it directly
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# kubelet detects change and restarts the pod automatically

# Watch it come back
crictl pods | grep apiserver
```

## 🖥️ Part 3: Hands-On — Break and Fix Scenarios

**Break 1: Create a CrashLoopBackOff**

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crasher
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","echo starting && exit 1"]
EOF

kubectl get pod crasher -w
kubectl logs crasher --previous
# Diagnose: exit 1 = app error
# Fix: correct the command
kubectl delete pod crasher
```

**Break 2: Selector mismatch**

```
kubectl create deployment mismatch --image=nginx:1.25 --replicas=2
kubectl expose deployment mismatch --port=80 --target-port=80 --name=mismatch-svc

# Manually break the selector
kubectl patch svc mismatch-svc \
  -p '{"spec":{"selector":{"app":"wrong-label"}}}'

kubectl get endpoints mismatch-svc    # should show <none>

# Diagnose and fix yourself
# Hint: compare `kubectl get svc -o yaml` selector vs `kubectl get pods --show-labels`
```

**Break 3: Pending pod from resource pressure**

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-hog
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "999Gi"    # impossible request
        cpu: "999"
EOF

kubectl get pod resource-hog
kubectl describe pod resource-hog | grep -A 5 Events
# Diagnose: Insufficient memory
# Fix: reduce requests to something schedulable
```

**Break 4: Broken liveness probe**

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-probe
spec:
  containers:
  - name: app
    image: nginx:1.25
    livenessProbe:
      httpGet:
        path: /this-path-does-not-exist
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
EOF

kubectl get pod bad-probe -w
# Watch RESTARTS climb
kubectl describe pod bad-probe | grep -A 10 Events
# Events show: Liveness probe failed: HTTP probe failed with statuscode: 404
# Fix: correct the path to /
```

**Break 5: Full diagnosis drill — five broken pods**

```
# Deploy all five broken scenarios at once
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-1
spec:
  containers:
  - name: app
    image: nginx:doesnotexist999
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-2
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 5 && exit 1"]
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-3
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "500Gi"
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-4
spec:
  nodeSelector:
    disk: ssd-nvme-ultrapremium  # no node has this label
  containers:
  - name: app
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-5
spec:
  containers:
  - name: app
    image: nginx:1.25
    livenessProbe:
      exec:
        command: ["cat","/tmp/healthy"]
      initialDelaySeconds: 2
      periodSeconds: 3
      failureThreshold: 1
EOF

# Your task: diagnose each one with only kubectl commands
# Answer the question: what is broken and how do you fix it?
# Use: kubectl get, describe, logs, events
```

## 🎯 Part 4: Interview Questions — Day 11

**Q1: A pod is Running but not serving traffic. Walk me through your diagnosis.**

First check kubectl get endpoints <service> — if empty, the pod isn't in the Service's endpoint list. This means either the pod's readinessProbe is failing (check kubectl describe pod events) or the Service selector doesn't match pod labels (compare both with kubectl get svc -o yaml and kubectl get pods --show-labels). If endpoints are populated, check if the targetPort matches what the container is actually listening on.

**Q2: kubectl is completely unresponsive. What do you do?**

Go directly to the control plane node. Check if the API server static pod is running with crictl pods | grep apiserver. Check its logs with crictl logs <container-id>. Check the static pod manifest at /etc/kubernetes/manifests/kube-apiserver.yaml for errors. Check if etcd is healthy. If the API server is crashing due to a bad manifest edit, fix the YAML and kubelet will restart it automatically.

**Q3: A node shows MemoryPressure=True. What happens to pods on it?**

The node starts evicting pods in order of QoS class — BestEffort first (no requests/limits), then Burstable (requests < limits), and Guaranteed last (requests == limits). The scheduler also stops placing new pods on that node. Check kubectl describe node for eviction threshold settings. Fix by reducing memory usage, adding resources, or scaling down workloads.

**Q4: How do you debug a pod that crashes before you can exec into it?**

Use kubectl logs <pod> --previous to read logs from the last crashed container. Check kubectl describe pod for the lastState field which shows exit code and reason. If the exit code is 137 it's OOMKilled. If 1 it's an app error. If 127 it's command not found. For very fast crashes where even --previous is empty, add command: ["sleep","3600"] temporarily to override entrypoint and exec in to investigate the filesystem and environment.

**Q5: Pods on a node are being evicted even though the node looks healthy. Why?**

Check kubectl describe node for eviction thresholds — kubelet evicts pods when disk or memory crosses the eviction threshold even before node conditions show pressure. Also check if a PodDisruptionBudget is involved. Check for taints added to the node. Check if someone ran kubectl drain. Look at events: kubectl get events --field-selector=reason=Evicted -A.

**Q6: You applied a NetworkPolicy and now your app can't resolve DNS. What happened?**

The NetworkPolicy blocked egress to port 53 UDP. DNS in Kubernetes uses CoreDNS on UDP/TCP port 53, and if your egress policy doesn't explicitly allow port: 53, protocol: UDP to any namespace, DNS resolution breaks silently — the app hangs on hostname lookups. Always add an explicit egress rule for UDP 53 to any NetworkPolicy that restricts egress.

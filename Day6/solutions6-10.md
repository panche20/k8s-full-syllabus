# Task 6 – Jobs and CronJobs

## Objective

* Create a Job to calculate Pi to 500 digits.
* Create a CronJob that runs every 2 minutes.
* Configure Job retry behavior and CronJob concurrency policy.

```
# Step 1: Create the Namespace
kubectl create namespace ckad-6

# Step 2: Create the Job (pi-job)
kubectl create job pi-job \
  --image=perl:5.34 \
  --namespace=ckad-6 \
  --dry-run=client -o yaml \
  -- perl -Mbignum=bpi -wle 'print bpi(500)' > job.yaml

# Edit job.yaml to include backoffLimit: 2 and completions: 1 under spec:

apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
  namespace: ckad-6
spec:
  completions: 1
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: pi-job
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(500)"]
      restartPolicy: Never

# Apply the Job:
kubectl apply -f job.yaml

# Step 3: Create the CronJob (cleanup)
kubectl create cronjob cleanup \
  --image=busybox \
  --schedule="*/2 * * * *" \
  --namespace=ckad-6 \
  --dry-run=client -o yaml \
  -- /bin/sh -c 'echo cleanup ran at $(date)' > cronjob.yaml

# Edit cronjob.yaml to add concurrencyPolicy: Forbid under spec:
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
  namespace: ckad-6
spec:
  schedule: "*/2 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command: ["/bin/sh", "-c", "echo cleanup ran at $(date)"]
          restartPolicy: OnFailure

# Apply the CronJob:
kubectl apply -f cronjob.yaml

# Step 4: Verification
1. Verify pi-job
Check that the job pod ran and completed successfully:

kubectl get job pi-job -n ckad-6

Expected status: COMPLETIONS: 1/1
Check the computed $\pi$ digits in the pod logs:

kubectl logs -n ckad-6 -l job-name=pi-job

# 2. Verify cleanup CronJob
Check the CronJob status:

kubectl get cronjob cleanup -n ckad-6

# Wait up to 2 minutes, then check that the scheduled Job was spawned and completed:
kubectl get jobs -n ckad-6 --watch

# Check the log output from the triggered CronJob pod:
kubectl logs -n ckad-6 -l job-name=$(kubectl get jobs -n ckad-6 -o jsonpath='{.items[-1].metadata.name}')

# Check logs:
kubectl logs -n ckad-6 -l job-name=cleanup-29905210
```

# Task 7 – Resource Limits and QoS Classes

## Objective

Understand Kubernetes QoS Classes:

* Guaranteed
* Burstable
* BestEffort

---

## Create Namespace

```bash
kubectl create namespace ckad-7
```

---

## Guaranteed Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: ckad-7
spec:
  containers:
  - name: nginx
    image: nginx:1.25

    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"

      limits:
        cpu: "200m"
        memory: "128Mi"
```

---

## Burstable Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  namespace: ckad-7
spec:
  containers:
  - name: nginx
    image: nginx:1.25

    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
```

---

## BestEffort Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
  namespace: ckad-7
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

Apply:

```bash
kubectl apply -f qos-pods.yaml
```

---

## Verify QoS

```bash
kubectl get pod guaranteed-pod -n ckad-7 -o yaml | grep qosClass
kubectl get pod burstable-pod -n ckad-7 -o yaml | grep qosClass
kubectl get pod besteffort-pod -n ckad-7 -o yaml | grep qosClass
```

Expected:

```text
Guaranteed
Burstable
BestEffort
```

---

# Task 8 – NetworkPolicy

**Step 1: Ensure namespace exists and deploy Pods & Services**

```
kubectl create namespace ckad-8 --dry-run=client -o yaml | kubectl apply -f -
```

**Deploy the frontend and backend pods:**

```
kubectl run frontend --image=nginx:1.25 --labels="tier=frontend" -n ckad-8
kubectl run backend --image=nginx:1.25 --labels="tier=backend" -n ckad-8
```

*Expose both as ClusterIP services on port 80:*

```
kubectl expose pod frontend --port=80 --target-port=80 -n ckad-8
kubectl expose pod backend --port=80 --target-port=80 -n ckad-8
```

**Step 2: Baseline verification**

Test that frontend can communicate with backend:

```
kubectl exec -n ckad-8 frontend -- curl -s --connect-timeout 2 http://backend
```

*(You will see the default NGINX HTML output).*

**Step 3: Create and apply the NetworkPolicy**

Save the following manifest as backend-netpol.yaml:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: ckad-8
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Apply the policy:**

```
kubectl apply -f backend-netpol.yaml
```

**Step 4: Verify traffic restrictions**

Verify allowed access (frontend):

```
kubectl exec -n ckad-8 frontend -- curl -s --connect-timeout 2 http://backend
```

*(Returns NGINX HTML successfully).*

**Deploy the attacker pod and test denied access:**

```
kubectl run attacker --image=nginx:1.25 --labels="tier=hacker" -n ckad-8
```

*Once attacker is running, attempt to reach backend:*

```
kubectl exec -n ckad-8 attacker -- curl -s --connect-timeout 2 http://backend
```

*(Command times out with exit code 28, confirming incoming traffic is blocked).*

# Task 9 – Services and DNS

## Objective

Understand Service discovery and DNS resolution.

---

## Create Namespace

```bash
kubectl create namespace ckad-9
```

---

## Deployment

```bash
kubectl create deployment api \
  --image=nginx:1.25 \
  -n ckad-9

kubectl scale deployment api \
  --replicas=2 \
  -n ckad-9
```

---

## Create ClusterIP Service

```bash
kubectl expose deployment api \
  --port=8080 \
  --target-port=80 \
  -n ckad-9
```

Verify:

```bash
kubectl get svc -n ckad-9
```

---

## DNS Resolution Tests

Launch temporary pod:

```bash
kubectl run dns-test \
  --image=busybox:1.35 \
  --restart=Never \
  --rm -it \
  -n ckad-9 -- sh
```

Inside:

```bash
nslookup api

wget -qO- http://api:8080

wget -qO- http://api.ckad-9:8080

wget -qO- http://api.ckad-9.svc.cluster.local:8080
```

All should succeed.

---

## Convert to NodePort

```bash
kubectl patch svc api \
  -n ckad-9 \
  -p '{"spec":{"type":"NodePort"}}'
```

Verify:

```bash
kubectl get svc api -n ckad-9
```

Expected:

```text
PORT(S)
8080:31xxx/TCP
```

Record the assigned NodePort.

---

# Task 10 – Helm Deploy and Upgrade

## Objective

Deploy, upgrade and rollback a Helm release.

---

## Create Namespace

```bash
kubectl create namespace ckad-10
```

---

## Add Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo update
```

---

## Install Chart

```bash
helm install my-nginx bitnami/nginx \
  --namespace ckad-10 \
  --set replicaCount=2 \
  --set service.type=ClusterIP
```

Verify:

```bash
helm list -n ckad-10

kubectl get deploy -n ckad-10
```

Expected:

```text
READY 2/2
```

---

## Upgrade Release

```bash
helm upgrade my-nginx bitnami/nginx \
  --namespace ckad-10 \
  --set replicaCount=3 \
  --set service.type=ClusterIP
```

Verify:

```bash
kubectl get deploy -n ckad-10
```

Expected:

```text
READY 3/3
```

---

## View History

```bash
helm history my-nginx -n ckad-10
```

Expected:

```text
REVISION STATUS
1        deployed
2        deployed
```

---

## Rollback

```bash
helm rollback my-nginx 1 -n ckad-10
```

Verify:

```bash
kubectl get deploy -n ckad-10
```

Expected:

```text
READY 2/2
```

---

## Key Learnings

* Job vs CronJob
* QoS Classes
* NetworkPolicy Enforcement
* Service Discovery
* Cluster DNS Resolution
* NodePort Services
* Helm Install
* Helm Upgrade
* Helm Rollback
* Helm Revision History
* Production Deployment Lifecycle

```
```

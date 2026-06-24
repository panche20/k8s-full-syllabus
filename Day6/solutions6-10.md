# Task 6 – Jobs and CronJobs

## Objective

* Create a Job to calculate Pi to 500 digits.
* Create a CronJob that runs every 2 minutes.
* Configure Job retry behavior and CronJob concurrency policy.

---

## Create Namespace

```bash
kubectl create namespace ckad-6
```

---

## Create Job

```yaml
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
      restartPolicy: Never

      containers:
      - name: pi
        image: perl:5.34
        command:
        - perl
        - -Mbignum=bpi
        - -wle
        - print bpi(500)
```

Apply:

```bash
kubectl apply -f pi-job.yaml
```

---

## Verify Job

```bash
kubectl get jobs -n ckad-6
kubectl logs job/pi-job -n ckad-6
```

Expected:

```text
3.14159265358979323846...
```

---

## Create CronJob

```yaml
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
          restartPolicy: Never

          containers:
          - name: cleanup
            image: busybox
            command:
            - sh
            - -c
            - echo cleanup ran at $(date)
```

Apply:

```bash
kubectl apply -f cleanup-cronjob.yaml
```

---

## Verify CronJob

```bash
kubectl get cronjobs -n ckad-6
kubectl get jobs -n ckad-6 --watch
```

Inspect logs:

```bash
kubectl logs <job-pod-name> -n ckad-6
```

Expected:

```text
cleanup ran at Tue Jun 24 10:30:00 UTC 2026
```

---

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

## Objective

Restrict access to backend so only frontend can communicate with it.

---

## Create Namespace

```bash
kubectl create namespace ckad-8
```

---

## Deploy Applications

```bash
kubectl run frontend \
  --image=nginx:1.25 \
  --labels=tier=frontend \
  -n ckad-8

kubectl run backend \
  --image=nginx:1.25 \
  --labels=tier=backend \
  -n ckad-8
```

Expose Services:

```bash
kubectl expose pod frontend \
  --port=80 \
  -n ckad-8

kubectl expose pod backend \
  --port=80 \
  -n ckad-8
```

---

## Verify Baseline Connectivity

```bash
kubectl run tester \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  -n ckad-8 -- sh
```

Inside:

```bash
wget -qO- http://backend
```

Expected:

```html
Welcome to nginx!
```

---

## Create NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
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

Apply:

```bash
kubectl apply -f backend-policy.yaml
```

---

## Verify Policy

Create attacker:

```bash
kubectl run attacker \
  --image=busybox:1.35 \
  --labels=tier=hacker \
  -n ckad-8
```

Frontend should succeed:

```bash
kubectl exec frontend -n ckad-8 -- wget -qO- http://backend
```

Attacker should fail:

```bash
kubectl exec attacker -n ckad-8 -- wget -T 3 -qO- http://backend
```

---

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

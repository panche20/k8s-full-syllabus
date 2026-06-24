# Task 11 – Liveness and Startup Probes

## Objective

Understand how Kubernetes uses Startup and Liveness probes to determine container health and restart unhealthy workloads automatically.

---

## Create Namespace

```bash
kubectl create ns ckad-11
```

---

## Create Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
  namespace: ckad-11
spec:
  containers:
  - name: app
    image: busybox

    command:
    - sh
    - -c
    - |
      touch /tmp/healthy
      sleep 30
      rm -f /tmp/healthy
      sleep 3600

    startupProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      failureThreshold: 10
      periodSeconds: 3

    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

Apply:

```bash
kubectl apply -f probe-pod.yaml
```

---

## Verify

Watch pod status:

```bash
kubectl get pod probe-pod -n ckad-11 -w
```

Check restart count:

```bash
kubectl describe pod probe-pod -n ckad-11
```

Expected:

```text
Restart Count: 1+
```

The container becomes unhealthy after deleting `/tmp/healthy`, causing Kubernetes to restart it.

---

# Task 12 – Labels, Selectors and Annotations

## Objective

Practice Kubernetes metadata management.

---

## Create Namespace

```bash
kubectl create ns ckad-12
```

---

## Create Pods

```bash
kubectl run pod-a --image=nginx --labels=env=prod,tier=web -n ckad-12

kubectl run pod-b --image=nginx --labels=env=prod,tier=db -n ckad-12

kubectl run pod-c --image=nginx --labels=env=dev,tier=web -n ckad-12

kubectl run pod-d --image=nginx --labels=env=dev,tier=db -n ckad-12
```

---

## Select env=prod AND tier=web

```bash
kubectl get pods \
  -n ckad-12 \
  -l env=prod,tier=web
```

Expected:

```text
pod-a
```

---

## Set-Based Selector

```bash
kubectl get pods \
  -n ckad-12 \
  -l 'env in (prod,dev)'
```

Expected:

```text
pod-a
pod-b
pod-c
pod-d
```

---

## Add Annotations

```bash
kubectl annotate pod pod-a \
  owner=chetan \
  team=platform \
  -n ckad-12
```

Verify:

```bash
kubectl describe pod pod-a -n ckad-12
```

---

## Custom Output Columns

```bash
kubectl get pods \
  -n ckad-12 \
  -o custom-columns=NAME:.metadata.name,ENV:.metadata.labels.env
```

Expected:

```text
NAME    ENV
pod-a   prod
pod-b   prod
pod-c   dev
pod-d   dev
```

---

# Task 13 – DaemonSet with Node Selector

## Objective

Deploy workloads on specific nodes only.

---

## Create Namespace

```bash
kubectl create ns ckad-13
```

---

## Label Worker Node

Check nodes:

```bash
kubectl get nodes
```

Label one worker:

```bash
kubectl label node worker-node-name monitoring=enabled
```

---

## Create DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitor-agent
  namespace: ckad-13
spec:
  selector:
    matchLabels:
      app: monitor-agent

  template:
    metadata:
      labels:
        app: monitor-agent

    spec:
      nodeSelector:
        monitoring: enabled

      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule

      containers:
      - name: monitor
        image: busybox
        command:
        - sh
        - -c
        - sleep infinity
```

Apply:

```bash
kubectl apply -f daemonset.yaml
```

---

## Verify

```bash
kubectl get pods -n ckad-13 -o wide
```

Expected:

```text
Only scheduled on labeled node(s)
```

---

# Task 14 – StatefulSet with Headless Service

## Objective

Deploy stateful applications with stable identities and storage.

---

## Create Namespace

```bash
kubectl create ns ckad-14
```

---

## Create Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
  namespace: ckad-14
spec:
  clusterIP: None

  selector:
    app: db

  ports:
  - port: 5432
```

Apply:

```bash
kubectl apply -f headless-service.yaml
```

---

## Create StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: ckad-14
spec:
  serviceName: db-headless

  replicas: 3

  selector:
    matchLabels:
      app: db

  template:
    metadata:
      labels:
        app: db

    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine

        env:
        - name: POSTGRES_PASSWORD
          value: test123

        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
  - metadata:
      name: pgdata

    spec:
      accessModes:
      - ReadWriteOnce

      resources:
        requests:
          storage: 256Mi
```

Apply:

```bash
kubectl apply -f statefulset.yaml
```

---

## Verify

Pods:

```bash
kubectl get pods -n ckad-14
```

Expected:

```text
db-0
db-1
db-2
```

PVCs:

```bash
kubectl get pvc -n ckad-14
```

Expected:

```text
pgdata-db-0
pgdata-db-1
pgdata-db-2
```

DNS Test:

```bash
kubectl exec -it db-0 -n ckad-14 -- sh
```

Inside:

```bash
nslookup db-1.db-headless.ckad-14.svc.cluster.local
```

Expected:

```text
Address: <pod-ip>
```

---

# Task 15 – Multi-Container Application with Resource Management

## Objective

Build a production-style application with:

* Main application container
* Sidecar logging container
* Metrics container
* Shared volume
* ConfigMap
* Readiness Probe
* Service

---

## Create Namespace

```bash
kubectl create ns ckad-15
```

---

## Create ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ckad-15
data:
  health.conf: |
    server {
      listen 80;

      location /health {
        return 200 'ok';
      }
    }
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

---

## Create Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-stack
  namespace: ckad-15
spec:
  volumes:
  - name: logs
    emptyDir: {}

  - name: config
    configMap:
      name: nginx-config

  containers:

  - name: app
    image: nginx:1.25

    resources:
      requests:
        cpu: 100m
        memory: 64Mi

      limits:
        cpu: 200m
        memory: 128Mi

    readinessProbe:
      httpGet:
        path: /health
        port: 80

    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

    - name: config
      mountPath: /etc/nginx/conf.d

  - name: sidecar
    image: busybox

    command:
    - sh
    - -c
    - |
      touch /var/log/nginx/access.log
      tail -f /var/log/nginx/access.log

    resources:
      requests:
        cpu: 50m
        memory: 32Mi

    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  - name: metrics
    image: busybox

    command:
    - sh
    - -c
    - |
      while true
      do
        echo metrics_ok
        sleep 10
      done

    resources:
      requests:
        cpu: 50m
        memory: 32Mi
```

Apply:

```bash
kubectl apply -f full-stack.yaml
```

---

## Create Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: full-stack-svc
  namespace: ckad-15
spec:
  selector:
    app: full-stack

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f service.yaml
```

---

## Verify

Check readiness:

```bash
kubectl get pod full-stack -n ckad-15
```

Generate traffic:

```bash
kubectl run curl \
  --image=busybox:1.35 \
  --restart=Never \
  --rm -it \
  -n ckad-15 -- wget -qO- http://full-stack-svc
```

Observe sidecar logs:

```bash
kubectl logs full-stack \
  -c sidecar \
  -n ckad-15 -f
```

Expected:

```text
GET /
GET /
GET /
```

---

# Key Learnings

This mock exam covered:

* Multi-container Pods
* Init Containers
* Deployments & Rollouts
* ConfigMaps & Secrets
* Persistent Volumes & PVCs
* Jobs & CronJobs
* Resource Requests & Limits
* QoS Classes
* Network Policies
* Services & DNS
* Helm
* Liveness & Startup Probes
* Labels & Selectors
* DaemonSets
* StatefulSets
* Headless Services
* Sidecar Pattern
* Readiness Probes
* Kubernetes Troubleshooting

These topics closely align with real CKAD exam objectives and production Kubernetes administration tasks.

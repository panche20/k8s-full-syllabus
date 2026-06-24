# Day 6 – CKAD Full Mock Exam

## Overview

This project contains hands-on solutions for a full CKAD-style mock exam covering:

* Multi-container Pods
* Init Containers
* Deployments & Rolling Updates
* ConfigMaps & Secrets
* Persistent Volumes & Claims
* Jobs & CronJobs
* Resource Management & QoS
* Network Policies
* Services & DNS
* Helm
* Probes
* Labels & Selectors
* DaemonSets
* StatefulSets
* Multi-container Application Design

---

## Task 1 – Multi-container Pod with Shared Volume

### Create Namespace

```bash
kubectl create ns ckad-1
```

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logger
  namespace: ckad-1
spec:
  volumes:
  - name: log-vol
    emptyDir: {}

  containers:
  - name: writer
    image: busybox
    command:
    - sh
    - -c
    - |
      while true
      do
        date >> /logs/app.log
        sleep 3
      done
    volumeMounts:
    - name: log-vol
      mountPath: /logs

  - name: reader
    image: busybox
    command:
    - sh
    - -c
    - |
      touch /logs/app.log
      tail -f /logs/app.log
    volumeMounts:
    - name: log-vol
      mountPath: /logs
```

### Verification

```bash
kubectl logs logger -c reader -n ckad-1 -f
```

Expected:

```text
Tue Jun 24 10:01:01 UTC 2026
Tue Jun 24 10:01:04 UTC 2026
```

---

## Task 2 – Init Container Dependency Check

### Create Namespace

```bash
kubectl create ns ckad-2
```

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  namespace: ckad-2
spec:
  initContainers:
  - name: check-svc
    image: busybox
    command:
    - sh
    - -c
    - |
      until nslookup backend
      do
        echo waiting for backend
        sleep 2
      done

  containers:
  - name: app
    image: nginx:1.25
```

Apply:

```bash
kubectl apply -f pod.yaml
```

### Create Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ckad-2
spec:
  selector:
    app: backend
  ports:
  - port: 80
```

Verification:

```bash
kubectl get pod -n ckad-2
```

Expected:

```text
Running
```

---

## Task 3 – Deployment Rolling Update

### Namespace

```bash
kubectl create ns ckad-3
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: ckad-3
spec:
  replicas: 4

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
      - name: nginx
        image: nginx:1.24

        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
```

Update:

```bash
kubectl set image deployment/web nginx=nginx:1.25 -n ckad-3
```

Annotate:

```bash
kubectl annotate deployment web \
  kubernetes.io/change-cause="image updated to 1.25" \
  -n ckad-3
```

Verification:

```bash
kubectl rollout status deployment/web -n ckad-3
kubectl rollout history deployment/web -n ckad-3
```

---

## Task 4 – ConfigMap & Secret

### Namespace

```bash
kubectl create ns ckad-4
```

### ConfigMap

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=staging \
  --from-literal=LOG_LEVEL=debug \
  -n ckad-4
```

### Secret

```bash
kubectl create secret generic app-secret \
  --from-literal=API_KEY=s3cr3t-k3y \
  -n ckad-4
```

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: ckad-4
spec:
  containers:
  - name: app
    image: busybox

    command:
    - sh
    - -c
    - |
      env
      sleep 3600

    envFrom:
    - configMapRef:
        name: app-config

    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: API_KEY

    volumeMounts:
    - mountPath: /etc/config
      name: config-vol

  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

Verification:

```bash
kubectl exec config-pod -n ckad-4 -- env
kubectl exec config-pod -n ckad-4 -- ls /etc/config
```

---

## Task 5 – Persistent Storage

### Namespace

```bash
kubectl create ns ckad-5
```

### PV + PVC + Pod

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv
spec:
  capacity:
    storage: 500Mi

  accessModes:
  - ReadWriteOnce

  storageClassName: manual

  persistentVolumeReclaimPolicy: Retain

  hostPath:
    path: /tmp/ckad-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pvc
  namespace: ckad-5
spec:
  accessModes:
  - ReadWriteOnce

  storageClassName: manual

  resources:
    requests:
      storage: 200Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
  namespace: ckad-5
spec:
  containers:
  - name: writer
    image: busybox

    command:
    - sh
    - -c
    - |
      echo "ckad exam data" > /data/result.txt
      sleep 3600

    volumeMounts:
    - mountPath: /data
      name: storage

  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: task-pvc
```

Verification:

```bash
kubectl get pvc -n ckad-5
kubectl exec storage-pod -n ckad-5 -- cat /data/result.txt
```

Expected:

```text
ckad exam data
```

# Day 6 — Full CKAD Mock Exam

**Rules — simulate real exam conditions:**

**Task 1 — Multi-container Pod with shared volume (6%)**

Create a Pod named logger in namespace ckad-1.

- Container 1: named writer, image busybox, writes the current date to /logs/app.log every 3 seconds indefinitely
- Container 2: named reader, image busybox, tails /logs/app.log continuously
- Both containers must share the log file via a volume named log-vol

Verify: **kubectl logs logger -c reader -n ckad-1* shows timestamps updating.

**Task 2 — Init container with dependency check (5%)**

Create a Pod named app-with-init in namespace ckad-2.

- Init container: named check-svc, image busybox, loops until a Service named backend exists in the same namespace (use nslookup backend in a loop)
- Main container: named app, image nginx:1.25, runs normally after init completes
- After creating the Pod, create a Service named backend (ClusterIP, port 80) that allows the init to pass

Verify: **Pod reaches Running state.*

**Task 3 — Deployment with rolling update strategy (7%)**

In namespace ckad-3:

- Create a Deployment named web with image nginx:1.24, 4 replicas
- Set rolling update strategy: maxSurge: 2, maxUnavailable: 0
- Add a readiness probe: HTTP GET /, port 80, every 5 seconds
- Update the image to nginx:1.25 and verify rollout completes with zero unavailable pods
- Annotate the rollout with the cause: kubectl rollout history must show "image updated to 1.25"


**Task 4 — ConfigMap and Secret consumption (6%)**

In namespace ckad-4:

- Create a ConfigMap named app-config with keys: APP_ENV=staging, LOG_LEVEL=debug
- Create a Secret named app-secret with key: API_KEY=s3cr3t-k3y
- Create a Pod named config-pod, image busybox, command env then sleep 3600
- Pod must consume APP_ENV and LOG_LEVEL from the ConfigMap as env vars
- Pod must consume API_KEY from the Secret as an env var
- Mount the ConfigMap as files at /etc/config

Verify: **kubectl exec config-pod -n ckad-4* -- env shows all three variables. 
**kubectl exec config-pod -n ckad-4* -- ls /etc/config shows the config files.

**Task 5 — Persistent storage (7%)**

In namespace ckad-5:

- Create a PersistentVolume named task-pv, 500Mi, ReadWriteOnce, hostPath: /tmp/ckad-data, storageClassName manual, reclaimPolicy Retain
- Create a PersistentVolumeClaim named task-pvc requesting 200Mi, storageClassName manual
- Create a Pod named storage-pod, image busybox, writes "ckad exam data" to /data/result.txt then sleeps
- Verify the PVC is Bound and the data is readable


**Task 6 — Jobs and CronJobs (6%)**

In namespace ckad-6:

- Create a Job named pi-job, image perl:5.34, that computes pi to 500 digits: perl -Mbignum=bpi -wle 'print bpi(500)'
- Set backoffLimit: 2, completions: 1
- Create a CronJob named cleanup, image busybox, schedule every 2 minutes, command echo cleanup ran at $(date), concurrencyPolicy: Forbid

Verify: Job completes successfully. CronJob creates a Job on schedule.

**Task 7 — Resource limits and QoS (5%)**

In namespace ckad-7:

- Create a Pod named guaranteed-pod, image nginx:1.25, with requests and limits both set to cpu: 200m, memory: 128Mi — verify its QoS class is  Guaranteed
- Create a Pod named burstable-pod, image nginx:1.25, with only requests set: cpu: 100m, memory: 64Mi — verify QoS is Burstable
- Create a Pod named besteffort-pod, image nginx:1.25, with no resources set — verify QoS is BestEffort


**Task 8 — NetworkPolicy (8%)**

In namespace ckad-8:

- Deploy two Pods: frontend (label tier: frontend, image nginx:1.25) and backend (label tier: backend, image nginx:1.25)
- Expose both with ClusterIP Services on port 80
- Verify frontend can reach backend (baseline)
- Apply a NetworkPolicy that: allows ingress to backend ONLY from pods with label tier: frontend, denies all other ingress to backend
- Verify frontend can still reach backend, but a third pod attacker (label tier: hacker) cannot


**Task 9 — Service types and DNS (6%)**

In namespace ckad-9:

- Create a Deployment named api, image nginx:1.25, 2 replicas
- Expose it as ClusterIP on port 8080 targeting port 80
- From a temporary busybox pod, resolve the Service DNS and curl it using the short name, the namespace-qualified name, and the fully qualified name
- Change the Service type to NodePort and note the assigned port


**Task 10 — Helm deploy and override (7%)**

Using Helm:

- Add the bitnami repo
- Install bitnami/nginx chart with release name my-nginx in namespace ckad-10
- Override: replicaCount=2, service.type=ClusterIP
- Verify the release is deployed with 2 replicas
- Upgrade the release to set replicaCount=3
- Check rollout history and rollback to revision 1
- Verify replica count is back to 2


**Task 11 — Liveness and startup probes (6%)**
In namespace ckad-11:

- Create a Pod named probe-pod, image busybox
- Command: creates /tmp/healthy on start, then deletes it after 30 seconds, then sleeps forever
- Configure a liveness probe: exec command cat /tmp/healthy, initialDelaySeconds: 5, periodSeconds: 5, failureThreshold: 3
- Configure a startup probe: exec command cat /tmp/healthy, failureThreshold: 10, periodSeconds: 3
- Watch the pod get killed and restarted after the file is deleted


**Task 12 — Labels, selectors and annotations (5%)**
In namespace ckad-12:

- Create 4 Pods: pod-a (labels: env=prod, tier=web), pod-b (labels: env=prod, tier=db), pod-c (labels: env=dev, tier=web), pod-d (labels: env=dev, tier=db)
- List only pods where env=prod AND tier=web
- List pods where env=prod OR env=dev using set-based selector
- Annotate pod-a with owner=chetan and team=platform
- Use kubectl get with a custom output column showing pod name and the env label value


**Task 13 — DaemonSet with node selector (6%)**
In namespace ckad-13:

- Create a DaemonSet named monitor-agent, image busybox, command sleep infinity
- Add a toleration so it runs on control-plane nodes
- Label one of your worker nodes with monitoring=enabled
- Add a nodeSelector so the DaemonSet ONLY runs on nodes with that label
- Verify it runs on the labeled node only


**Task 14 — StatefulSet with headless Service (8%)**
In namespace ckad-14:

- Create a headless Service named db-headless, selector app=db, port 5432
- Create a StatefulSet named db, image postgres:15-alpine, 3 replicas, env POSTGRES_PASSWORD=test123
- Add a volumeClaimTemplate named pgdata, 256Mi, ReadWriteOnce
- Verify pods are named db-0, db-1, db-2
- Verify each has its own PVC: pgdata-db-0, pgdata-db-1, pgdata-db-2
- Exec into db-0 and resolve db-1.db-headless.ckad-14.svc.cluster.local


**Task 15 — Multi-container patterns + resource management (12%)**
In namespace ckad-15, deploy a complete application:

**Pod named full-stack with 3 containers:*

- app: image nginx:1.25, requests cpu:100m memory:64Mi, limits cpu:200m memory:128Mi
- sidecar: image busybox, tails /var/log/nginx/access.log from a shared volume, requests cpu:50m memory:32Mi
- metrics: image busybox, runs while true; do echo metrics_ok; sleep 10; done, requests cpu:50m memory:32Mi


- Create a ConfigMap nginx-config with a custom nginx.conf that adds a /health endpoint returning 200
- Mount it into the app container at /etc/nginx/conf.d/health.conf
- Add a readiness probe on the app container hitting /health port 80
- Create a Service full-stack-svc exposing port 80
- Verify kubectl logs full-stack -c sidecar updates when you curl the service


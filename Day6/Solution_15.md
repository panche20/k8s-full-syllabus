# CKAD Task 15 — Multi-container Patterns + Resource Management

Let's complete this lab step by step, using **imperative commands where possible and YAML where necessary**. I'll explain the important CKAD concepts as we go.

## 1. Understand the architecture

<img width="789" height="529" alt="image" src="https://github.com/user-attachments/assets/067c5908-4546-4133-a7f4-7c86e841ea23" />

You will create one Pod containing three containers:

<img width="521" height="154" alt="image" src="https://github.com/user-attachments/assets/023e91a7-8bfd-47b7-99ca-cc15b406a7bb" />

All three containers share the same Pod network namespace, meaning they share the same IP address and can communicate over localhost.

The app and sidecar containers also share a volume for access logs.

## 2. Create the namespace

```
kubectl create namespace ckad-15

# Set it as your current namespace:
kubectl config set-context --current --namespace=ckad-15

# Verify:
kubectl get ns ckad-15
```

## 3. Create the custom NGINX ConfigMap

We need a custom NGINX configuration that defines /health returning HTTP 200.

**Create nginx-config.yaml:**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ckad-15
data:
  health.conf: |
    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
```

```
kubectl apply -f nginx-config.yaml

# Verify:
kubectl get configmap nginx-config
kubectl describe configmap nginx-config
```

**Important**: 

The ConfigMap configuration contains its own server block. We will remove the default NGINX configuration when starting the app container to avoid conflicting server definitions.

## 4. Create the complete Pod manifest

**Create full-stack.yaml.**

```
apiVersion: v1
kind: Pod
metadata:
  name: full-stack
  namespace: ckad-15
spec:
  containers:
    # Main application container
    - name: app
      image: nginx:1.25
      command:
        - /bin/sh
        - -c
        - rm -f /etc/nginx/conf.d/default.conf &&
          nginx -g 'daemon off;'
      ports:
        - containerPort: 80

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
        initialDelaySeconds: 5
        periodSeconds: 5

      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/health.conf
          subPath: health.conf
        - name: nginx-logs
          mountPath: /var/log/nginx

    # Sidecar container
    - name: sidecar
      image: busybox
      command:
        - /bin/sh
        - -c
        - tail -F /var/log/nginx/access.log

      resources:
        requests:
          cpu: 50m
          memory: 32Mi

      volumeMounts:
        - name: nginx-logs
          mountPath: /var/log/nginx

    # Metrics container
    - name: metrics
      image: busybox
      command:
        - /bin/sh
        - -c
        - while true; do echo metrics_ok; sleep 10; done

      resources:
        requests:
          cpu: 50m
          memory: 32Mi

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config

    - name: nginx-logs
      emptyDir: {}
```

**Apply:**

```
kubectl apply -f full-stack.yaml
```

## 5. Create the Service

**Create full-stack-svc.yaml.**

```
apiVersion: v1
kind: Service
metadata:
  name: full-stack-svc
  namespace: ckad-15
spec:
  selector:
    # Matches Pod labels
    app: full-stack
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

**Correction required**: 

The Pod manifest above currently has no app: full-stack label. Add this under its metadata:
metadata:

```
  name: full-stack
  namespace: ckad-15
  labels:
    app: full-stack
```

**Now apply the Service:**

```
kubectl apply -f full-stack-svc.yaml
```

## 6. Verify the Pod and containers

```
kubectl get pod full-stack

# Expected:
NAME         READY   STATUS    RESTARTS
full-stack   3/3     Running   0

# Check all three containers:
kubectl get pod full-stack -o jsonpath='{.spec.containers[*].name}'

# Expected:
app sidecar metrics

# Check resource configuration:
kubectl describe pod full-stack
```

**Look for:**

- Container app: CPU request 100m, limit 200m.
- Container app: memory request 64Mi, limit 128Mi.
- Container sidecar: CPU request 50m, memory request 32Mi.
- Container metrics: CPU request 50m, memory request 32Mi.

## 7. Verify the readiness probe

```
kubectl describe pod full-stack

# You should see:
Readiness: http-get http://:80/health

# Check Pod readiness:
kubectl get pod full-stack

# The Pod should show 3/3 READY.

# Test the health endpoint directly:
kubectl exec full-stack -c app -- curl localhost/health

# Expected:
healthy

# Note: curl may not be installed in the NGINX container. If so, use the Service test below instead.
```

## 8. Verify the Service

```
kubectl get svc full-stack-svc

# Expected:
NAME             TYPE        CLUSTER-IP      PORT(S)
full-stack-svc   ClusterIP  10.x.x.x        80/TCP

# Check endpoints:
kubectl get endpoints full-stack-svc

Expected: the IP address of your Pod on port 80.
```

## 9. Verify sidecar logs update when you curl the Service

This is the most important part of the lab.

**Terminal 1: Watch sidecar logs**

```
kubectl logs full-stack -c sidecar -f
```
Initially, it might show nothing.

**Terminal 2: Generate HTTP traffic**

Run a temporary BusyBox Pod:

```
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  -n ckad-15 \
  --command -- sh -c \
  "while true; do curl -s http://full-stack-svc/; sleep 2; done"
```

Or simply execute a curl request from an existing Pod that has curl installed:

```
curl http://full-stack-svc/
```

If you want to test from your workstation, use port-forward:

```
kubectl port-forward svc/full-stack-svc 8080:80
```

**Then:**
curl http://localhost:8080/


**Expected sidecar output**

You should see NGINX access logs similar to:

10.244.0.1 - - [10/Nov/2026:13:30:00 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.0"
10.244.0.1 - - [10/Nov/2026:13:30:02 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.0"
10.244.0.1 - - [10/Nov/2026:13:30:04 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.0"


**This proves that:**

- NGINX is receiving HTTP requests.
- NGINX writes access logs to /var/log/nginx/access.log.
- The shared emptyDir volume makes those logs accessible to the sidecar.
- The sidecar reads the logs using tail -F.
- Kubernetes exposes the sidecar's stdout through kubectl logs.

## 10. Verify the metrics container

```
kubectl logs full-stack -c metrics

# Expected:
metrics_ok
metrics_ok
metrics_ok

# Follow the logs:
kubectl logs full-stack -c metrics -f
```

## 11. CKAD concepts tested in this task

Concept	What you need to understand
Multi-container Pod	Multiple containers share Pod network namespace
Sidecar pattern	Supporting container handles logging/monitoring
Shared volume	emptyDir shared between containers
ConfigMap	Externalized application configuration
subPath	Mount a single ConfigMap file
Readiness probe	Determines whether Pod can receive traffic
Resource requests	Scheduling requirements
Resource limits	Maximum CPU/memory usage
Service	Stable networking for application Pods
Container-specific logs	kubectl logs POD -c CONTAINER

**Important interview distinction**

Readiness vs Liveness vs Startup probes

- Readiness: Should this Pod receive traffic?
- Liveness: Should Kubernetes restart this container?
- Startup: Has this application completed startup?
In this task, readiness is appropriate because NGINX should receive traffic only after its /health endpoint responds successfully.

## 12. Final cleanup

```
kubectl delete namespace ckad-15
```

This removes all resources created in the namespace.

**CKAD exam tip:**

This task is primarily testing your ability to write a multi-container Pod YAML quickly. Practice creating the Pod, ConfigMap, shared volume, resource requests, and probes without referring to documentation.

One useful improvement for your lab practice is to try recreating this entire task in 10–12 minutes using only imperative commands and kubectl edit/kubectl patch where appropriate, then validate everything with kubectl describe, kubectl logs, and kubectl get endpoints.

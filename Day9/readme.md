# Day 9 — etcd Backup, Certificate Management & API Server Hardening

## Part 1: Why etcd is Everything

etcd holds 100% of your cluster state. Every object — pods, deployments, secrets, RBAC rules, everything — lives as a key-value entry in etcd. 
If etcd is lost and you have no backup, your cluster is gone. The workloads keep running on nodes (kubelet doesn't need etcd) but you lose all control — you can't schedule, scale, or change anything.

```
Lost etcd + no backup = rebuild cluster from scratch
Lost etcd + backup    = restore in minutes
```

This is why etcd backup is a non-negotiable operational requirement and a guaranteed CKA exam task.

## Part 2: etcd Backup with etcdctl

**Understanding etcd on kubeadm clusters**

```
# etcd runs as a static pod
kubectl get pod -n kube-system -l component=etcd

# Get the etcd pod name on kind
ETCD_POD=$(kubectl get pod -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')
echo $ETCD_POD

# Inspect etcd configuration — find cert paths and endpoints
kubectl describe pod $ETCD_POD -n kube-system | grep -A 5 "Command"
```

Three pieces of information you always need for etcdctl:

--endpoints — where etcd listens (usually https://127.0.0.1:2379)
--cacert — CA certificate path
--cert — server certificate path
--key — server key path

All four are visible in the etcd static pod manifest.

**Taking a snapshot**

```
# On your kind cluster — exec into control plane node first
docker exec -it k8s-mastery-control-plane bash

# Set up etcdctl environment (inside the node)
export ETCDCTL_API=3    # always use API version 3

# Take the snapshot
etcdctl snapshot save /tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot is healthy
etcdctl snapshot status /tmp/etcd-backup-*.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# Output shows:
# HASH | REVISION | TOTAL KEYS | TOTAL SIZE
```

**Copy backup off the node**

```
# Exit the control plane node
exit

# Copy backup to your local machine
docker cp k8s-mastery-control-plane:/tmp/etcd-backup-*.db ./etcd-backup.db

# In production — ship to S3
aws s3 cp etcd-backup.db s3://your-backup-bucket/etcd/$(date +%Y/%m/%d)/
```

## Part 3: etcd Restore — The CKA Money Task
Restore is harder than backup. The exact sequence matters. Get the order wrong and the cluster won't come back.

**Restore sequence**

```
1. Stop the API server (so nothing writes to etcd during restore)
2. Restore snapshot to a new data directory
3. Update etcd static pod manifest to point to new data directory
4. kubelet detects manifest change → restarts etcd
5. API server comes back, reads restored state
```

```
# On the control plane node
docker exec -it k8s-mastery-control-plane bash

# Step 1: Stop the API server by moving its manifest out
# kubelet watches /etc/kubernetes/manifests/ — removing the file stops the pod
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak

# Wait for API server pod to stop
# (watch with: crictl pods | grep apiserver)
sleep 10

# Step 2: Restore snapshot to new data directory
etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 3: Update etcd manifest to use the restored data directory
# Edit /etc/kubernetes/manifests/etcd.yaml
# Find the --data-dir flag and the hostPath volume
# Change: /var/lib/etcd → /var/lib/etcd-restored

sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' \
  /etc/kubernetes/manifests/etcd.yaml

# Step 4: Bring API server back
mv /tmp/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml

# Step 5: Wait for everything to come back
# Watch from outside the node:
exit
kubectl get pods -n kube-system -w
# Takes 60-90 seconds for all components to restart
```

**Verify restore worked**

```
# Check etcd health
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check cluster is back
kubectl get nodes
kubectl get pods -A
```

**CKA exam tip:** 

The exam gives you a broken or empty etcd and asks you to restore from a provided snapshot file. 
The path to certs and the snapshot location are always given. The most common failure point is forgetting to update the hostPath volume in the etcd manifest — not just the --data-dir flag. Both must point to the new directory.

## Part 4: Certificate Management

kubeadm clusters use a PKI with a 1-year expiry for component certs (10 years for the CA). Know how to check, renew, and rotate.

**Check certificate expiry**

```
# Check all certificates at once
docker exec k8s-mastery-control-plane \
  kubeadm certs check-expiration

# Output shows each cert, its expiry date, and the CA that signed it:
# CERTIFICATE                EXPIRES               RESIDUAL TIME   CA
# admin.conf                 Nov 15, 2025 10:00 UTC 364d           ca
# apiserver                  Nov 15, 2025 10:00 UTC 364d           ca
# apiserver-etcd-client      Nov 15, 2025 10:00 UTC 364d           etcd-ca
# ...

# Check a specific cert manually
docker exec k8s-mastery-control-plane \
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -dates -subject
```

**Renew all certificates**

```
# Renew everything at once — run on control plane node
docker exec k8s-mastery-control-plane kubeadm certs renew all

# Renew a specific cert
docker exec k8s-mastery-control-plane kubeadm certs renew apiserver

# After renewal — MUST restart control plane components
# Static pods read certs on startup — they won't pick up new certs automatically
docker exec k8s-mastery-control-plane bash -c \
  "mv /etc/kubernetes/manifests/*.yaml /tmp/ && sleep 5 && mv /tmp/*.yaml /etc/kubernetes/manifests/"

# Update kubeconfig — it has an embedded cert that also got renewed
docker exec k8s-mastery-control-plane \
  kubeadm kubeconfig user --client-name=admin > ~/.kube/config
```

**Certificate locations — know this map cold**

```
/etc/kubernetes/pki/
├── ca.crt + ca.key                    — cluster CA (signs all component certs)
├── apiserver.crt + apiserver.key      — API server serving cert
├── apiserver-kubelet-client.crt+key   — API server → kubelet auth
├── apiserver-etcd-client.crt+key      — API server → etcd auth
├── front-proxy-ca.crt + key           — aggregation layer CA
├── front-proxy-client.crt + key       — aggregation layer client
├── sa.pub + sa.key                    — ServiceAccount token signing
└── etcd/
    ├── ca.crt + ca.key                — etcd CA
    ├── server.crt + server.key        — etcd serving cert
    ├── peer.crt + peer.key            — etcd peer-to-peer
    └── healthcheck-client.crt+key     — liveness probe auth
```

**Manual certificate creation (CKS depth)**

```
# Generate a new cert for a component manually
# Example: new kubelet client cert

# Step 1: Generate key
openssl genrsa -out kubelet-client.key 2048

# Step 2: CSR — O=system:nodes is critical (kubelet group)
openssl req -new \
  -key kubelet-client.key \
  -out kubelet-client.csr \
  -subj "/CN=system:node:worker-1/O=system:nodes"

# Step 3: Sign with cluster CA
openssl x509 -req \
  -in kubelet-client.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kubelet-client.crt \
  -days 365

# Verify the signed cert
openssl x509 -in kubelet-client.crt -noout -text | grep -E "Subject:|Issuer:|Not After"
```

## Part 5: API Server Hardening

The API server is the cluster's front door. Every request goes through it. Hardening it is pure CKS material but CKA tests the flags.

**Admission controllers — the API server's gatekeepers**

Every request to the API server goes through:

```
Authentication → Authorization (RBAC) → Admission Controllers → etcd
```

*Admission controllers are plugins that validate or mutate requests before they're persisted. Some critical ones:*

<img width="906" height="402" alt="image" src="https://github.com/user-attachments/assets/0568d71a-2d84-4273-98dd-963e175161a9" />


```
# See which admission controllers are enabled
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission

# Enable an additional admission controller
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Find: --enable-admission-plugins=...
# Add your controller to the comma-separated list
```

**Key API server security flags**

```
# In /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver

    # Disable anonymous auth — no unauthenticated access
    - --anonymous-auth=false

    # Only allow specific auth modes
    - --authorization-mode=Node,RBAC

    # Audit logging — log all requests
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml

    # Encryption at rest for secrets
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml

    # Disable profiling endpoint (info leak)
    - --profiling=false

    # Restrict kubelet API access
    - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

**Audit policy — control what gets logged**

```
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all secret access at RequestResponse level (full body)
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log pod creation at Request level (no response body)
- level: Request
  verbs: ["create","update","patch","delete"]
  resources:
  - group: ""
    resources: ["pods"]
  - group: "apps"
    resources: ["deployments"]

# Don't log read-only requests to configmaps
- level: None
  verbs: ["get","list","watch"]
  resources:
  - group: ""
    resources: ["configmaps"]

# Don't log health checks (too noisy)
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]

# Default — log metadata only for everything else
- level: Metadata
```

## Part 6: Hands-On Exercises

**Exercise 1: Full backup → disaster → restore drill**

```
# Step 1: Create some resources to prove restore works
kubectl create namespace before-backup
kubectl create deployment canary --image=nginx:1.25 --replicas=3 -n before-backup
kubectl create configmap canary-config --from-literal=key=value -n before-backup

# Step 2: Take backup
docker exec k8s-mastery-control-plane bash -c "
  export ETCDCTL_API=3
  etcdctl snapshot save /tmp/pre-disaster.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  etcdctl snapshot status /tmp/pre-disaster.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --write-out=table
"

# Step 3: Simulate disaster — delete namespace
kubectl delete namespace before-backup
kubectl get ns before-backup   # gone

# Step 4: Restore (follow Part 3 steps exactly)
# After restore:
kubectl get namespace before-backup          # back
kubectl get deployment -n before-backup      # canary back with 3 replicas
kubectl get configmap -n before-backup       # canary-config back
```

**Exercise 2: Check and renew certificates**

```
# Check all cert expiry dates
docker exec k8s-mastery-control-plane kubeadm certs check-expiration

# Check a specific cert in detail
docker exec k8s-mastery-control-plane \
  openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -E "Not Before|Not After|Subject|DNS|IP"

# Renew all (safe to run even if certs aren't expiring soon)
docker exec k8s-mastery-control-plane kubeadm certs renew all
docker exec k8s-mastery-control-plane kubeadm certs check-expiration
# Dates should now be 1 year from today
```

**Exercise 3: Enable audit logging**

```
# Create audit policy on control plane node
docker exec k8s-mastery-control-plane bash -c "
cat <<EOF > /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ''
    resources: ['secrets']
- level: Metadata
EOF
"

# Add audit flags to API server manifest
docker exec k8s-mastery-control-plane bash -c "
  sed -i '/- kube-apiserver/a\\    - --audit-log-path=/var/log/kubernetes/audit.log\n    - --audit-log-maxage=30\n    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
"

# Wait for API server to restart (60-90 seconds)
kubectl get pods -n kube-system -w

# Trigger a secret access
kubectl get secret -n kube-system

# Check audit log
docker exec k8s-mastery-control-plane \
  tail -20 /var/log/kubernetes/audit.log | jq .
```

**Exercise 4: Inspect what's inside etcd directly**

```
# Read a specific key from etcd
docker exec k8s-mastery-control-plane bash -c "
  export ETCDCTL_API=3
  etcdctl get /registry/namespaces/default \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --print-value-only | strings | head -30
"

# List all keys (see everything K8s stores)
docker exec k8s-mastery-control-plane bash -c "
  export ETCDCTL_API=3
  etcdctl get / --prefix --keys-only \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    | head -40
"
# See: /registry/pods/, /registry/secrets/, /registry/deployments/ etc.
```

## Part 7: Interview Questions — Day 10

**Q1: Walk me through restoring etcd from a snapshot. What exact steps do you take?**

Stop the API server by moving its static pod manifest out of /etc/kubernetes/manifests/. Run etcdctl snapshot restore pointing to a new data directory. Update the etcd static pod manifest — both --data-dir flag AND the hostPath volume — to point to the new directory. Move the API server manifest back. Wait 60-90 seconds for all components to restart. Verify with kubectl get nodes and etcdctl endpoint health.

**Q2: etcd backup completed but snapshot status shows 0 keys. What happened?**

The backup ran against the wrong endpoint or with the wrong certs, so it connected to nothing (or a new empty etcd). Always verify immediately after backup with etcdctl snapshot status --write-out=table and check that TOTAL KEYS is non-zero — a healthy cluster should have thousands of keys. Also check REVISION is greater than 0.

**Q3: Certificates expired overnight and the cluster is down. How do you recover?**

If kubeadm is available: exec onto the control plane node, run kubeadm certs renew all, then restart all control plane components by briefly moving their manifests out of /etc/kubernetes/manifests/ and back. Update your local kubeconfig since the admin cert was also renewed. If the API server cert expired: kubelet can still read the static pod manifest and restart components, but you may need to use the local admin.conf on the control plane node directly since your remote kubeconfig has the old cert.

**Q4: What is the NodeRestriction admission controller and why does it matter?**

It limits kubelet's API permissions — a kubelet can only modify its own Node object and pods scheduled to its own node. Without it, a compromised kubelet on worker-1 could modify worker-2's node object or read secrets for pods running elsewhere. It's a critical blast-radius limiter for node compromise scenarios, and mandatory for CKS.

**Q5: What's the difference between authentication and admission control?**

Authentication establishes who is making the request (valid cert, valid token). Authorization (RBAC) decides if that identity is allowed to perform the action. Admission control runs after both and can mutate or reject the request based on policy — for example, rejecting a pod with no resource limits even if the user has create permissions. Admission controllers act on the content of the request, not just the identity.

**Q6: How do you encrypt secrets at rest in etcd?**

Create an EncryptionConfiguration manifest with an AES-CBC or AES-GCM provider and a base64-encoded 32-byte key. Add --encryption-provider-config flag to the API server static pod manifest pointing to that file. Restart the API server. Existing secrets are NOT automatically re-encrypted — run kubectl get secrets -A -o json | kubectl replace -f - to force a re-write of all secrets through the new encryption path. Verify by reading the raw etcd value and confirming it's no longer plaintext base64

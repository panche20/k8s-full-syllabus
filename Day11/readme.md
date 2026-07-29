# Day 11 — Full CKA Mock Exam

*Week 2 boss battle. This is as close to the real CKA as possible — same task types, same weight distribution, same time pressure. The real exam is 2 hours, 17 tasks, passing score 66%.*

## ⚡ Exam Setup — Do This First

```
# Set these before the clock starts — saves minutes over 2 hours
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Verify your cluster
kubectl get nodes
kubectl config current-context

# Tip: use kubectl explain constantly
# kubectl explain pod.spec.tolerations
# kubectl explain deployment.spec.strategy
```

## Task 1 — kubeadm cluster info (4%)

*List all nodes in the cluster and output to a file.*

- Write the names of all nodes and their roles to /tmp/nodes.txt
- Format: one node per line, <nodename> <role>
- Also write the Kubernetes server version to /tmp/k8s-version.txt

**Solution**

```
# 1. Output <nodename> <role> for all nodes to /tmp/nodes.txt
kubectl get nodes -o custom-columns='NAME:.metadata.name,ROLE:.metadata.labels.node-role\.kubernetes\.io/control-plane' --no-headers | \
awk '{
  role = ($2 != "<none>" && $2 != "") ? "control-plane" : "worker";
  print $1, role
}' > /tmp/nodes.txt

# 2. Output the Kubernetes server version to /tmp/k8s-version.txt
kubectl version 2>/dev/null | grep -i "Server Version" | awk '{print $3}' > /tmp/k8s-version.txt || \
kubectl version -o json | jq -r '.serverVersion.gitVersion' > /tmp/k8s-version.txt
```

## Task 2 — Static pod (5%)

*Create a static pod on the control plane node.*

- Name: static-web
- Image: nginx:1.25
- Namespace: default
- The pod must survive a kubectl delete pod attempt — prove it recreates

**Solution**

**Step 1: Access the Target Node & Find the Manifest Path**

- SSH into the control plane node (or node where you want the static pod to run) and inspect the kubelet configuration to verify its static pod directory (by        default /etc/kubernetes/manifests):

```
# SSH into the node
ssh control-plane-node

# Identify the staticPodPath configured in kubelet (default: /etc/kubernetes/manifests)
grep -i "staticpodpath" /var/lib/kubelet/config.yaml
```

**Step 2: Create the Static Pod Manifest FileCreate the pod manifest inside the static pod directory (/etc/kubernetes/manifests/static-web.yaml):**

```
cat <<EOF | sudo tee /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  namespace: default
spec:
  containers:
  - name: web
    image: nginx:1.25
EOF
```

*The kubelet automatically detects the new YAML file and starts the pod. When created as a static pod, kubelet automatically appends the node's hostname as a suffix to the pod name (e.g., static-web-<nodename>).*

**Step 3: Verify Creation & Prove Automatic Recreation**

*1. Verify the static pod is running*

```
kubectl get pods -n default
```

*2. Test deletion to prove it recreatesStatic pods are managed by the local kubelet on the node, not by the Kubernetes API Server. Deleting it via kubectl only removes the API Server "mirror pod", causing kubelet to immediately recreate it.*

```
# Delete the pod via API server
kubectl delete pod static-web-<nodename> -n default
```

*3. Confirm recreation*

Wait 5 seconds and inspect the pod again:

```
kubectl get pods -n default
```

## Task 3 — Cluster upgrade (8%)

*Your kind cluster is already at its current version. Simulate the upgrade knowledge:*

- Show the full upgrade plan with kubeadm upgrade plan
- Write the output to /tmp/upgrade-plan.txt
- Write the exact commands needed to upgrade a worker node (in order) to /tmp/upgrade-worker-steps.txt — this is a written answer, not execution

**Solution**

**Step 1: Generate kubeadm upgrade plan Output**

*Run the kubeadm upgrade plan command and save the output directly to /tmp/upgrade-plan.txt:*

```
Bash
sudo kubeadm upgrade plan > /tmp/upgrade-plan.txt
```

*(If kubeadm requires cluster access or flags in your specific environment, run sudo kubeadm upgrade plan --config=... or similar, ensuring stdout goes to /tmp/upgrade-plan.txt).*

**Step 2: Write Worker Node Upgrade Steps to /tmp/upgrade-worker-steps.txt**

*The standard Kubernetes (kubeadm) procedure to upgrade a worker node involves 6 core steps in strict order:*

- **Drain the node** (from the control plane / management machine)

- **Upgrade** kubeadm package

- **Run kubeadm node upgrade** configuration update

- Upgrade **kubelet and kubectl** packages

- Restart **kubelet** service

- Uncordon the node (from the control plane / management machine)

*Write these exact steps directly into /tmp/upgrade-worker-steps.txt:*

```
Bash
cat <<'EOF' > /tmp/upgrade-worker-steps.txt
# 1. On Control Plane: Drain worker node before maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. On Worker Node: Upgrade kubeadm tool
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=<target-version>

# 3. On Worker Node: Apply node upgrade configuration
sudo kubeadm upgrade node

# 4. On Worker Node: Upgrade kubelet and kubectl packages
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=<target-version> kubectl=<target-version>

# 5. On Worker Node: Reload system daemon and restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. On Control Plane: Uncordon worker node to resume scheduling
kubectl uncordon <node-name>
EOF
```

**Verification**

*Check that both target files are populated properly:*

```
Bash
# Verify upgrade plan file exists and isn't empty
head -n 10 /tmp/upgrade-plan.txt

# Verify worker steps file formatting
cat /tmp/upgrade-worker-steps.txt
```

## Task 4 — etcd backup (8%)

- Take a snapshot of etcd to /tmp/etcd-snapshot.db
- Verify the snapshot is valid — write the snapshot status (hash, revision, total keys, size) to /tmp/etcd-status.txt
- The etcd certs are at their default kubeadm locations

**Solution**

*To take the etcd snapshot and verify its status using the default kubeadm certificate paths, run the following commands on the control plane node:*

**Step 1: Take the etcd SnapshotRun etcdctl with API version 3 and the default kubeadm certificates located in /etc/kubernetes/pki/etcd/:**

```
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-snapshot.db
```
  
**Step 2: Verify Snapshot and Write Status to FileOutput the snapshot status details (hash, revision, total keys, size) to /tmp/etcd-status.txt:**

```
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-snapshot.db --write-out=table > /tmp/etcd-status.txt
```

*(Note: If your environment uses etcdutl instead of etcdctl for snapshot status verification, run etcdutl snapshot status /tmp/etcd-snapshot.db --write-out=table > /tmp/etcd-status.txt).*

**Verification**

*Inspect /tmp/etcd-status.txt to confirm it contains the snapshot information*:

```
cat /tmp/etcd-status.txt
```

*Expected Output Format:*

```
Plaintext

+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 3c271b4a |    12480 |       1042 |     3.2 MB |
+----------+----------+------------+------------+
```

## Task 5 — RBAC: ServiceAccount with permissions (6%)

*In namespace cka-5:*

- Create a ServiceAccount named deploy-manager
- Create a Role named deployment-admin that allows all verbs on deployments and replicasets in apps apiGroup
- Bind the SA to the role
- Verify: kubectl auth can-i delete deployments --as=system:serviceaccount:cka-5:deploy-manager -n cka-5 returns yes
- Verify: kubectl auth can-i delete secrets --as=system:serviceaccount:cka-5:deploy-manager -n cka-5 returns no

**Solution**

## Step 1: Create Namespace, ServiceAccount, Role, and RoleBinding

*You can execute these imperatively using kubectl or apply them all at once:*

```
# 1. Create the namespace (if it doesn't already exist)
kubectl create namespace cka-5 --dry-run=client -o yaml | kubectl apply -f -

# 2. Create the ServiceAccount
kubectl create serviceaccount deploy-manager -n cka-5

# 3. Create the Role with all verbs on deployments and replicasets in apps group
kubectl create role deployment-admin \
  --verb="*" \
  --resource=deployments,replicasets \
  --namespace=cka-5

# 4. Bind the ServiceAccount to the Role
kubectl create rolebinding deploy-manager-binding \
  --role=deployment-admin \
  --serviceaccount=cka-5:deploy-manager \
  --namespace=cka-5
```

## Step 2: Run Verification Checks

*Test the permissions using kubectl auth can-i:*

**Test 1: Deployment Deletion (Should return yes)**

```
Bash
kubectl auth can-i delete deployments --as=system:serviceaccount:cka-5:deploy-manager -n cka-5
```

**Expected Output:**

```
Plaintext
yes
```

**Test 2: Secret Deletion (Should return no)**

```
Bash
kubectl auth can-i delete secrets --as=system:serviceaccount:cka-5:deploy-manager -n cka-5
```

**Expected Output:**

```
Plaintext
no
```

## Task 6 — User certificate auth (6%)

- Generate a private key and CSR for user jane in group developers
- Submit a CertificateSigningRequest to Kubernetes
- Approve it
- Extract the signed certificate to /tmp/jane.crt
- Create a kubeconfig context jane-context using the cert
- Grant jane view permissions on namespace cka-6
- Verify jane can list pods but not delete them in cka-6

**Solution**

Here is the step-by-step solution for Task 6.

## Step 1: Create Namespace cka-6

*Ensure the target namespace exists:*

```
Bash
kubectl create namespace cka-6 --dry-run=client -o yaml | kubectl apply -f -
```

## Step 2: Generate Private Key and CSR File

*Generate a 2048-bit RSA private key and create a Certificate Signing Request (CSR) specifying user jane and group developers:*

```
Bash
# 1. Generate private key
openssl genrsa -out /tmp/jane.key 2048

# 2. Generate CSR with CN=jane and O=developers
openssl req -new -key /tmp/jane.key -out /tmp/jane.csr -subj "/CN=jane/O=developers"
```

## Step 3: Create and Submit the Kubernetes CSR Object

*Base64 encode the CSR file content and apply the CertificateSigningRequest manifest:*

```
Bash
# Get base64 representation of the CSR (single-line)
CSR_BASE64=$(cat /tmp/jane.csr | base64 | tr -d '\n')

# Submit CSR to Kubernetes API server
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  request: ${CSR_BASE64}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
```

## Step 4: Approve the CSR and Extract /tmp/jane.crt

*Approve the request and extract the signed certificate into /tmp/jane.crt:*

```
Bash
# Approve the CSR
kubectl certificate approve jane-csr

# Extract the client certificate from status.certificate
kubectl get csr jane-csr -o jsonpath='{.status.certificate}' | base64 -d > /tmp/jane.crt
```

## Step 5: Configure Credentials and Context in Kubeconfig

*Set jane's credentials and create the context jane-context:*

```
Bash
# 1. Register credentials with jane.key and /tmp/jane.crt
kubectl config set-credentials jane \
  --client-certificate=/tmp/jane.crt \
  --client-key=/tmp/jane.key \
  --embed-certs=true

# 2. Create the context bound to namespace cka-6
kubectl config set-context jane-context \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=jane \
  --namespace=cka-6
```

## Step 6: Grant jane View Permissions in cka-6

*Bind user jane to the system cluster role view inside namespace cka-6:*

```
Bash
kubectl create rolebinding jane-view-binding \
  --clusterrole=view \
  --user=jane \
  --namespace=cka-6
```

## Step 7: Run Verification Checks

*Verify permissions using kubectl auth can-i as user jane in namespace cka-6:*

**Test 1: List Pods (Should return yes)**

```
Bash
kubectl auth can-i list pods --as=jane -n cka-6
Expected Output: yes
```

**Test 2: Delete Pods (Should return no)**

```
Bash
kubectl auth can-i delete pods --as=jane -n cka-6
Expected Output: no
```

## Task 7 — Node management (5%)

- Cordon k8s-mastery-worker so no new pods are scheduled
- Drain all workloads from it (ignore DaemonSets, force delete pods with local storage)
- Verify no non-DaemonSet pods remain on the node
- Uncordon the node
- Verify the scheduler places new pods there again — deploy a test Deployment and confirm

**Solution**

*Here is the step-by-step procedure to execute and verify Task 7.*

## Step 1: Cordon the Node

*Mark the node as unschedulable so new pods cannot be placed on it:*

```
Bash
kubectl cordon k8s-mastery-worker
Verification:

Bash
kubectl get node k8s-mastery-worker
(Should show status: Ready,SchedulingDisabled)
```

## Step 2: Drain All Workloads from the Node

*Evict all running workloads while ignoring DaemonSets and forcing the deletion of pods using local storage (emptyDir):*

```
Bash
kubectl drain k8s-mastery-worker --ignore-daemonsets --delete-emptydir-data --force
```

## Step 3: Verify No Non-DaemonSet Pods Remain

*Check the remaining pods running on k8s-mastery-worker:*

```
Bash
kubectl get pods -A --field-selector spec.nodeName=k8s-mastery-worker
```

*Confirm that the only pods remaining on the node belong to system DaemonSets (such as kube-proxy, aws-node or vpc-cni, calico-node, etc.).*

## Step 4: Uncordon the Node

*Mark the node as schedulable again:*

```
Bash
kubectl uncordon k8s-mastery-worker
Verification:

Bash
kubectl get node k8s-mastery-worker
(Should show status: Ready)
```

## Step 5: Verify Scheduler Places New Pods on the Node

*Deploy a temporary test deployment and scale it to ensure pods get scheduled on k8s-mastery-worker:*

```
Bash
# 1. Create a quick test deployment
kubectl create deployment drain-test --image=nginx:alpine --replicas=3 -n default

# 2. Check pod placement across nodes
kubectl get pods -n default -o wide

# 3. Clean up the test deployment
kubectl delete deployment drain-test -n default
```

*Confirm from the -o wide output that at least one pod from drain-test was scheduled onto k8s-mastery-worker.*

## Task 8 — Networking: NetworkPolicy (7%)

*In namespace cka-8:*

- Deploy Pod api (label role: api, image nginx:1.25)
- Deploy Pod db (label role: db, image nginx:1.25)
- Deploy Pod cache (label role: cache, image nginx:1.25)
- Expose all three with ClusterIP Services
- Apply NetworkPolicies such that:
- db accepts ingress ONLY from api
- cache accepts ingress ONLY from api
- api can reach db and cache but blocks all other ingress
- All pods can egress to DNS (UDP 53)
- Verify by exec into a pod and testing connectivity

**Solution**

*Here is the step-by-step solution for Task 8.*

## Step 1: Create Namespace and Deploy Pods & Services

```
# 1. Create the namespace
kubectl create namespace cka-8 --dry-run=client -o yaml | kubectl apply -f -

# 2. Deploy Pods with appropriate labels
kubectl run api --image=nginx:1.25 --labels="role=api" -n cka-8
kubectl run db --image=nginx:1.25 --labels="role=db" -n cka-8
kubectl run cache --image=nginx:1.25 --labels="role=cache" -n cka-8

# 3. Expose Pods via ClusterIP services on port 80
kubectl expose pod api --port=80 --target-port=80 -n cka-8
kubectl expose pod db --port=80 --target-port=80 -n cka-8
kubectl expose pod cache --port=80 --target-port=80 -n cka-8
```

## Step 2: Apply Network Policies

*Create a combined NetworkPolicy manifest defining the ingress and egress rules for all three pods:*

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress-policy
  namespace: cka-8
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 80
  egress:
  - ports:
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache-ingress-policy
  namespace: cka-8
spec:
  podSelector:
    matchLabels:
      role: cache
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 80
  egress:
  - ports:
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress-egress-policy
  namespace: cka-8
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Ingress
  - Egress
  ingress: [] # Blocks all external ingress into API pod
  egress:
  # Allow DNS lookup
  - ports:
    - protocol: UDP
      port: 53
  # Allow egress to db and cache on port 80
  - to:
    - podSelector:
        matchLabels:
          role: db
    - podSelector:
        matchLabels:
          role: cache
    ports:
    - protocol: TCP
      port: 80
EOF
```

## Step 3: Run Verification Checks

**Test 1: api reaching db and cache (Should succeed)**

```
# Test connection from api -> db
kubectl exec -n cka-8 api -- curl -s --connect-timeout 2 db

# Test connection from api -> cache
kubectl exec -n cka-8 api -- curl -s --connect-timeout 2 cache
```

*Expected Output: HTML output from Nginx (200 OK).*

**Test 2: Direct connection from db -> cache (Should time out / fail)**

```
kubectl exec -n cka-8 db -- curl -s --connect-timeout 2 cache
```

*Expected Output: Connection timed out (curl: (28) Connection timed out).*

**Test 3: Direct connection from cache -> api (Should time out / fail)**

```
kubectl exec -n cka-8 cache -- curl -s --connect-timeout 2 api
```

*Expected Output: Connection timed out (curl: (28) Connection timed out).*

**Test 4: DNS resolution from any pod (Should succeed)**

```
kubectl exec -n cka-8 db -- nslookup kubernetes.default
```

*Expected Output: Resolves IP address for kubernetes.default.svc.cluster.local.*

## Task 9 — Storage: PV, PVC, Pod (7%)

*In namespace cka-9:*

- Create a PersistentVolume cka-pv: capacity 2Gi, ReadWriteOnce, hostPath: /tmp/cka-storage, storageClass cka-sc, reclaimPolicy Retain
- Create a PersistentVolumeClaim cka-pvc: requests 1Gi, storageClass cka-sc
- Verify PVC is Bound
- Create a Pod storage-writer that writes hostname to /data/hostname.txt on startup then sleeps
- Verify the file exists on the hostPath after the pod runs

**Solution**

*Here is the step-by-step solution for Task 9.*

## Step 1: Create Namespace cka-9

```
Bash
kubectl create namespace cka-9 --dry-run=client -o yaml | kubectl apply -f -
```

## Step 2: Create the PersistentVolume (PV) and PersistentVolumeClaim (PVC)

*Apply the manifest defining cka-pv and cka-pvc:*

```
Bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cka-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cka-sc
  hostPath:
    path: /tmp/cka-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cka-pvc
  namespace: cka-9
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: cka-sc
EOF
```

## Step 3: Verify PVC is Bound

*Check that cka-pvc successfully bound to cka-pv:*

```
Bash
kubectl get pvc -n cka-9 cka-pvc
Expected Output:

Plaintext
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cka-pvc   Bound    cka-pv   2Gi        RWO            cka-sc         5s
```

## Step 4: Create the Pod storage-writer

*Create the pod that writes its hostname to /data/hostname.txt and sleeps:*

```
Bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-writer
  namespace: cka-9
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "hostname > /data/hostname.txt && sleep 3600"]
    volumeMounts:
    - name: storage-vol
      mountPath: /data
  volumes:
  - name: storage-vol
    persistentVolumeClaim:
      claimName: cka-pvc
EOF
```

## Step 5: Verify the File Exists on the HostPath

*Wait for the pod to transition to Running, then verify the file contents:*

**1. Check from inside the Pod**

```
Bash
kubectl exec -n cka-9 storage-writer -- cat /data/hostname.txt
Expected Output: storage-writer
```

**2. Check from the Host / Node (/tmp/cka-storage)**

*SSH or log onto the node where the pod is running and check the host directory:*

```
Bash
cat /tmp/cka-storage/hostname.txt
Expected Output: storage-writer
```

## Task 10 — Deployment operations (6%)

*In namespace cka-10:*

- Create a Deployment webapp with image nginx:1.24, 3 replicas
- Scale to 5 replicas imperatively
- Update the image to nginx:1.25 — record the cause as "upgrade to 1.25"
- Verify rollout completes with zero unavailable pods
- Roll back to the previous version
- Verify pods are running nginx:1.24 again

**Solution**

*Here is the complete step-by-step solution for Task 10.*

**Step 1: Create Namespace and Initial Deployment**

```
Bash
# 1. Create namespace
kubectl create namespace cka-10 --dry-run=client -o yaml | kubectl apply -f -

# 2. Create webapp deployment with nginx:1.24 and 3 replicas
kubectl create deployment webapp \
  --image=nginx:1.24 \
  --replicas=3 \
  -n cka-10
```

**Step 2: Scale Deployment to 5 Replicas Imperatively**

```
Bash
kubectl scale deployment webapp --replicas=5 -n cka-10
```

**Step 3: Update Image with Change Cause Annotation**

*Update the image to nginx:1.25 and set the kubernetes.io/change-cause annotation to record the reason:*

```
Bash
# 1. Update image
kubectl set image deployment/webapp nginx=nginx:1.25 -n cka-10

# 2. Record the change-cause annotation
kubectl annotate deployment/webapp kubernetes.io/change-cause="upgrade to 1.25" -n cka-10

# 3. Verify rollout completion
kubectl rollout status deployment/webapp -n cka-10
```

**Step 4: Verify Rollout Status and Image Version**

*Check the rollout history and ensure all 5 pods are running nginx:1.25:*

```
Bash
# Verify history contains the change cause
kubectl rollout history deployment/webapp -n cka-10

# Verify active container images in running pods
kubectl get pods -n cka-10 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

**Step 5: Roll Back to Previous Version and Verify**

```
Bash
# 1. Perform undo rollback
kubectl rollout undo deployment/webapp -n cka-10

# 2. Wait for rollback to finish
kubectl rollout status deployment/webapp -n cka-10

# 3. Verify pods are running nginx:1.24 again
kubectl get pods -n cka-10 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
Expected Output:

Plaintext
webapp-xxxxx-xxxxx    nginx:1.24
webapp-xxxxx-xxxxx    nginx:1.24
webapp-xxxxx-xxxxx    nginx:1.24
webapp-xxxxx-xxxxx    nginx:1.24
webapp-xxxxx-xxxxx    nginx:1.24
```

## Task 11 — ConfigMap and Secret in a Pod (5%)

*In namespace cka-11:*

- Create ConfigMap app-env with: ENV=production, REGION=eu-west-1
- Create Secret db-creds with: DB_PASS=secretpass
- Create Pod config-consumer:
- All ConfigMap keys injected as env vars
- DB_PASS injected from Secret as env var
- ConfigMap mounted as files at /etc/app-config
- Verify all three are accessible inside the pod

**Solution**

*Here is the step-by-step solution for Task 11.*

**Step 1: Create Namespace cka-11**

```
Bash
kubectl create namespace cka-11 --dry-run=client -o yaml | kubectl apply -f -
```

**Step 2: Create ConfigMap and Secret**

```
Bash
# 1. Create ConfigMap app-env
kubectl create configmap app-env \
  --from-literal=ENV=production \
  --from-literal=REGION=eu-west-1 \
  -n cka-11

# 2. Create Secret db-creds
kubectl create secret generic db-creds \
  --from-literal=DB_PASS=secretpass \
  -n cka-11
```

**Step 3: Create Pod config-consumer**

*Create the pod configured with:*

- envFrom to inject all keys from app-env as environment variables.
- env mapping DB_PASS specifically from the db-creds Secret.
- volumes and volumeMounts mounting app-env at /etc/app-config.

```
Bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-consumer
  namespace: cka-11
spec:
  containers:
  - name: consumer
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-env
    env:
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: DB_PASS
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app-config
  volumes:
  - name: config-volume
    configMap:
      name: app-env
EOF
```

**Step 4: Run Verification Checks**

**Test 1: Verify Environment Variables**

*Check that ENV, REGION, and DB_PASS are present inside the container:*

```
Bash
kubectl exec -n cka-11 config-consumer -- printenv | grep -E 'ENV|REGION|DB_PASS'
Expected Output:

Plaintext
ENV=production
REGION=eu-west-1
DB_PASS=secretpass
```

**Test 2: Verify Volume Mounts**

*Check that the files were mounted properly under /etc/app-config:*

```
Bash
kubectl exec -n cka-11 config-consumer -- ls /etc/app-config
Expected Output:

Plaintext
ENV
REGION
```

```
Bash
kubectl exec -n cka-11 config-consumer -- cat /etc/app-config/ENV /etc/app-config/REGION
Expected Output:

Plaintext
production
eu-west-1
```

## Task 12 — Troubleshoot a broken cluster component (8%)

*A scheduler is broken on your cluster. Fix it.*

```
# Break it first
docker exec k8s-mastery-control-plane bash -c \
  "sed -i 's|kube-scheduler|kube-schedulerXXX|g' \
  /etc/kubernetes/manifests/kube-scheduler.yaml"

# Wait 30 seconds then try:
kubectl run test-broken --image=nginx:1.25
kubectl get pod test-broken    # should stay Pending — scheduler is broken
```

- Diagnose exactly what is broken
- Fix it
- Verify test-broken pod gets scheduled and runs

## Task 13 — Troubleshoot a broken worker node (8%)

*Break and fix a worker node.*

```
# Break the kubelet on the worker
docker exec k8s-mastery-worker bash -c "systemctl stop kubelet"

# Wait 60 seconds
```

- Identify the node is NotReady
- SSH into (exec into) the worker node
- Diagnose what is wrong
- Fix it
- Verify the node returns to Ready

## Task 14 — Resource management (5%)

*In namespace cka-14:*

- Create a ResourceQuota tight-quota: max 4 pods, requests.cpu=1, requests.memory=1Gi, limits.cpu=2, limits.memory=2Gi
- Create a LimitRange container-defaults: default CPU limit 300m, default memory limit 256Mi, default CPU request 100m, default memory request 128Mi
- Create a Deployment quota-test with 2 replicas, image nginx:1.25, NO resource spec
- Verify LimitRange injected defaults
- Try to scale to 6 replicas — verify it's rejected by quota
- Write the quota usage output to /tmp/quota-status.txt

**Solution**

*Here is the step-by-step solution for Task 14.*

## Step 1: Create Namespace cka-14

```
Bash
kubectl create namespace cka-14 --dry-run=client -o yaml | kubectl apply -f -
```

## Step 2: Create ResourceQuota and LimitRange

*Apply the manifest defining tight-quota and container-defaults:*

```
Bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tight-quota
  namespace: cka-14
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: container-defaults
  namespace: cka-14
spec:
  limits:
  - default:
      cpu: 300m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
EOF
```

## Step 3: Create Deployment quota-test with 2 Replicas and No Resources Spec

```
Bash
kubectl create deployment quota-test \
  --image=nginx:1.25 \
  --replicas=2 \
  -n cka-14
```

## Step 4: Verify LimitRange Injected Default Resources

*Inspect one of the running pods to confirm the LimitRange automatically applied requests and limits:*

```
Bash
kubectl get pods -n cka-14 -l app=quota-test -o jsonpath='{.items[0].spec.containers[0].resources}'
Expected Output:

JSON
{"limits":{"cpu":"300m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

## Step 5: Scale to 6 Replicas and Verify Rejection

*Attempt to scale quota-test past the 4-pod quota ceiling:*

```
Bash
kubectl scale deployment quota-test --replicas=6 -n cka-14
Check the ReplicaSet events to confirm the ResourceQuota rejected pods exceeding the limit:

Bash
kubectl describe rs -n cka-14 -l app=quota-test
Expected Warning Event:

FailedCreate ... forbidden: exceeded quota: tight-quota, requested: pods=1, used: pods=4, limited: pods=4
```

## Step 6: Write Quota Usage Output to /tmp/quota-status.txt

*Output the quota usage details directly to /tmp/quota-status.txt:*

```
Bash
kubectl describe resourcequota tight-quota -n cka-14 > /tmp/quota-status.txt
```

**Verification**

```
Check the contents of /tmp/quota-status.txt:

Bash
cat /tmp/quota-status.txt
Expected Output Format:

Plaintext
Name:            tight-quota
Namespace:       cka-14
Resource         Used   Hard
--------         ----   ----
limits.cpu       600m   2
limits.memory    512Mi  2Gi
pods             4      4
requests.cpu     200m   1
requests.memory  256Mi  1Gi
```






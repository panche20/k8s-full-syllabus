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












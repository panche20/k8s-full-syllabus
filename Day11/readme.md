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
























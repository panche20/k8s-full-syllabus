# Upgrading Kubernetes Cluster with kubeadm (v1.33 to v1.34)

A complete step-by-step guide to upgrading a Kubernetes cluster created with kubeadm from v1.33 to v1.34.

## Prerequisites & Version Planning

Before beginning, ensure all nodes have active network access, sufficient disk space, and that cluster backups (such as etcd snapshots) have been completed.

### Step 0: Update Kubernetes Package Repository on ALL Nodes

Run this on the **control plane and all worker nodes**:

```
# Update repository definition to target v1.34
sudo sed -i 's/v1.33/v1.34/g' /etc/apt/sources.list.d/kubernetes.list

# Update package index and check available versions
sudo apt-get update
sudo apt-cache madison kubeadm
```

*(Identify the target version string, e.g., 1.34.1-1.1)*

## Phase 1: Upgrading the Control Plane Node

**Note: Run commands on the Control Plane node (master) unless specified otherwise.**

**1. Upgrade kubeadm**

```
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm='1.34.1-1.1'
sudo apt-mark hold kubeadm

# Verify binary version
kubeadm version
```

**2. Plan and Apply the Control Plane Upgrade**

```
# Verify upgrade components and readiness
sudo kubeadm upgrade plan

# Apply upgrade (version must match your installed kubeadm version)
sudo kubeadm upgrade apply v1.34.1 -y
```

**3. Drain the Control Plane Node**

```
kubectl drain master --ignore-daemonsets --delete-emptydir-data
```

**4. Upgrade Kubelet & Kubectl**

```
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet='1.34.1-1.1' kubectl='1.34.1-1.1'
sudo apt-mark hold kubelet kubectl

# Reload and restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**5. Uncordon the Control Plane Node**

```
kubectl uncordon master
```

## Phase 2: Upgrading Worker Nodes

Repeat this section one worker node at a time.

**1. Drain the Worker Node**

(Execute on the Control Plane)

```
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
```

**2. Upgrade kubeadm**

(Execute on worker-1)

```
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm='1.34.1-1.1'
sudo apt-mark hold kubeadm
```

**3. Upgrade Node Configuration**
(Execute on worker-1)

```
sudo kubeadm upgrade node
```

**4. Upgrade Kubelet & Kubectl**
(Execute on worker-1)

```
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet='1.34.1-1.1' kubectl='1.34.1-1.1'
sudo apt-mark hold kubelet kubectl

# Reload and restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**5. Uncordon the Worker Node**
(Execute on the Control Plane)

```
kubectl uncordon worker-1
```

## Phase 3: Cluster Verification

**Execute on the Control Plane:**

```
kubectl get nodes -o wide
```

**Expected output:**

```
NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   30d   v1.34.1
worker-1   Ready    <none>          30d   v1.34.1
```

**Check static pods and core system workloads:**

```
kubectl get pods -n kube-system
```

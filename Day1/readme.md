# Day 1 — Kubernetes Cluster Architecture Deep Dive
Let's build the mental model first, then get hands-on. By end of today you'll understand K8s internals well enough to answer any architecture question in an interview.

## Part 1: The Big Picture
Kubernetes is a distributed system with two kinds of machines:

Control Plane — the brain. Makes decisions.
Worker Nodes — the muscle. Runs your workloads.

## Part 2: Every Component Explained

### Control Plane Components

kube-apiserver — The only component everything talks to. It's the front door. kubectl, kubelet, controllers — all go through it. It validates, authenticates, authorizes, and persists to etcd. Stateless — you can run multiple replicas for HA.

etcd — A distributed key-value store. Holds 100% of cluster state: nodes, pods, configs, secrets, everything. If etcd dies, your cluster brain dies. This is why etcd backup (Day 10) is a CKA exam staple. Only the API server talks to etcd directly.

kube-scheduler — Watches for unscheduled pods, picks the best node using a scoring algorithm (resource fit, affinity, taints/tolerations, topology spread). It only decides — kubelet does the actual work.

kube-controller-manager — Runs a bundle of control loops in one binary: Node controller, Deployment controller, ReplicaSet controller, Endpoints controller, etc. Each loop watches state and reconciles toward desired state.

cloud-controller-manager — Separates cloud-specific logic (AWS Load Balancers, EBS volumes, node lifecycle) from core K8s. Introduced so K8s stays cloud-agnostic.

### Worker Node Components

kubelet — The node agent. Talks to the API server, receives pod specs (PodSpecs), and tells the container runtime to start/stop containers. Also runs liveness/readiness probes.

kube-proxy — Maintains network rules on every node (iptables or IPVS). Enables Service abstraction — when you hit a ClusterIP, kube-proxy routes it to the right pod.

Container Runtime (containerd) — The actual thing that runs containers. Kubernetes talks to it via CRI (Container Runtime Interface). Docker was replaced by containerd as the default.

## Part 3: What Happens When You Run kubectl apply
This is a classic interview question. Walk through it step by step:

```
1. kubectl → HTTPS request to kube-apiserver
2. API server: authenticate (cert/token) → authorize (RBAC) → admission controllers
3. API server writes the object to etcd
4. API server returns 200 OK to kubectl
5. Controller-manager detects new Deployment → creates ReplicaSet → creates Pods
6. Pods are in "Pending" state (no node assigned yet)
7. Scheduler watches Pending pods → scores nodes → assigns node to pod (writes to etcd)
8. kubelet on the assigned node watches its pod list → sees new pod
9. kubelet calls containerd via CRI → pulls image → starts container
10. kubelet updates pod status → Running
```

## Part 4: Hands-On Setup on Your EC2 Ubuntu Instance
Let's install kind (Kubernetes in Docker) — perfect for your EC2 instance.

### Step 1: Install Kind + Kubectl

```
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

# Verify
kubectl version --client
kind version
```
### Step 2: Create a multi-node cluster

```
# Create a config file for a 3-node cluster
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name k8s-mastery --config kind-config.yaml
```

### Step 3: Explore the control plane components

```
# See all system pods (control plane lives here)
kubectl get pods -n kube-system

# Describe the API server pod — see all its flags
kubectl describe pod -n kube-system -l component=kube-apiserver

# Check etcd
kubectl describe pod -n kube-system -l component=etcd

# Check component health
kubectl get componentstatuses   # deprecated but still tested in CKA

# See nodes
kubectl get nodes -o wide
```

### Step 4: Understand contexts

```
# See all contexts (clusters you can talk to)
kubectl config get-contexts

# Switch context
kubectl config use-context kind-k8s-mastery

# See current context
kubectl config current-context

# Shortcut — set a namespace so you don't type -n every time
kubectl config set-context --current --namespace=default
```

### Step 5: kubectl power user commands

```
# Explain any resource (your best friend)
kubectl explain pod
kubectl explain pod.spec.containers

# Get everything in a namespace
kubectl get all -n kube-system

# Watch resources in real time
kubectl get pods -w

# Raw etcd read (see what's actually stored)
# Run inside the etcd pod:
kubectl exec -it -n kube-system etcd-k8s-mastery-control-plane -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/namespaces/default
```

## Part 5: Interview Questions — Day 1

Practice answering these out loud:
Q1: What happens if etcd goes down?

The cluster continues running existing workloads (kubelet doesn't need etcd), but no new changes can be made — no deployments, no scaling, nothing. It's read-only effectively.

Q2: Can you run workloads on the control plane node?

By default no — a taint (node-role.kubernetes.io/control-plane:NoSchedule) prevents it. You can remove the taint but it's not recommended in production.

Q3: What's the difference between kube-scheduler and controller-manager?

Scheduler decides where a pod runs. Controller-manager ensures the desired state is maintained (right number of replicas, etc).

Q4: Is the API server stateless?

Yes. All state lives in etcd. That's why you can run multiple API server replicas for HA with a load balancer in front.

Q5: What is a watch in Kubernetes?

Kubernetes components don't poll — they use etcd watches (long-lived HTTP connections). When state changes, etcd pushes a notification. This is how the scheduler instantly knows a new pod is Pending.

# Explaination of YAML files.

```
Can you explain me with this YAML file

ubuntu@ip-172-31-1-93:~$ kubectl get configmap coredns -n kube-system -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30 {
           disable success cluster.local
           disable denial cluster.local
        }
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2026-07-30T05:36:08Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "262"
  uid: 072d0876-ea58-4634-b6a0-cc90b784b5d5
ubuntu@ip-172-31-1-93:~$
```

*This is the standard Kubernetes ConfigMap used to configure CoreDNS, which is the internal DNS server running inside your Kubernetes cluster.*

**CoreDNS resolves domain names for your pods and services (like mapping my-service.default.svc.cluster.local to an internal IP address).**

## 1. Top-Level Kubernetes Wrapper

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

- kind: ConfigMap: A Kubernetes object used to store non-confidential configuration data in key-value pairs.
- namespace: kube-system: Lives in the system namespace where core Kubernetes components run.
- data.Corefile: The key Corefile holds the actual CoreDNS configuration string.

## 2. CoreDNS Configuration (Corefile) Breakdown

*The block starting with .:53 { ... } means: "Listen for all DNS queries on port 53 (the standard DNS port)."*

**Here is what each plugin block inside does:**

**🛠️ Basic Health & Management**

- errors: Logs any errors that occur during DNS resolution to standard error (stderr).
- health { lameduck 5s }: Exposes a health endpoint for Kubernetes liveness probes. When shutting down, it enters a "lameduck" period for 5 seconds to let existing connections finish before stopping.
- ready: Exposes an HTTP readiness probe on port 8181 so Kubernetes knows when CoreDNS is ready to accept traffic.
- reload: Automatically reloads the configuration whenever this ConfigMap is modified—no need to restart the CoreDNS pods manually!

## 🌐 Kubernetes Cluster DNS Resolution

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
   pods insecure
   fallthrough in-addr.arpa ip6.arpa
   ttl 30
}
```

- cluster.local: The default domain suffix for your Kubernetes cluster. CoreDNS listens for queries ending in this zone.
- pods insecure: Resolves pod IP addresses directly (e.g., 10-244-0-5.default.pod.cluster.local). The insecure setting allows backward-compatible A-record lookups without strict IP checking.
- fallthrough ...: If a reverse DNS lookup (in-addr.arpa for IPv4 or ip6.arpa for IPv6) isn't matched by the kubernetes plugin, pass it down to the next plugins (like forward).
- ttl 30: Sets the Time-To-Live for DNS responses to 30 seconds.

## 🔄 External Forwarding & Performance

```
forward . /etc/resolv.conf {
   max_concurrent 1000
}
```

- forward . /etc/resolv.conf: If a query isn't inside cluster.local (for example, looking up google.com), CoreDNS forwards it to the upstream DNS servers defined in the node's /etc/resolv.conf.
- max_concurrent 1000: Caps concurrent upstream DNS queries at 1,000 to prevent overwhelming external DNS servers under high load.

```
cache 30 {
   disable success cluster.local
   disable denial cluster.local
}
```

- Caches external DNS responses for up to 30 seconds.
- disable ... cluster.local: Disables caching for internal cluster addresses so that new Kubernetes service IP changes are picked up instantly without waiting for cache expiration.

## 📊 Monitoring & Routing

- prometheus :9153: Exposes CoreDNS metrics for Prometheus scrapers on port 9153.
- loop: Detects simple DNS forwarding loops (e.g., if /etc/resolv.conf points back to CoreDNS itself) and halts CoreDNS to prevent infinite loops.
- loadbalance: Randomizes the order of A/AAAA records in responses when a service has multiple pods, spreading traffic evenly.

# How to read logs:

```
How to read these logs?

ubuntu@ip-172-31-1-93:~$ kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
coredns-674b8bbfcf-9v89h   1/1     Running   0          52m   192.168.219.71   master   <none>           <none>
coredns-674b8bbfcf-wxl82   1/1     Running   0          52m   192.168.219.66   master   <none>           <none>

ubuntu@ip-172-31-1-93:~$ kubectl logs -n kube-system -l k8s-app=kube-dns
maxprocs: Leaving GOMAXPROCS=2: CPU quota undefined
.:53
[INFO] plugin/reload: Running configuration SHA512 = 1b226df79860026c6a52e67daa10d7f0d57ec5b023288ec00c5e05f93523c894564e15b91770d3a07ae1cfbe861d15b37d4a0027e69c546ab112970993a3b03b
CoreDNS-1.12.0
linux/amd64, go1.23.3, 51e11f1
maxprocs: Leaving GOMAXPROCS=2: CPU quota undefined
.:53
[INFO] plugin/reload: Running configuration SHA512 = 1b226df79860026c6a52e67daa10d7f0d57ec5b023288ec00c5e05f93523c894564e15b91770d3a07ae1cfbe861d15b37d4a0027e69c546ab112970993a3b03b
CoreDNS-1.12.0
linux/amd64, go1.23.3, 51e11f1
ubuntu@ip-172-31-1-93:~$
```

*Here is how to read both the output from kubectl get pods and the kubectl logs command output.*

## Part 1: Reading kubectl get pods

```
NAME                      READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
coredns-674b8bbfcf-9v89h   1/1     Running   0          52m   192.168.219.71   master   <none>           <none>
coredns-674b8bbfcf-wxl82   1/1     Running   0          52m   192.168.219.66   master   <none>           <none>
```

- NAME: The name of each CoreDNS pod running inside your cluster. There are two replicas for high availability.
- READY (1/1): Indicates that 1 out of 1 container in each pod is fully initialized and passing its readiness probes.
- STATUS (Running): Both pods are actively running without any crashes or failures.
- RESTARTS (0): Neither container has restarted since creation, which indicates stability.
- AGE (52m): The pods were created 52 minutes ago.
- IP: The internal Pod network IP assigned to each CoreDNS instance (192.168.219.71 and 192.168.219.66).
- NODE (master): Both pods are scheduled on the node named master.

## Part 2: Reading kubectl logs

*Because you ran kubectl logs -n kube-system -l k8s-app=kube-dns, Kubernetes combined the logs from both CoreDNS pods that matched the label k8s-app=kube-dns. That is why you see two identical blocks of output.*

**Here is what each line in those blocks means:**

**1. CPU Quota Initialization**

```
maxprocs: Leaving GOMAXPROCS=2: CPU quota undefined
```

- CoreDNS is written in Go. The Go runtime detected 2 CPU cores available on the node (GOMAXPROCS=2) and set its concurrency threads accordingly because no explicit Kubernetes CPU limit was set on the pod.

**2. Server Block Binding**

```
.:53
```

- Confirms that CoreDNS started successfully and is listening for DNS requests on port 53 for all domains (.).

**3. Corefile Load Verification**

```
[INFO] plugin/reload: Running configuration SHA512 = 1b226df79860026c6a52e67daa10d7f0d57ec5b023288ec00c5e05f93523c894564e15b91770d3a07ae1cfbe861d15b37d4a0027e69c546ab112970993a3b03b
```

- The reload plugin computed a SHA512 hash of your active Corefile configuration. CoreDNS watches for changes to this hash; if you edit the ConfigMap, it will log a new hash when it auto-reloads.

**4. Build Information**

```
CoreDNS-1.12.0
linux/amd64, go1.23.3, 51e11f1
```

- Displays the CoreDNS version (1.12.0), architecture (linux/amd64), Go compiler version (1.23.3), and the Git commit hash (51e11f1).

**Summary**

- Everything shown in your terminal output represents a healthy, fully operational CoreDNS deployment:
- Both pods are Running with zero restarts.
- CoreDNS successfully started, loaded its configuration, and is listening on port 53.

## Logs reading:

```
How do I read these logs?

ubuntu@ip-172-31-1-93:~$ sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers | head -40

Chain KUBE-SERVICES (2 references)
num  target     prot opt source               destination
1    KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
2    KUBE-SVC-I24EZXP75AX5E7TU  tcp  --  0.0.0.0/0            10.104.82.28         /* calico-apiserver/calico-api:apiserver cluster IP */ tcp dpt:443
3    KUBE-SVC-UA7HCJMIMBJJDU4H  tcp  --  0.0.0.0/0            10.97.111.122        /* calico-system/whisker cluster IP */ tcp dpt:8081
4    KUBE-SVC-RMIM2MJTF3UE534N  tcp  --  0.0.0.0/0            10.111.128.49        /* calico-system/goldmane cluster IP */ tcp dpt:7443
5    KUBE-SVC-RK657RLKDNVNU64O  tcp  --  0.0.0.0/0            10.99.235.130        /* calico-system/calico-typha:calico-typha cluster IP */ tcp dpt:5473
6    KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
7    KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
8    KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
9    KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

ubuntu@ip-172-31-1-93:~$
```

*This output shows the core Network Address Translation (NAT) routing table managed by kube-proxy.*

*In Kubernetes, when a Pod talks to a Service's Virtual IP (ClusterIP), iptables intercepts the traffic at the Linux kernel level and jumps down a sequence of rule chains to balance traffic across back-end Pods.*

**1. Understanding the Table Columns**

<img width="860" height="646" alt="image" src="https://github.com/user-attachments/assets/571a9f1e-109c-4569-806b-baa8b03d5c7f" />

**2. Row-by-Row Breakdown**

*Rules 1–5: Core Control Plane & Network Services*

- Rule 1 (10.96.0.1:443): Handles secure HTTPS traffic directed to the Kubernetes API server itself (default/kubernetes).
- Rules 2–5 (10.104.82.28, etc.): Manage Calico networking components (calico-apiserver, whisker, goldmane, calico-typha). These direct internal cluster networking and security policies.

*Rules 6–8: CoreDNS Routing (Connecting to the ConfigMap from earlier)*

Notice how your CoreDNS setup is registered here across three distinct rules:

- Rule 6: Intercepts UDP traffic sent to 10.96.0.10:53 (standard DNS queries).
- Rule 7: Intercepts TCP traffic sent to 10.96.0.10:53 (fallback for large DNS responses over 512 bytes).
- Rule 8: Intercepts metrics requests sent to 10.96.0.10:9153 (for Prometheus monitoring).

*Rule 9: Catch-All NodePorts*

- Rule 9 (KUBE-NODEPORTS): Positioned at the absolute end. If a packet doesn't match any ClusterIP directly, this rule checks if the packet was sent to a NodePort on the physical node's external IP address.

**3. How Traffic Flows Through This Table (An Example)**

*When a Pod sends a query to CoreDNS (10.96.0.10:53 via UDP):*

```
[ Pod Query ] 
     │
     ▼
[ KUBE-SERVICES Chain ] ──> Matches Rule 6 
     │
     ▼
[ Jumps to KUBE-SVC-TCOU7JCQXEZGVUNU Chain ]
     │
     ├─> Picks a Pod IP randomly (192.168.219.71 or 192.168.219.66)
     │
     ▼
[ Rewrites Destination IP (DNAT) ] ──> Sends traffic directly to the chosen CoreDNS Pod
```

- Kernel enters KUBE-SERVICES and checks rules from top to bottom.
- It hits Rule 6 because the target IP is 10.96.0.10 on UDP port 53.
- It jumps to the child chain KUBE-SVC-TCOU7JCQXEZGVUNU.
- That child chain performs load balancing and rewrites the destination IP from 10.96.0.10 to one of your real CoreDNS Pod IPs (192.168.219.71 or 192.168.219.66).


# How to Enable Audit Logging in kubeadm

*If you want audit logs to be recorded to /var/log/kubernetes/audit.log, follow these steps on your control plane node:*

## Step 1: Create an Audit Policy File

*Create a policy file that tells Kubernetes what events to log (e.g., at /etc/kubernetes/audit-policy.yaml):*

```
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Don't log read-only requests (get, list, watch)
  - level: None
    verbs: ["get", "list", "watch"]

  # Log resource changes (create, update, delete) at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods", "services", "configmaps"]
```

## Step 2: Configure kube-apiserver to Use the Policy

*Edit the API server manifest file on your control plane:*

```
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

*Add the following flags under spec.containers.command:*

```
spec:
  containers:
  - command:
    - kube-apiserver
    # Add these 3 lines:
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxbackup=10
```

*Then, under volumeMounts and volumes, make sure the host directory /var/log/kubernetes and policy file are mounted into the container:*

```
volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-log
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
```

*Once saved, kubelet will automatically restart kube-apiserver. From that point on, /var/log/kubernetes/audit.log will begin generating on your control plane host.*

- Now that audit logging is active, every single action hitting your Kubernetes API server is recorded. Since you just created and deleted an nginx pod, those events are saved in your audit log file.
- Depending on how your cluster is set up (bare metal/VM vs. a containerized control plane like k8s-mastery-control-plane), here is how to find those exact logs.

## Step 1: Access the Audit Log File

*If your control plane is a standard Linux VM or host, the file lives directly on disk. If you are using a Docker-based lab environment (like k8s-mastery-control-plane), you'll access it via docker exec.*

**If on a standard host VM:**

```
sudo tail -f /var/log/kubernetes/audit.log
```

**If using a Docker-based control plane container:**

```
docker exec -it k8s-mastery-control-plane tail -f /var/log/kubernetes/audit.log
```

## Step 2: Filter for Your "Nginx" Pod Actions

*Because audit logs are dense JSON lines, piping them through grep and jq makes it easy to spot the creation, updates, and deletion of your nginx pod.*

**Run this command on your control plane node:**

```
sudo grep -i "nginx" /var/log/kubernetes/audit.log \
  | jq '{time: .requestReceivedTimestamp, user: .user.username, verb: .verb, resource: .objectRef.resource, name: .objectRef.name}'
```

**What to Look For in the Output**

*When you run the filter, you should see a sequence of JSON blocks mapping out your exact test:*

- "verb": "create": Shows the moment you deployed the pod, capturing your username and the resource name (nginx).
- "verb": "update" / "patch": Internal updates made by Kubernetes controllers (like binding the pod to a node or updating its status).
- "verb": "delete": Shows the moment you deleted the pod, verifying who executed the command and when.

**TROUBLESHOOTING**

```
ubuntu@master:/etc/kubernetes/manifests$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "kubernetes-admin" cannot list resource "pods" in API group "" in the namespace "default"
ubuntu@master:/etc/kubernetes/manifests$
```

- The error occurs because the user identity (kubernetes-admin) specified in your active kubeconfig file lacks the RBAC permissions required to list pods in the default namespace.
- This usually happens when kubeconfig was generated with a restricted role or certificate, or if default RBAC cluster roles/bindings were modified.
- Here is how to diagnose and fix the permission issue on your control plane node:

## Step 1: Check Current Kubeconfig Context

*Verify which user and context your kubectl command is using:*

```
kubectl config get-contexts
kubectl config view --minify
```

*Check the user section to ensure you are pointing to the expected certificate/user credentials.*

## Step 2: Test Permissions via can-i

*Check if kubernetes-admin has cluster-admin rights or if the restriction applies everywhere:*

```
# Check if you can list pods in default
kubectl auth can-i list pods -n default

# Check if you have full admin access across the cluster
kubectl auth can-i "*" "*"
```

*If both return no, your current credentials do not have administrative privileges.*

## Step 3: Restore Superuser Access

*Since you are directly on the control plane node (ubuntu@master), you can bypass broken RBAC configurations or user restrictions using the emergency administrative key generated by kubeadm.*

**Option A: Use the Native admin.conf Credentials**

*By default, kubeadm generates a full superuser configuration file at /etc/kubernetes/admin.conf. Use this file directly to grant your account access or fix RBAC:*

```
# Test if the master admin.conf works
sudo kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -A
```

**If that works, copy admin.conf to your user directory to fix your default kubectl configuration:**

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Option B: Rebind cluster-admin Role (If RBAC was modified)

*If the kubernetes-admin user was intentionally bound to a role that lost permissions, use the superuser configuration to restore the binding:*

```
sudo kubectl --kubeconfig=/etc/kubernetes/admin.conf create clusterrolebinding fix-admin-binding \
  --clusterrole=cluster-admin \
  --user=kubernetes-admin
```

## Step 4: Verify the Fix

*After restoring your ~/.kube/config, verify that your standard kubectl commands function properly:*

```
kubectl get pods
```






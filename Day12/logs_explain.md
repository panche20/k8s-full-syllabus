ubuntu@ip-172-31-1-93:~$ kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES

coredns-674b8bbfcf-9v89h   1/1     Running   0          52m   192.168.219.71   master   <none>           <none>

coredns-674b8bbfcf-wxl82   1/1     Running   0          52m   192.168.219.66   master   <none>           <none>

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

ized and passing its readiness probes.

STATUS (Running): Both pods are actively running without any crashes or failures.

RESTARTS (0): Neither container has restarted since creation, which indicates stability.

AGE (52m): The pods were created 52 minutes ago.

IP: The internal Pod network IP assigned to each CoreDNS instance (192.168.219.71 and 192.168.219.66).

NODE (master): Both pods are scheduled on the node named master.

Part 2: Reading kubectl logs
Because you ran kubectl logs -n kube-system -l k8s-app=kube-dns, Kubernetes combined the logs from both CoreDNS pods that matched the label k8s-app=kube-dns. That is why you see two identical blocks of output.

Here is what each line in those blocks means:

1. CPU Quota Initialization
Plaintext
maxprocs: Leaving GOMAXPROCS=2: CPU quota undefined
CoreDNS is written in Go. The Go runtime detected 2 CPU cores available on the node (GOMAXPROCS=2) and set its concurrency threads accordingly because no explicit Kubernetes CPU limit was set on the pod.

2. Server Block Binding
Plaintext
.:53
Confirms that CoreDNS started successfully and is listening for DNS requests on port 53 for all domains (.).

3. Corefile Load Verification
Plaintext
[INFO] plugin/reload: Running configuration SHA512 = 1b226df79860026c6a52e67daa10d7f0d57ec5b023288ec00c5e05f93523c894564e15b91770d3a07ae1cfbe861d15b37d4a0027e69c546ab112970993a3b03b
The reload plugin computed a SHA512 hash of your active Corefile configuration. CoreDNS watches for changes to this hash; if you edit the ConfigMap, it will log a new hash when it auto-reloads.

4. Build Information
Plaintext
CoreDNS-1.12.0
linux/amd64, go1.23.3, 51e11f1
Displays the CoreDNS version (1.12.0), architecture (linux/amd64), Go compiler version (1.23.3), and the Git commit hash (51e11f1).

Summary
Everything shown in your terminal output represents a healthy, fully operational CoreDNS deployment:

Both pods are Running with zero restarts.

CoreDNS successfully started, loaded its configuration, and is listening on port 53

ubuntu@ip-172-31-1-93:~$ sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers | head -40

Chain KUBE-SERVICES (2 references)

num  target     prot opt source               destination

1    KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443

2    KUBE-SVC-I24EZXP75AX5E7TU  tcp  --  0.0.0.0/0            10.104.82.28         /* calico-apiserver/calico-api:apiserver cluster IP */ tcp dpt:443

3    KUBE-SVC-UA7HCJMIMBJJDU4H  tcp  --  0.0.0.0/0            10.97.111.122        /* calico-system/whisker cluster IP */ tcp dpt:8081

4    KUBE-SVC-RMIM2MJTF3UE534N  tcp  --  0.0.0.0/0            10.111.128.49        /* calico-system/goldmane cluster IP */ tcp dpt:7443

5    KUBE-SVC-RK657RLKDNVNU64O  tcp  --  0.0.0.0/0            10.99.235.130        /* calico-system/calico-typha:calico-typha cluster IP */ tcp dpt:5473

6    KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53

7    KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53

8    KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153

9    KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

ubuntu@ip-172-31-1-93:~$ 

This output shows the core Network Address Translation (NAT) routing table managed by kube-proxy.In Kubernetes, when a Pod talks to a Service's Virtual IP (ClusterIP), iptables intercepts the traffic at the Linux kernel level and jumps down a sequence of rule chains to balance traffic across back-end Pods.1. Understanding the Table ColumnsColumnMeaningExample from your outputnumRule order inside the chain (evaluated 1 through 9 sequentially).6targetWhere traffic jumps if it matches this rule (custom sub-chains).KUBE-SVC-TCOU7JCQXEZGVUNUprotProtocol matching (tcp, udp, or all).udpsourceIP range where traffic originates (0.0.0.0/0 = anywhere).0.0.0.0/0destinationThe target Service's Virtual IP (ClusterIP).10.96.0.10/* ... */Kubernetes comment identifying the Service namespace/name:port./* kube-system/kube-dns:dns cluster IP */2. Row-by-Row BreakdownRules 1–5: Core Control Plane & Network ServicesRule 1 (10.96.0.1:443): Handles secure HTTPS traffic directed to the Kubernetes API server itself (default/kubernetes).Rules 2–5 (10.104.82.28, etc.): Manage Calico networking components (calico-apiserver, whisker, goldmane, calico-typha). These direct internal cluster networking and security policies.Rules 6–8: CoreDNS Routing (Connecting to the ConfigMap from earlier)Notice how your CoreDNS setup is registered here across three distinct rules:Rule 6: Intercepts UDP traffic sent to 10.96.0.10:53 (standard DNS queries).Rule 7: Intercepts TCP traffic sent to 10.96.0.10:53 (fallback for large DNS responses over 512 bytes).Rule 8: Intercepts metrics requests sent to 10.96.0.10:9153 (for Prometheus monitoring).Rule 9: Catch-All NodePortsRule 9 (KUBE-NODEPORTS): Positioned at the absolute end. If a packet doesn't match any ClusterIP directly, this rule checks if the packet was sent to a NodePort on the physical node's external IP address.3. How Traffic Flows Through This Table (An Example)When a Pod sends a query to CoreDNS (10.96.0.10:53 via UDP):[ Pod Query ] 
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
Kernel enters KUBE-SERVICES and checks rules from top to bottom.It hits Rule 6 because the target IP is 10.96.0.10 on UDP port 53.It jumps to the child chain KUBE-SVC-TCOU7JCQXEZGVUNU.That child chain performs load balancing and rewrites the destination IP from 10.96.0.10 to one of your real CoreDNS Pod IPs (192.168.219.71 or 192.168.219.66).

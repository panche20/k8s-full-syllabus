# Day 19 — Senior K8s Interview Mastery

This is the day that turns knowledge into a job. Every question here is pulled from real interviews at Google, Stripe, Booking.com, Revolut, Grafana Labs, and other top European tech companies. No fluff — just the exact questions, the exact answers interviewers want to hear, and the traps to avoid.

## 🧠 The Interview Mental Model

*Senior K8s interviews test three things simultaneously:*

```
Depth      — do you know WHY not just HOW
Breadth    — can you connect networking to security to storage
Production — have you actually operated this at scale, or just read about it
```

*Every answer should follow this structure:*

```
1. Direct answer (10 seconds)
2. The mechanism (30 seconds)
3. Production nuance or gotcha (20 seconds)
```

## 🔥 Section 1: Architecture Questions

**Q1: Explain how Kubernetes achieves high availability for the control plane.**

The API server is stateless — you can run 2-5 replicas behind a load balancer. All state lives in etcd, which uses the Raft consensus algorithm requiring a quorum of (n/2)+1 nodes — so 3 etcd nodes tolerate 1 failure, 5 nodes tolerate 2. The scheduler and controller-manager use leader election — multiple replicas run but only one is active at a time, elected via a Lease object in etcd. On EKS/GKE/AKS the control plane is fully managed — you never see it. On self-managed you need: odd number of etcd nodes (3 or 5), API server behind a load balancer, and etcd backups on a schedule. Production gotcha: etcd is extremely sensitive to disk latency — put it on NVMe SSDs and never share the disk with other workloads.

**Q2: What happens in the cluster when a node running 10 pods suddenly loses network connectivity?**

The node's kubelet stops sending heartbeats to the API server. After node-monitor-grace-period (default 40 seconds) the node status goes Unknown. The node controller adds two NoExecute taints: node.kubernetes.io/unreachable and node.kubernetes.io/not-ready. Pods without tolerations for these taints are evicted after pod-eviction-timeout (default 5 minutes). The Deployment controller creates replacement pods on healthy nodes. StatefulSet pods are NOT rescheduled — K8s deliberately waits for the node to return to prevent split-brain on stateful workloads. If the node never returns, an operator must manually delete the Node object to unblock StatefulSet rescheduling. Production nuance: this 5-minute window is tunable but shrinking it too much causes false evictions during normal network blips.

**Q3: How does the Kubernetes API server handle 10,000 concurrent watch requests without falling over?**

The API server uses HTTP/2 multiplexing for watches — many watches share a single connection. Watches are implemented as long-lived HTTP GET requests with ?watch=true. When an object changes in etcd, etcd sends a watch event to the API server, which fans it out to all interested watchers. The watch cache (WatchCache) in the API server stores a ring buffer of recent events — most watch clients get served from this cache without hitting etcd, dramatically reducing etcd read pressure. Production nuance: the --watch-cache-sizes API server flag lets you tune cache size per resource type. Under heavy load, the API server uses ResourceVersion to let clients resume watches after reconnection without replaying all history.

**Q4: Explain etcd's Raft consensus and what happens during a leader election.**

Raft divides time into terms. Each term starts with a leader election: a follower that hasn't heard from the leader starts an election, votes for itself, and requests votes from peers. A candidate needs a majority vote to become leader. The new leader sends heartbeats to suppress future elections. All writes go to the leader — it appends to its log, replicates to followers, and commits when a majority acknowledge. If the leader dies, the follower with the most up-to-date log wins the next election. Production nuance: etcd requires stable low-latency network between members — RTT above 10ms between nodes starts causing election timeouts and instability. This is why you never stretch a single etcd cluster across regions.

## 🌐 Section 2: Networking Deep Dives

**Q5: A pod can curl google.com but can't reach another pod by service name. Trace every possible failure point.**

DNS first: exec into the pod, check /etc/resolv.conf points to CoreDNS ClusterIP. Run nslookup <service-name> — if it fails, CoreDNS is the problem. Check CoreDNS pods are running and healthy. Check the CoreDNS ConfigMap for misconfigurations. If DNS resolves but connection fails: check kubectl get endpoints <service> — empty means pods aren't Ready or selector is wrong. Check if the targetPort matches what the container is actually listening on. Check NetworkPolicy — is egress to that service allowed? Check if kube-proxy is running on the node and its iptables rules are intact. Check if the Service's sessionAffinity is sending you to a broken pod. Final check: bypass the service and curl the pod IP directly — if that works, the Service or kube-proxy is the problem.

**Q6: Explain the packet journey from an external user hitting a LoadBalancer Service to an app pod.**

User hits the cloud load balancer IP. Cloud LB forwards to one of the node IPs on the NodePort (30000-32767). On the node, iptables PREROUTING chain intercepts the packet at the KUBE-SERVICES chain, matches the NodePort, and DNAT's it to one of the pod IPs in the KUBE-SVC chain. If the selected pod is on a different node, the packet is encapsulated (or routed, depending on CNI) to that node. The pod receives the packet appearing to come from the node IP (SNAT'd), processes it, and the response traverses the same path in reverse. Production nuance: externalTrafficPolicy: Local skips the final hop — the LB only routes to nodes that have an endpoint locally — preserving the client's real source IP but causing uneven load distribution.

**Q7: What is CNI and how does Calico implement the Kubernetes networking model differently from Flannel?**

CNI is a specification that kubelet calls when creating or deleting pods — it delegates IP assignment and network wiring to a plugin. Flannel uses VXLAN overlay — it encapsulates pod packets in UDP packets between nodes. This adds overhead (50-70 bytes per packet) and requires kernel VXLAN support. It's simple but doesn't support NetworkPolicy — Flannel just makes a flat network. Calico can operate in two modes: VXLAN overlay (like Flannel) or BGP routing, where each node is a BGP peer and routes pod CIDRs natively at L3 without encapsulation. Calico also has its own dataplane for NetworkPolicy enforcement using iptables or eBPF. Production nuance: on cloud providers, Calico BGP mode requires the cloud network to support BGP peering — AWS VPC doesn't natively, so you use VXLAN or AWS VPC CNI instead.

**Q8: How does kube-proxy use iptables and what's the performance problem at scale?**

kube-proxy programs iptables rules for every Service and Endpoint. For each packet destined for a ClusterIP, iptables traverses a chain of rules linearly until a match. With 1,000 Services each with 5 endpoints, that's potentially 5,000 rules traversed per packet. iptables rule evaluation is O(n) — performance degrades linearly with rule count. Above ~10,000 endpoints, latency becomes measurable. The solutions: IPVS mode — kube-proxy uses the kernel's IPVS hash table for O(1) lookups regardless of rule count. Cilium replaces kube-proxy entirely with eBPF maps, which are also O(1) and bypass iptables entirely. Production nuance: switching from iptables to IPVS mode requires ipvs kernel module and ipvsadm on all nodes.

## 💾 Section 3: Storage Deep Dives

**Q9: A StatefulSet with 3 replicas — explain exactly what happens to storage when pod-1 is deleted.**

The Pod is deleted and immediately recreated with the same name (db-1) by the StatefulSet controller. The PVC data-db-1 is NOT deleted — it's owned by the StatefulSet template, not the Pod. The new db-1 pod is bound to the same data-db-1 PVC. If the PVC was ReadWriteOnce, it must unmount from the old pod before mounting to the new one — this is why pod termination grace period matters for StatefulSets. The data is completely preserved. This is fundamentally different from a Deployment where pods are interchangeable and have no persistent identity. Production nuance: if the node that db-1 ran on is dead, the PVC may be stuck in Terminating because the kubelet can't unmount it. You sometimes need to manually force-delete the pod AND the VolumeAttachment object.

**Q10: What is volumeBindingMode: WaitForFirstConsumer and when does not using it cause problems?**

With the default Immediate binding, a PVC triggers PV creation (via StorageClass dynamic provisioning) the moment it's created — in a random or default availability zone. When the scheduler places the pod, it must go to the same AZ as the PV. If your cluster spans 3 AZs and the PV landed in us-east-1a, the pod must too — even if us-east-1a is overloaded or the pod has affinity rules preferring another zone. With WaitForFirstConsumer, the PVC stays Pending until a pod requests it. The scheduler picks the best node for the pod first, then the PV is provisioned in that node's AZ. This decouples scheduling from storage provisioning. Production nuance: you must ensure your StorageClass has this mode set on AWS EBS, GCP PD, and Azure Disk — they're all single-AZ storage. EFS/NFS which span AZs don't need it.

## 🔐 Section 4: Security Deep Dives

**Q11: A developer says they need cluster-admin for their CI/CD pipeline. How do you respond?**

I'd ask what specific actions the pipeline needs to perform — create Deployments, update ConfigMaps, read Secrets — and build the minimal RBAC that covers exactly those actions. cluster-admin in a CI/CD pipeline is a critical security risk: if the pipeline is compromised (malicious PR, stolen token), the attacker has full cluster access. Instead: create a ServiceAccount per environment, give it only update on deployments and create on configmaps in the target namespace, use short-lived tokens via IRSA or Workload Identity. Audit the SA's permissions quarterly. If they truly need broad access (creating new CRDs, managing RBAC), create a scoped ClusterRole that allows exactly those resources, not cluster-admin. Production nuance: pipelines should never have delete on namespaces — a misconfigured pipeline could wipe production.

**Q12: Explain the attack surface of a privileged pod and how an attacker escapes to the host.**

A privileged pod has privileged: true in securityContext, which gives it all Linux capabilities and disables seccomp and AppArmor. From inside: the attacker can mount the host filesystem (mount /dev/sda1 /mnt), read host SSH keys, read other containers' filesystems via /proc/<pid>/root, and write cron jobs to the host. They can also load kernel modules, manipulate iptables directly, or exploit the container runtime via /var/run/docker.sock if it's mounted. The escape sequence: nsenter --target 1 --mount --uts --ipc --net --pid -- bash — this enters all host namespaces and gives a root shell on the host. Prevention: never run privileged containers, use PSA to enforce it cluster-wide, run Falco to detect privilege escalation attempts at runtime.

**Q13: How does Kubernetes manage ServiceAccount token security after version 1.24?**

Before 1.22: long-lived static tokens auto-created as Secrets — valid forever, available to anyone with Secret read access. From 1.22: projected volumes with bound service account tokens — short-lived (default 1 hour), audience-bound (only valid for the K8s API), automatically rotated by kubelet before expiry, and tied to the pod's lifetime. The token is only accessible inside the pod via the projected volume, not as a cluster-wide Secret. From 1.24: the old auto-created token Secrets are no longer generated. If you need a long-lived token (for external tools), you must explicitly create a Secret of type kubernetes.io/service-account-token — this is a deliberate friction to discourage long-lived tokens. Production nuance: third-party tools that relied on auto-created token Secrets broke at 1.24 — check your external tools before upgrading.

## 📊 Section 5: Observability

**Q14: Your P99 latency spiked at 2am. How do you investigate using the observability stack?**

Start with metrics in Grafana — pull up the HTTP request duration histogram for that time range, look at P50/P95/P99. Identify which service spiked. Correlate with infrastructure metrics — did CPU throttling spike? Memory approach limits? Node CPU spike? Check HPA events — did the service scale down at 2am (reduced replicas → more load per pod)? Check for pod restarts — a rolling restart or CrashLoop at 2am would cause latency spikes as requests hit restarting pods. Switch to Loki — filter logs for that service in that time window, look for slow query logs, timeout errors, connection pool exhaustion. If tracing is set up, find a slow trace in Jaeger from that window — it shows exactly which service hop consumed the 1.9 seconds. Production nuance: always check deployment history — kubectl rollout history — a 2am deployment is the most common cause of latency spikes.

**Q15: What is a Prometheus recording rule and when do you need one?**

A recording rule pre-computes an expensive PromQL query and stores the result as a new metric at each scrape interval. For example: sum by (namespace) (rate(http_requests_total[5m])) computed across 50 services with 10,000 pods each becomes expensive when Grafana runs it on every dashboard refresh for every user. A recording rule computes it once every 15 seconds and stores namespace:http_requests_total:rate5m — dashboards then query the pre-computed metric instantly. Use recording rules for: queries used in multiple dashboards, queries with many-to-one aggregations across thousands of series, queries used in alerts (alerting rules run frequently — expensive queries slow alert evaluation). Production nuance: recording rule names must follow the convention level:metric:operations — this is enforced by some linting tools and makes rules discoverable.

## ⚙️ Section 6: Platform Engineering

**Q16: How do you implement multi-tenancy on a shared Kubernetes cluster?**

Several layers work together. Namespace isolation: each team gets dedicated namespaces. RBAC: team members get namespace-scoped roles, never cluster-level. ResourceQuota: hard limits on CPU, memory, pods per namespace — prevents one team from starving others. LimitRange: default requests/limits so quota accounting works. NetworkPolicy: default-deny between namespaces — teams can't talk to each other unless explicitly allowed. PSA: enforce baseline or restricted on all tenant namespaces. For stronger isolation: use separate node pools per tenant with taints/tolerations, so workloads don't share kernel-level resources. Hierarchical namespaces (HNC controller) for nested team structures. Production nuance: soft multi-tenancy (namespaces + RBAC) is sufficient for trusted internal teams. For untrusted tenants (SaaS customers), use separate clusters or Kata Containers for VM-level isolation — containers on a shared kernel are not truly isolated.

**Q17: Design the deployment pipeline for a microservices application with 20 services.**

Git is the source of truth for both app code and K8s config. Each service has its own repo (or directory in a monorepo) with a Dockerfile and Helm chart. CI (GitHub Actions): on PR — run tests, trivy scan, helm lint. On merge to main — build image, tag with git SHA, push to ghcr.io, update the image tag in the GitOps repo via PR. GitOps repo: separate repo with ArgoCD ApplicationSet managing all 20 services. ArgoCD detects the tag update and syncs automatically. Environments: dev auto-deploys from main. Staging requires a passing integration test suite. Production requires manual approval via ArgoCD PR. Rollback: git revert the tag change — ArgoCD reverts the deployment. For coordinated releases across multiple services: use ArgoCD sync waves to order them. Production nuance: avoid coupling service releases — each service should deploy independently. If 20 services must all deploy together, that's a monolith disguised as microservices.

**Q18: A team reports their pods are being evicted every night at 2am. Walk me through diagnosing it.**

Check eviction events: kubectl get events -A --field-selector=reason=Evicted — this shows which pods and why. Check if it's memory or disk pressure: kubectl describe nodes around 2am (check Prometheus node memory metric history). If memory: identify what runs at 2am — CronJobs, batch jobs, scheduled backups. Check if a CronJob creates a memory spike that triggers eviction of other pods. Check pod QoS classes — BestEffort pods evict first. A batch job with no resource limits (BestEffort) consuming memory causes K8s to evict other BestEffort pods to protect Guaranteed ones. Fix: add resource requests/limits to the CronJob and any other BestEffort pods. Consider giving the batch job a low-priority PriorityClass so it's evicted first if there's pressure. Production nuance: also check if a node is being drained at 2am by an automated maintenance script.

## 🏗️ Section 7: System Design Questions

**Q19: Design a Kubernetes platform for a fintech company with strict data residency requirements (EU data must stay in EU).**

Multi-cluster architecture: separate clusters per region (eu-west-1, eu-central-1, us-east-1). Never replicate EU data outside EU clusters. GitOps: ArgoCD running in each region, syncing from region-specific branches or paths in the GitOps repo. Secrets: Vault deployed per region, EU Vault never syncs secrets to US. Networking: no cross-region pod communication — services communicate via regional API gateways with explicit data classification. Ingress: Cloudflare with geo-routing rules — EU users route to EU clusters. Database: Postgres per region with no cross-region replication for PII tables. Audit: audit logs per cluster shipped to regional SIEM — EU audit logs stay in EU. Compliance controls: OPA Gatekeeper policies enforce that pods in EU namespaces only reference EU-region storage classes and secrets. Monitoring: Grafana Cloud with data residency enabled, or self-hosted Grafana per region. Production nuance: the hardest part is developer experience — engineers need to test their service in the right region. Use feature flags to separate EU-specific logic and run integration tests against EU clusters from the start.

**Q20: How would you design a zero-downtime migration from a self-managed cluster to EKS?**

Phase 1 — parallel run: provision EKS alongside the existing cluster. Set up ArgoCD on EKS pointing to the same GitOps repo. Deploy all workloads to EKS. Run in shadow mode — real traffic still goes to old cluster, EKS runs but receives no external traffic. Phase 2 — stateless services: migrate stateless services first. Update DNS with low TTL (60 seconds). Use weighted routing (Route53 weighted records): 5% to EKS, monitor for errors, gradually increase to 100%. Rollback is instant — adjust DNS weights. Phase 3 — stateful services: migrate databases last and carefully. Use read replicas in EKS, promote to primary, update app connection strings. For caches: warm the new Redis before cutting over. Phase 4 — decommission: once 100% traffic on EKS for 2 weeks with no incidents, decommission old cluster. Production nuance: the migration exposes every implicit assumption your apps make — hardcoded IPs, node-level configuration, non-portable PVs. Budget 2-3x more time than estimated.

## ⚡ Section 8: Speed Round — 30 Questions, 30 Seconds Each

*Answer each without thinking more than 30 seconds. These are the rapid-fire questions that test fluency.*

**21. What's the default grace period for pod termination?**

30 seconds. Configurable via terminationGracePeriodSeconds.

**22. How do you run a one-off command in a running pod without exec?**

kubectl debug with a copy, or kubectl run --rm -it for a new pod.

**23. What does kubectl apply do differently from kubectl create?**

apply is declarative — merges changes, creates if not exists. create fails if resource exists.

**24. How do you see which image a running pod is using?**

kubectl get pod <name> -o jsonpath='{.spec.containers[*].image}'

**25. A Deployment has revisionHistoryLimit: 0. What breaks?**

kubectl rollout undo no longer works — no old ReplicaSets are kept.

**26. What is a headless service?**

clusterIP: None — no virtual IP. DNS returns individual pod IPs. Required for StatefulSets.

**27. How do you force a ConfigMap update to restart pods?**

Add a checksum/config annotation in the Deployment template that hashes the ConfigMap content — changes trigger rolling restart.

**28. What is the difference between Recreate and RollingUpdate deployment strategy?**

Recreate kills all pods then creates new ones — causes downtime but avoids running two versions simultaneously. RollingUpdate replaces pods gradually — zero downtime but two versions run briefly.

**29. How do you prevent a specific pod from being evicted during node pressure?**

Set priorityClassName to a high-priority class with high value, or set QoS to Guaranteed (requests == limits).

**30. What does kubectl rollout pause do?**

Stops the rolling update mid-way — new pods keep the old image. Useful for canary validation before continuing.

**31. How many etcd nodes do you need for a cluster that tolerates 2 simultaneous failures?**

5 nodes. Quorum requires 3 of 5.

**32. What is the projected volume type?**

Combines multiple sources (ConfigMap, Secret, ServiceAccountToken, DownwardAPI) into one volume mount.

**33. How do you check what RBAC permissions a user has?**

kubectl auth can-i --list --as=<username> -n <namespace>

**34. What does kubectl diff -f manifest.yaml do?**

Shows the diff between the manifest and the live cluster state — without applying anything.

**35. What is a PodDisruptionBudget?**

Guarantees minimum available pods during voluntary disruptions (drain, rolling update). minAvailable or maxUnavailable.

**36. How do you copy a file from a pod to your local machine?**

kubectl cp <namespace>/<pod>:/path/to/file ./local-file

**37. What is kubectl port-forward actually doing?**

Creates a tunnel from your local port through the API server to the pod's port. Traffic goes: localhost → API server → pod.

**38. Why can't you change a Deployment's selector after creation?**

Selectors are immutable. Changing them would orphan existing pods (no longer selected) and the new selector might accidentally adopt other pods.

**39. What happens when you set requests.cpu: 0?**

The pod is treated as BestEffort for CPU — it gets whatever CPU is available, no guarantee. First to be throttled under pressure.

**40. How do you run a pod on every node including the control plane?**

Use a DaemonSet with a toleration for node-role.kubernetes.io/control-plane:NoSchedule.

**41. What is an EndpointSlice vs an Endpoint?**

EndpointSlice shards endpoint data into 100-endpoint chunks — reduces API server load when services have many backends. Endpoints is the older single-object approach.

**42. How do you make a service sticky — same client always routes to same pod?**

service.spec.sessionAffinity: ClientIP — uses client IP for sticky routing. Or use Istio's consistentHash in DestinationRule for cookie-based stickiness.

**43. What is downwardAPI volume?**

Exposes pod metadata (labels, annotations, resource limits) as files inside the container — without calling the K8s API.

**44. Difference between kubectl delete pod and kubectl delete pod --force?**

Normal delete: sends SIGTERM, waits gracePeriod, then SIGKILL. Force delete: skips grace period, removes from API immediately. The container may keep running briefly if kubelet is slow.

**45. What is a readinessGate?**

Adds custom conditions to pod readiness — the pod is only Ready when all readinessGates pass. Used by Ingress controllers and service meshes to hold traffic until the sidecar is ready.

**46. How do you see the resource limits of a node?**

kubectl describe node <name> | grep -A 5 "Allocatable"

**47. What is Lease in Kubernetes?**

A lightweight object used for leader election by control plane components (scheduler, controller-manager) and kubelet heartbeats. Located in coordination.k8s.io API group.

**48. What does imagePullPolicy: IfNotPresent do?**

Skips pulling the image if it's already on the node. Risk: if you push a new image with the same tag, nodes with the old image cached won't pull the update. Always use unique tags.

**49. How do you get the logs of a previous container in a pod?**

kubectl logs <pod> -c <container> --previous

**50. What is the maximum number of pods per node by default in Kubernetes?**

110 pods per node. Configurable via --max-pods in kubelet config. AWS EKS limits are lower based on instance type and ENI limits.

## 🎭 Part 3: Behavioral Questions — Senior Level

*These separate senior engineers from mid-level.*

**B1: Tell me about a time Kubernetes caused a production incident and how you resolved it.**

**Structure your answer:**

```
Context    — what was running, what was the blast radius
Detection  — how you found out (alert, user report)
Diagnosis  — what tools you used, what you found
Resolution — what you changed, how long it took
Prevention — what you added to prevent recurrence
```

Example angle: etcd disk pressure causing API server slowness → diagnosed via Prometheus etcd metrics → freed disk → added disk space alert + etcd data pruning CronJob.

**B2: How do you handle a disagreement with a developer who wants to run their container as root?**

I'd understand their reason first — sometimes there's a legitimate constraint (legacy app, specific filesystem operation). Then explain the risk specifically: running as root means a container escape gives the attacker root on the node, access to all secrets on that node, and potentially the ability to pivot to the control plane. I'd offer alternatives: try runAsUser: 0 with allowPrivilegeEscalation: false and see what actually breaks; use an init container as root to set up filesystem permissions then switch to non-root for the main container; rebuild the image properly. If they still insist: escalate to security review, document the exception formally with a review date. Don't just block — help them find a path forward.

**B3: How do you stay current with Kubernetes given how fast it moves?**

*Strong answer hits these:*

- Track the Kubernetes changelog for each minor release (SIG release notes)
- Follow specific SIGs relevant to your work (sig-network, sig-storage, sig-security)
- CNCF landscape changes — new projects graduating
- Real learning via hands-on: building the topics you've been studying here
- Community: KubeCon talks (YouTube), K8s Slack, Twitter/X K8s community
- Certification maintenance: CKA/CKAD/CKS require renewal every 2 years — forces staying current

## 🖥️ Part 4: Live Coding Scenarios

**Scenario 1: Write from memory in 5 minutes**

Write a complete Deployment YAML for a production-grade web app with:

- 3 replicas, rolling update with zero downtime
- Resource requests and limits
- Liveness and readiness probes
- Non-root security context
- Config from ConfigMap, secret from Secret

```
# Timer: 5 minutes. No notes. Go.
# Then compare against this reference:

cat <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
      annotations:
        checksum/config: "{{ configmap-hash }}"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: ghcr.io/youruser/web-app:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]
        envFrom:
        - configMapRef:
            name: web-app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: DB_PASSWORD
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5
          failureThreshold: 2
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
EOF
```

**Scenario 2: Debug this broken manifest in 3 minutes**

```
# What is wrong with this? Find all issues.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app          # issue 1
  template:
    metadata:
      labels:
        app: broken-app    # MISMATCH with selector
    spec:
      containers:
      - name: app
        image: nginx:latest  # issue 2: mutable tag
        resources:
          limits:
            memory: "64Mi"   # issue 3: no requests — BestEffort for memory
          requests:
            cpu: "100m"      # issue 4: no memory request
        livenessProbe:
          httpGet:
            path: /health
            port: 8080       # issue 5: nginx listens on 80, not 8080
          initialDelaySeconds: 3
          failureThreshold: 1  # issue 6: too aggressive — restarts on first failure
        readinessProbe:
          httpGet:
            path: /health    # issue 7: same probe as liveness — should check deps
            port: 8080
```

**Issues:**

*selector/label mismatch (pod never selected), latest tag, no memory request (BestEffort), liveness port wrong (8080 vs 80), failureThreshold: 1 causes immediate restart on any blip, readiness path is same as liveness (should check dependencies).*


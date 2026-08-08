# Task 3 — Falco Runtime Detection

**Part A: The Framework (Falco as a topic)**

**The 'Why' (The Problem)**

*Before Falco (2016, Sysdig → CNCF graduated project), container security tooling lived at two extremes: image scanning (Trivy/Clair — catches known CVEs in a static image, tells you nothing about what happens after the container starts) and network policy / admission control (catches misconfiguration before a pod runs, tells you nothing once it's running). Nobody was watching the actual behavior of a running container in real time.*

*The gap: a container gets compromised via an app-layer RCE (a vulnerable library, an exposed debug endpoint). The attacker now has a shell inside a container that passed every scan and every admission check clean. Nothing in the pre-deploy pipeline sees this — you need something watching syscalls live, because that's the only ground truth that can't be faked by a well-formed manifest. Falco's a pain point: "my container is running exactly the process it's supposed to be running — how do I know if that stops being true?"*

**Deep-Dive Mechanics**

Falco taps the Linux kernel at the syscall boundary via one of three drivers:

- Kernel module (falco.ko) — oldest, fastest, but requires kernel headers match and taints the kernel.
- sysdig-probe eBPF driver (legacy) — no kernel module, but still needs kernel-version-specific compilation.
- Modern eBPF probe (CO-RE — Compile Once, Run Everywhere) — the current default. Built once against BTF (BPF Type Format) metadata, portable across kernel versions without recompiling. This is what removed most of the "why won't Falco start on this node" pain.

*Pipeline: syscall → driver captures raw event → libscap (capture) turns it into a stream → libsinsp (state engine) enriches it — attaches process tree, container ID → maps to Kubernetes pod/namespace (via container runtime metadata or, in modern setups, the k8smeta plugin talking to a lightweight k8s-metacollector, replacing the old pattern where every Falco pod hit the API server directly, which didn't scale past small clusters) → the rules engine evaluates each event against loaded YAML rules (conditions are boolean expressions over fields like proc.name, fd.name, container.image.repository) → matched rule fires an output (stdout, file, syslog, gRPC, or via Falcosidekick fan-out to Slack/PagerDuty/S3/etc.).*

*Critical architectural fact for what you're about to do: Falco is a DaemonSet — one pod per node, watching only that node's kernel. It has zero visibility into a node it isn't running on. This bites people constantly (see gotchas below).*

**The Alternative Landscape**

<img width="900" height="557" alt="image" src="https://github.com/user-attachments/assets/68040f99-690d-42cd-9ef8-946837807bf1" />

<img width="866" height="340" alt="image" src="https://github.com/user-attachments/assets/9a3ce963-841d-4ec8-834f-15c1599d7fc0" />

**Why Falco over the alternatives** in most orgs: maturity, the size of its community rule set (you get "Terminal shell in container" and "Contact K8s API Server" for free — you didn't write those), and CNCF graduation means procurement/security teams trust it. You'd reach for Tetragon specifically when you need prevention (block the exec, not just alert on it) with lower overhead — that's the "better way" below.

**Interview POV & Edge Cases**

This shows up as: "How would you detect a container escape / reverse shell / crypto-miner in a K8s cluster in real time?" — Falco is the expected answer, and the follow-up is always "what are the blind spots?" The senior-level gotchas:

- DaemonSet blind spot: Falco on node A cannot see anything on node B. If you're troubleshooting "why didn't my alert fire," the very first check is "was my test pod even scheduled on the node whose Falco pod I'm reading logs from?" This trips up almost everyone the first time.
- Sandboxed runtimes (gVisor/Kata): gVisor intercepts syscalls in userspace before they hit the host kernel — Falco (which taps the host kernel) sees a hollowed-out, mostly useless event stream for gVisor-sandboxed pods. This is a real production limitation, not a config error.
- Rule tuning vs. alert fatigue: "Contact K8s API Server From Container" is infamous for flooding SOC dashboards on managed K8s (AKS/EKS) because add-ons legitimately talk to the API server constantly. Interviewers want to hear that you know detection ≠ actionable alert — untuned Falco is worse than no Falco because it trains people to ignore it.
- Falco is detective, not preventive. It tells you after the shell was spawned. This is the classic interview trap: "so Falco stopped the attack?" — no, it didn't, and conflating detection with prevention is the wrong answer.
- Priority inflation: everything at CRITICAL/WARNING with no differentiation is an anti-pattern — same failure mode as panic: true logging everywhere.

**The 'Better Way' (Evolution)**

Tetragon (Cilium/eBPF-based) is the modern answer to "Falco but with teeth." It hooks LSM (Linux Security Module) points in-kernel, so it can deny the syscall before it executes — a true reverse-shell prevention control, not just a notification. Lower overhead than syscall-capture-and-userspace-evaluate because policy decisions happen in-kernel. The pattern maturing right now industry-wide is Falco (broad detection + huge rule library) feeding Falco Talon (auto-response: kill pod, cordon node, revoke token) for the known-bad cases, with Tetragon/KubeArmor layered in for a smaller set of always-block policies (e.g., "never allow nc to spawn in the payments namespace, full stop").

**Part B: The Hands-On Solution**

Setup note: I'll use a single nicolaka/netshoot pod for every trigger in this task — it has bash, curl, and nc all in one image, so you're not fighting missing binaries in a minimal image mid-exam.

```
kubectl run test-pod --image=nicolaka/netshoot --restart=Never -- sleep infinity
kubectl wait --for=condition=Ready pod/test-pod --timeout=60s
```

**1. Find the Falco pod and confirm rules are loading**

```
kubectl get pods -A | grep -i falco
```

Grab the namespace/name, then check the startup log for the rule-loading lines:

```
kubectl logs -n <falco-ns> <falco-pod> | grep -iE "falco version|loading rules|starting internal webserver|error"
```

You want to see something like:

```
Falco version: 0.4x.x
Falco initialized with configuration file: /etc/falco/falco.yaml
Loading rules from file /etc/falco/falco_rules.yaml
Loading rules from file /etc/falco/falco_rules.local.yaml
Starting internal webserver, listening on port 8765
```

If you instead see rule loading error or a schema validation failure, that's your actual finding for this checkbox — Falco can be running (pod Ready) while not enforcing (rules failed to parse), which is exactly the kind of "looks healthy, isn't" gap this exam item is testing for.

Gotcha: identify which node your test-pod lands on and match it to the Falco pod on that same node — the DaemonSet blind spot from Part A applies here directly.

```
kubectl get pod test-pod -o wide
kubectl get pods -n <falco-ns> -o wide   # match the NODE column
```

From here on, $FALCO_POD = the Falco pod co-located with test-pod.

**2. Trigger the three default rules, capture to /tmp/falco-alerts.txt**

Start tailing that specific Falco pod's logs to the file before triggering anything:

```
kubectl logs -f -n <falco-ns> $FALCO_POD --since=1m > /tmp/falco-alerts.txt &
LOGPID=$!
sleep 2
```

**a) Terminal shell in container**

```
kubectl exec -it test-pod -- bash
exit
```

Fires because a shell (bash) was spawned with a TTY attached where the parent is the container's entrypoint chain (runc/containerd-shim) — exactly what kubectl exec -it produces.

**b) Read sensitive file**

```
kubectl exec -it test-pod -- cat /etc/passwd
```

Real gotcha, worth knowing cold for the interview and for this exam: the current upstream Falco sensitive_files macro is /etc/shadow, /etc/sudoers, /etc/pam.conf, /etc/security/pwquality.conf — /etc/passwd is not in the default list (it was pulled years ago to cut noise, since /etc/passwd is world-readable and read constantly by benign tooling). A lot of tutorials still say "cat /etc/passwd to trigger Falco" — that's stale advice against the current ruleset. Verify what your specific build actually ships:

```
kubectl exec -n <falco-ns> $FALCO_POD -- grep -A6 "list: sensitive_file_names" /etc/falco/falco_rules.yaml
```

If /etc/passwd isn't listed, use /etc/shadow instead (netshoot runs as root by default, so the read succeeds):

```
kubectl exec -it test-pod -- cat /etc/shadow
```

This reliably fires "Read sensitive file untrusted" (priority WARNING). If your exam grader specifically checks for /etc/passwd in the alert output, add a one-line override (covered in the custom-rule section below — same append: true pattern) rather than assuming the base rule covers it.

**c) Contact K8s API Server from container**

```
kubectl exec -it test-pod -- curl -sk https://kubernetes.default.svc.cluster.local/api \
  --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

Gotcha: this rule keys off fd.sip.name — Falco correlating the connection's destination IP back to a DNS name it observed being resolved (kubernetes.default.svc.cluster.local), not a hardcoded IP. Curl the FQDN, don't just hit the ClusterIP numerically, or Falco has no name to match against and won't fire. Also — this rule explicitly excludes anything in kube-system and a long list of known K8s infra images (k8s_containers macro). Testing from a kube-system pod will silently never fire this rule; that's why we used a plain default-namespace pod.

Stop the capture:

```
kill $LOGPID
```

Confirm all three landed:

```
grep -E "Terminal shell in container|Read sensitive file untrusted|Contact K8S API Server" /tmp/falco-alerts.txt
```

**3. Write the custom rule to /tmp/custom-rule.yaml**

```
- rule: Netcat Utility Launched in Container
  desc: >
    Detects execution of nc, ncat, or netcat inside a container. The built-in
    "Netcat Remote Code Execution in Container" rule only fires when nc/ncat is
    launched with RCE flags (-e, -c, --exec). This rule deliberately has no flag
    restriction, so it also catches plain listeners, port scans, and exfil usage
    that the default rule misses.
  condition: >
    spawned_process
    and container
    and proc.name in (nc, ncat, netcat)
  output: >
    CRITICAL: netcat-family process launched inside container (container_name=%container.name image=%container.image.repository:%container.image.tag process=%proc.name command=%proc.cmdline user=%user.name pid=%proc.pid)
  priority: CRITICAL
  tags: [container, network, process, mitre_execution, T1059, T1071]
```

This is worth naming distinctly from the built-in rule (not overriding it) — Falco does last-one-wins on duplicate rule names, and if you reuse the built-in name you must re-declare the entire rule block or you silently lose fields.

**4. Apply it and prove it fires**

First, find out how this Falco install actually loads rules — don't guess:

```
kubectl exec -n <falco-ns> $FALCO_POD -- grep -A10 "^rules_file" /etc/falco/falco.yaml
```

If it's Helm-managed (most common — check with helm list -n <falco-ns>):

```
helm upgrade falco falcosecurity/falco -n <falco-ns> --reuse-values \
  --set-file customRules."custom-rule\.yaml"=/tmp/custom-rule.yaml
kubectl rollout status daemonset/falco -n <falco-ns>
```

The chart's ConfigMap checksum annotation should force a pod rollout automatically; if not, kubectl rollout restart daemonset falco -n <falco-ns>.

If it's a raw ConfigMap already mounted at /etc/falco/rules.d/ or referenced as a falco_rules.local.yaml key:

```
kubectl get cm -n <falco-ns>
kubectl edit cm <rules-configmap-name> -n <falco-ns>   # paste the rule in under the appropriate key
kubectl rollout restart daemonset falco -n <falco-ns>
```

Either way, confirm the new rule loaded:

```
kubectl logs -n <falco-ns> $FALCO_POD | grep -i "Netcat Utility"
```

**Prove it fires:**

```
kubectl logs -f -n <falco-ns> $FALCO_POD --since=1m >> /tmp/falco-alerts.txt &
LOGPID=$!
kubectl exec -it test-pod -- nc -lvp 4444 &
sleep 3
kill $LOGPID
grep "Netcat Utility Launched" /tmp/falco-alerts.txt
```

You should see a CRITICAL line with container_name, image, process=nc, and the full command= cmdline — matching the required output fields.

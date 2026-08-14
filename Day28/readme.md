That gap is real and specific: strong theoretical K8s knowledge doesn't automatically give you a diagnostic algorithm. Given your chaos labs and internals work, you already know what etcd/kubelet/scheduler do — what's missing is the habit of asking "which component emitted this error" before you start Googling the string. Let's fix that.

The Roadmap
#	Module	Layer
1	Mental model + anatomy of a K8s error (today)	Meta
2	Pod & container lifecycle failures	Container runtime
3	Scheduling & Pending pod failures	Scheduler
4	Networking & DNS failures	CNI/kube-proxy/CoreDNS
5	Storage failures	CSI/PV/PVC
6	Control plane failures	etcd/API server/controllers
7	Node-level failures	kubelet/OS/cgroups
8	Ingress/Gateway API & app-layer failures	L7
9	Observability-driven triage	Cross-cutting
10	Live incident simulations (SRE interview style)	Everything at once

Modules 6–7 will lean on your chaos labs directly. Let's start.

Module 1 — The Troubleshooting Mental Model & Anatomy of a K8s Error
The 'Why' (The Problem)

Debugging a monolith on 3 servers means SSH in, tail -f a log, done — one layer of indirection. Kubernetes has roughly eight: kubectl → API server → etcd → controllers → scheduler → kubelet → CRI → container runtime → cgroups/namespaces/kernel. A single symptom ("pod won't start") can originate at any of these layers, and each layer has its own vocabulary of failure.

The reason strong engineers stall here isn't lack of knowledge — it's lack of an algorithm. Without one, every error becomes a fresh Google search instead of a five-second triage. Senior SREs aren't reading errors faster because they've memorized more strings; they've internalized which component is allowed to say what, so a string like FailedScheduling instantly tells them "scheduler, not kubelet — don't waste time checking node disk pressure."

Deep-Dive Mechanics

1. Everything is a reconciliation loop. Every K8s error is fundamentally: desired state (spec, in etcd) ≠ observed state (status, reported by some component). Troubleshooting is just identifying which controller owns the mismatch.

2. The "From" field is the most important field you're ignoring. Run kubectl describe pod <name> and look at the Events table:

Type     Reason          Age   From               Message
----     ------          ---   ----               -------
Warning  FailedScheduling 30s  default-scheduler  0/3 nodes available...
Normal   Pulling          25s  kubelet            Pulling image "nginx:bad-tag"
Warning  Failed           20s  kubelet            Error: ErrImagePull

From tells you the emitting component. That alone routes you: default-scheduler → scheduling problem (resources/taints/affinity), kubelet → node-local problem (image, volume, CRI). Most people read Message first and skip From — reverse that habit.

3. Three separate status systems get conflated constantly:

Pod Phase: Pending / Running / Succeeded / Failed / Unknown — coarse, almost useless alone.
Container State: Waiting / Running / Terminated, each with a Reason (CrashLoopBackOff, ImagePullBackOff, ContainerCreating, OOMKilled) — this is where the real signal lives.
Pod Conditions: PodScheduled → Initialized → ContainersReady → Ready, each True/False/Unknown. A Ready: False with everything else True almost always means a failing readiness probe, not a crash.

4. Exit codes are a language of their own — check these before you check logs:

Code	Meaning	First move
0	Clean exit	Check if it's a Job/CronJob that isn't supposed to be long-running
1	Generic app error	logs --previous
137	128+9 = SIGKILL	Usually OOMKilled — check kubectl describe for OOMKilled, then resource limits
143	128+15 = SIGTERM	App didn't handle graceful shutdown before terminationGracePeriodSeconds
126	Not executable	Bad entrypoint/permissions in image
127	Command not found	Typo in entrypoint or wrong base image

5. Log discipline: kubectl logs <pod> shows the current container's logs. If it already crashed and restarted, that's the new attempt — you want kubectl logs <pod> --previous for the one that actually died. Init containers and sidecars need -c <container-name> explicitly. And Events are namespace-scoped and TTL'd (~1h by default) — if you're investigating something from an hour ago, the Events are already gone unless you're shipping them somewhere.

Quick-Reference: The Diagnostic Tree
Pod not healthy?
├─ Phase: Pending
│   └─ describe → Events → FailedScheduling
│       → resource requests vs node capacity, taints/tolerations,
│         node/pod affinity, unbound PVC
├─ Container Waiting: ContainerCreating (stuck)
│   → image pull latency, CNI not assigning IP, volume mount failure
├─ Container Waiting: ImagePullBackOff / ErrImagePull
│   → wrong tag/registry auth (imagePullSecrets)/egress blocked
├─ Container Waiting: CrashLoopBackOff
│   → describe for last exit code + logs --previous
│     (137=OOM, 1=app error, 126/127=entrypoint issue)
├─ Running but Ready: False
│   → readiness probe config vs actual app health endpoint/port
└─ Evicted
    → node conditions: DiskPressure/MemoryPressure/PIDPressure
The Alternative Landscape
Approach	Speed to root cause (unfamiliar error)	Setup cost	Works air-gapped / whiteboard	Signal depth
Raw kubectl describe/logs	Medium (needs the algorithm above)	None	Yes	Low — unstructured, single-pod scope
k9s TUI	Medium-fast	Low	Yes	Same data, faster navigation
Centralized logging (Loki/EFK)	Fast for cross-pod patterns	High	No	High — correlated across pods/time
eBPF flow visibility (Cilium Hubble, Pixie)	Fast for network-layer mysteries	Medium	No	Very high — packet/flow level, no app changes needed
AI copilots (k8sgpt, Robusta)	Fast for common patterns, weak on novel ones	Low	No	Medium — good first hypothesis, verify manually

You'd choose raw kubectl mastery first regardless of what you adopt later — it's the substrate every other tool is built on, and it's the only one that works when you're whiteboarding in an interview with no cluster access.

Interview POV & Edge Cases

Interviewers rarely test "do you know what CrashLoopBackOff means" — they test whether you have a repeatable process when they hand you an unfamiliar symptom. Expect: "A pod is stuck in Pending, walk me through your triage." They're grading the order of your commands, not just the destination.

Common gotchas senior engineers get tripped on:

Confusing Error (a generic reason from the container runtime, not Kubernetes) with an actual root cause — it's a prompt to go read logs --previous, not an answer.
Forgetting Events expire — investigating a stale incident with kubectl describe alone and finding nothing.
Treating Ready: False as a crash when the container is actually running fine but failing its readiness probe (classic self-inflicted outage during a deploy).
Not checking kubectl get events --sort-by=.lastTimestamp -A cluster-wide when the failing pod's own Events are empty — the real cause (e.g., a NetworkPolicy or ResourceQuota) can live outside the pod's own nameserver of events.
The 'Better Way' (Evolution)

Two real shifts happening in production troubleshooting: eBPF-based observability (Cilium Hubble, Pixie) gives you network/syscall-level visibility with zero app instrumentation, closing the "it's a CNI problem" blind spot raw kubectl can't see into. And AI-assisted RCA (k8sgpt, Robusta) ingests Events+logs+metrics and proposes a root-cause hypothesis — genuinely useful as a first pass, but treat it as a hypothesis generator, not ground truth, since it can confidently misattribute causes in novel failure modes.

Want to go straight into Module 2 (Pod & container lifecycle failures) next, or would you rather I build you a hands-on lab first — deliberately break 5–6 of these failure modes on your kubeadm cluster so you practice the diagnostic tree live?

Updated memory

Good — that's the same cluster you're already running chaos labs on, so no new setup needed. Here's the tutorial format I'd suggest, built specifically to train the algorithm from Module 1, not just pattern-recognition.

The Lab Format: Break → Predict → Diagnose → Fix → Debrief

Every lab follows five steps, in this order, every time:

Break — exact command/manifest to inject a real failure
Predict — before running any diagnostic command, write down which component (From field) you expect to complain, and why. This is the step almost everyone skips, and it's the one that actually builds the algorithm.
Diagnose — walk the diagnostic tree from Module 1 with real kubectl output
Fix — remediate it
Debrief — name the Reason/exit code, and note the adjacent error it's easy to confuse it with (this is the interview-differentiation muscle)
Module 1 Lab Pack (7 labs, mapped to the diagnostic tree)
Lab	Failure	Trains
1	Over-requested CPU/memory	FailedScheduling — resource path
2	Taint/toleration mismatch	FailedScheduling — scheduling constraints path
3	Bad image tag, no registry auth	ImagePullBackOff
4	Memory limit too low for workload	CrashLoopBackOff → exit 137 (OOM)
5	Bad entrypoint/app misconfig	CrashLoopBackOff → exit 1 vs 127
6	Wrong readiness probe port	Ready: False (running, not crashing)
7	Fill worker node disk	Evicted — node condition path

Let's do Lab 1 now as the template — run this on your master:

1. Break

bash
kubectl create deployment greedy-pod --image=nginx --replicas=1
kubectl set resources deployment greedy-pod --requests=cpu=100,memory=100Gi

(100 CPU cores / 100Gi memory — no single EC2 worker in a 2-node kubeadm lab has that.)

2. Predict (do this before step 3)
Which component do you expect to log the complaint — kubelet, scheduler, or controller-manager? Write your answer down.

3. Diagnose

bash
kubectl get pods
kubectl describe pod <greedy-pod-xxxx>

Look specifically at the From field and the Events Reason. You should see default-scheduler and FailedScheduling, with a message like 0/2 nodes are available: 2 Insufficient cpu, 2 Insufficient memory.

4. Fix

bash
kubectl set resources deployment greedy-pod --requests=cpu=100m,memory=128Mi

5. Debrief
FailedScheduling is always the scheduler, always pre-node-assignment — the pod never even reaches a kubelet, which is why kubectl logs on it will just error with "pod has no containers." That's your tell: if logs fails outright and describe shows no Node: assigned, you're still in the scheduler's territory, not the kubelet's.

Run Lab 1, and when you're ready send me your describe output (or just tell me what you predicted vs. saw) and I'll walk you into Lab 2. Want me to write out Labs 2–7 in full now, or go one at a time as you clear each one?

Added to memory

Good instinct — production doesn't page you politely one incident at a time either. Here's a full batch. I won't solve any of these up front; that defeats the point. Work through them however suits you: post your reasoning here like before and I'll critique it, actually reproduce one on your cluster and report what you find, or just think them through mentally first. Come back to any of them whenever — they'll keep.

📟 Incident 5 — P1

Rollout stuck, old and new pods coexisting for 40+ minutes

payments Deployment updated 40 minutes ago (image bump only, no other changes). kubectl rollout status never completes. kubectl get pods shows 3 old-version pods still Running and 3 new-version pods stuck 0/1 Running — not crashing, not restarting, just never becoming Ready. Old pods are still serving traffic fine. No alerts have fired because nothing is actually down.

📟 Incident 6 — P2

One specific customer's requests always fail; everyone else is fine

Support escalates: one enterprise customer reports 100% failure on every API call for the last hour. Every other customer, same endpoints, same service, works perfectly. The customer's requests are confirmed reaching your edge (visible in gateway access logs) but always come back with an error.

📟 Incident 7 — P1

Full outage at 3:14 AM, nobody touched anything

Total outage, ~6 minutes, self-resolved before anyone finished loading a dashboard. Deploy history: nothing in the last 5 days. No kubectl commands run by anyone during that window per audit log. It's the second time this exact thing has happened — both times roughly 90 days apart.

📟 Incident 8 — P2

Autoscaler added nodes, pods are still Pending

Traffic spike triggered the autoscaler correctly — 2 new nodes joined the cluster 4 minutes ago, both show Ready. But 6 pods from the affected Deployment are still Pending, and it's been long enough that this shouldn't still be scheduling delay.

📟 Incident 9 — P3

A nightly Job that took 2 minutes now takes 45, no changes in 2 weeks

nightly-reconciliation CronJob has run daily for months, historically ~2 minutes. Over the last ~10 days it's crept up — 8 min, then 15, then last night 45 minutes. No code deploys, no config changes, no notable growth in the input dataset per the team that owns it.

📟 Incident 10 — P2

One pod cycles: Scheduled → Running → Evicted, repeatedly, on both workers

A specific pod in a batch-processing Deployment gets scheduled, runs for 2-3 minutes, gets evicted, immediately reschedules onto the other worker, runs 2-3 minutes, evicted again — bouncing back and forth for the last hour. Every other pod on both nodes is completely stable throughout.

📟 Incident 11 — P1

One namespace can't reach anything outside the cluster; everything internal is fine

Pods in the data-pipeline namespace can resolve DNS fine and reach every other in-cluster Service without issue. Every attempt to reach anything external (an S3 endpoint, an external API) times out. Every other namespace on the same nodes has normal external connectivity right now.

📟 Incident 12 — P3

Prometheus itself stops scraping the moment you actually need it

During the biggest incident of the quarter, someone opens Grafana to check checkout's dashboards — every panel says "No data" for the last 20 minutes. The incident being investigated is unrelated to Prometheus itself.

Rapid round — shorter, for pattern-recognition reps:

13. A Job's pod succeeds and exits 0, but the Job object itself never shows Complete.
14. kubectl exec into a pod works fine; the app inside reports it can't connect to its own sidecar on localhost.
15. A Deployment scaled to 10 replicas; only 7 ever come up, no errors anywhere, ResourceQuota isn't mentioned by anyone on the team.
16. After a routine node AMI upgrade, every pod using a specific PVC fails to mount — Multi-Attach error, even though nothing was ever scaled up.
17. A cron-triggered batch job works fine 29 days a month; fails every time it lands on the last day of a month.
18. kubectl get pods is slow — genuinely slow, several seconds — but only for one specific namespace.

Whenever you're ready to dig into one — any order, any number at once — just say the incident number and walk me through your reasoning the way you did for #4.

Can you please share solutions from incident 5 onwards till incident 18. I would go through these tonight and master these concepts!
Pls share all the solutions

Systematically diagnosed fourteen Kubernetes incidents using established patterns

Fair enough — you asked for volume, here's volume. Same format throughout: root cause, how you'd confirm it, the fix, and the one-line transferable lesson.

5 — Rollout stuck, old pods still serving
Cause: New pods are failing their readiness probe — likely the new image changed startup time, port, or health-check path. Deployment's rollout is readiness-gated: it won't scale down old pods until new ones pass Ready, so nothing pages because old pods keep serving the whole time.
Confirm: kubectl describe pod <new-pod> → look for repeated Unhealthy readiness events; kubectl logs <new-pod> to see if the app even started.
Fix: Correct the probe config to match the new image's real health endpoint/port (or fix the app if it regressed). kubectl rollout undo as immediate mitigation while you sort out root cause.
Lesson: A stuck rollout with old pods still healthy is a readiness problem on the new pods — check probe config diffs alongside image diffs.

6 — One customer fails 100%, everyone else fine
Cause: Usually not customer-specific logic — it's this customer's requests consistently landing on one unhealthy backend pod via session affinity/consistent hashing, or their unique request shape (large payload, unusual header/auth format) tripping a size or validation limit nobody else hits.
Confirm: Check gateway/Envoy access logs for the actual upstream response code and which pod served their requests specifically; check for session affinity config on the Service/HTTPRoute.
Fix: Depends on branch — fix/replace the specific bad backend, or adjust the limit/webhook rule tripped by their request shape.
Lesson: "One customer always fails" often means "this customer always lands on the one broken pod," not a customer-specific bug — confirm where it fails before assuming app logic.

7 — 3:14 AM outage, no human action, recurs ~90 days apart
Cause: The ~90-day period is the whole clue — that's Let's Encrypt's certificate lifetime. cert-manager's automatic renewal is rotating a cert, and the proxy isn't hot-reloading the new Secret cleanly, causing a brief gap.
Confirm: kubectl get certificate -A — check renewal timestamps against outage times; check cert-manager and Envoy Gateway logs/events for activity at 3:14 AM.
Fix: Ensure the proxy watches and reloads the TLS Secret without requiring a restart, or adjust cert-manager's renewal timing/buffer.
Lesson: "Nobody touched anything" + fixed periodicity = an automated system, almost always. Check everything on a schedule (cert renewal, token rotation, cron) before treating it as unexplainable.

8 — New nodes Ready, pods still Pending
Cause: kubectl get nodes showing Ready only reflects kubelet's basic health check — some CNIs (Cilium notably) apply their own taint (e.g. node.cilium.io/agent-not-ready:NoSchedule) until their own networking agent DaemonSet is actually functional on that node, deliberately blocking scheduling until real pod networking works.
Confirm: kubectl describe node <new-node> → check Taints; kubectl describe pod <pending-pod> → FailedScheduling citing an untolerated taint.
Fix: Wait for the CNI agent pod to become Ready (taint self-removes); if stuck, diagnose why the CNI agent itself won't start on the new node (Module 2/4).
Lesson: "Node Ready" ≠ "node ready for pods" — some CNIs taint new nodes on purpose to prevent exactly this race.

9 — Nightly job creeps from 2 min to 45 min, no code/data changes
Cause: EBS gp2 burst-credit exhaustion — burstable IOPS volumes deplete a credit balance under sustained I/O; once exhausted, performance drops to a much lower baseline, invisible anywhere in Kubernetes.
Confirm: CloudWatch BurstBalance metric on the volume — trending toward 0% matching the slowdown.
Fix: Move to gp3 (flat provisioned baseline, no credit system) or provision more IOPS.
Lesson: Cloud-provider burstable-performance resources degrade silently with zero Kubernetes-visible signal — "no code changes" doesn't rule out infrastructure decay.

10 — One pod bounces Scheduled→Evicted between both nodes, everything else stable
Cause: This one pod's resource request is under-sized relative to its real usage (Burstable QoS) — wherever it lands, it eventually ranks worst in eviction ordering under normal fluctuation, gets evicted, reschedules on the other node, repeats.
Confirm: kubectl top pod vs its requests; check Evicted reason on prior instances.
Fix: Right-size the request to match real usage.
Lesson: One pod bouncing while everything else is stable is almost always that pod's own request-vs-usage mismatch, not cluster-wide pressure.

11 — One namespace has no external egress, internal fine, other namespaces on same nodes fine
Cause: A NetworkPolicy scoped to that namespace with a default-deny egress rule that never allowlisted external CIDRs — since other namespaces on the same nodes are unaffected, this rules out node/CNI/security-group causes immediately.
Confirm: kubectl get networkpolicy -n data-pipeline — check egress rules for missing external CIDR allowance.
Fix: Add an explicit egress rule for required external ranges/ports.
Lesson: Same default-deny trap as Module 4, but namespace-scoped — "other namespaces fine, same nodes" should point you straight at NetworkPolicy, not infrastructure.

12 — Prometheus goes dark during the actual incident
Cause: The incident itself (mass pod restarts, label churn) is spiking Prometheus's own time-series cardinality, pushing it over its own memory limit right when it's needed most — the "who watches the watchmen" problem.
Confirm: Check if the Prometheus pod itself restarted/OOMKilled around that time; its own self-scraped prometheus_tsdb_head_series metric for a spike.
Fix: Raise Prometheus's resource limits with real headroom, reduce label cardinality sources, and seriously consider isolating your monitoring stack's blast radius from what it watches.
Lesson: Your observability stack is a workload subject to every failure mode in Modules 1–7 — including failing exactly when incident-driven churn stresses it hardest.

13 — Job's pod exits 0, Job never shows Complete
Cause: .spec.completions is set >1 and not all required successful completions have happened yet.
Confirm: kubectl get job -o yaml — compare completions vs status.succeeded.
Lesson: One successful pod ≠ Job done if it needs N successes — check the completions count first.

14 — App can't reach its own sidecar on localhost
Cause: Regular containers in a pod don't start in guaranteed order — the app started before the sidecar was actually listening.
Confirm: Check the sidecar's own readiness/logs at that moment.
Lesson: "localhost" doesn't mean "already listening" — use native sidecars (restartPolicy: Always init containers) or add retry logic.

15 — Deployment scaled to 10, only 7 come up, no errors "anywhere"
Cause: Almost always a ResourceQuota nobody remembered exists — or the missing 3 are sitting Pending with FailedScheduling events nobody actually checked.
Confirm: kubectl get resourcequota -n <ns>; check the missing pods' own status directly.
Lesson: "No errors anywhere" often just means nobody checked the individual pod events yet.

16 — After AMI upgrade, PVC pods hit Multi-Attach without scaling
Cause: Nodes were replaced without a proper drain, leaving a stale VolumeAttachment pointing at a now-gone node — Module 5 Lab 4's scenario via routine ops.
Confirm: kubectl get volumeattachment for one referencing a node that no longer exists.
Lesson: Node replacement must cordon+drain before termination, or you inherit the stuck-VolumeAttachment problem as routine fallout.

17 — Cron job fails only on the last day of the month
Cause: A date-math bug in the job's own code (month-boundary/day-31 edge case) — genuinely an application bug, not infrastructure.
Confirm: Check the job's logs from a failing run for a date-parsing error.
Lesson: Not everything is a Kubernetes problem — calendar-aligned failures are a classic app-code signature; don't reach for infra diagnostics first.

18 — kubectl get pods slow, only in one namespace
Cause: That namespace has accumulated a huge number of uncleaned objects (completed Jobs/Pods with no TTL), making list operations against it genuinely slower.
Confirm: kubectl get pods -n <ns> --no-headers | wc -l (and Jobs/Events) vs a normal namespace.
Lesson: Object hygiene matters — ttlSecondsAfterFinished on Jobs exists specifically to prevent this.

That's the full set — ten years of "why is this happening" compressed into fourteen. Whenever you want to actually reproduce any of these instead of just reading them, say the number and I'll build the lab the same way we did for 1–3.




Claude is AI and c

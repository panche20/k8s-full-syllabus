# Kubernetes Troubleshooting MasterClass

## The Roadmap

<img width="912" height="538" alt="image" src="https://github.com/user-attachments/assets/8b86eb22-1332-4194-96b8-90bfedd61cda" />

<img width="916" height="550" alt="image" src="https://github.com/user-attachments/assets/cf2eacc0-d585-465e-84d6-41075f595782" />

*Modules 6–7 will lean on your chaos labs directly. Let's start.*

**Module 1 — The Troubleshooting Mental Model & Anatomy of a K8s Error**

**The 'Why' (The Problem)**

Debugging a monolith on 3 servers means SSH in, tail -f a log, done — one layer of indirection. Kubernetes has roughly eight: kubectl → API server → etcd → controllers → scheduler → kubelet → CRI → container runtime → cgroups/namespaces/kernel. A single symptom ("pod won't start") can originate at any of these layers, and each layer has its own vocabulary of failure.

The reason strong engineers stall here isn't lack of knowledge — it's lack of an algorithm. Without one, every error becomes a fresh Google search instead of a five-second triage. Senior SREs aren't reading errors faster because they've memorized more strings; they've internalized which component is allowed to say what, so a string like FailedScheduling instantly tells them "scheduler, not kubelet — don't waste time checking node disk pressure."

**Deep-Dive Mechanics**

**1. Everything is a reconciliation loop.** Every K8s error is fundamentally: desired state (spec, in etcd) ≠ observed state (status, reported by some component). Troubleshooting is just identifying which controller owns the mismatch.

**2. The "From" field is the most important field you're ignoring.** Run kubectl describe pod <name> and look at the Events table:

```
Type     Reason          Age   From               Message
----     ------          ---   ----               -------
Warning  FailedScheduling 30s  default-scheduler  0/3 nodes available...
Normal   Pulling          25s  kubelet            Pulling image "nginx:bad-tag"
Warning  Failed           20s  kubelet            Error: ErrImagePull
```

From tells you the emitting component. That alone routes you: default-scheduler → scheduling problem (resources/taints/affinity), kubelet → node-local problem (image, volume, CRI). Most people read Message first and skip From — reverse that habit.

**3. Three separate status systems get conflated constantly:**

- **Pod Phase:** Pending / Running / Succeeded / Failed / Unknown — coarse, almost useless alone.
- **Container State:** Waiting / Running / Terminated, each with a Reason (CrashLoopBackOff, ImagePullBackOff, ContainerCreating, OOMKilled) — this is where the real signal lives.
- **Pod Conditions:** PodScheduled → Initialized → ContainersReady → Ready, each True/False/Unknown. A Ready: False with everything else True almost always means a failing readiness probe, not a crash.

**4. Exit codes are a language of their own** — check these before you check logs:

<img width="923" height="435" alt="image" src="https://github.com/user-attachments/assets/77f7e2b5-b42d-4701-b2ac-405abadb2683" />

**5. Log discipline:** kubectl logs <pod> shows the current container's logs. If it already crashed and restarted, that's the new attempt — you want kubectl logs <pod> --previous for the one that actually died. Init containers and sidecars need -c <container-name> explicitly. And Events are namespace-scoped and TTL'd (~1h by default) — if you're investigating something from an hour ago, the Events are already gone unless you're shipping them somewhere.

**Quick-Reference: The Diagnostic Tree**

```
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
```

**The Alternative Landscape**

<img width="916" height="550" alt="image" src="https://github.com/user-attachments/assets/5d4aee67-9034-4107-bb16-90fd2f97035b" />

You'd choose raw kubectl mastery first regardless of what you adopt later — it's the substrate every other tool is built on, and it's the only one that works when you're whiteboarding in an interview with no cluster access.

**Interview POV & Edge Cases**

Interviewers rarely test "do you know what CrashLoopBackOff means" — they test whether you have a repeatable process when they hand you an unfamiliar symptom. Expect: "A pod is stuck in Pending, walk me through your triage." They're grading the order of your commands, not just the destination.

*Common gotchas senior engineers get tripped on:*

- Confusing Error (a generic reason from the container runtime, not Kubernetes) with an actual root cause — it's a prompt to go read logs --previous, not an answer.
- Forgetting Events expire — investigating a stale incident with kubectl describe alone and finding nothing.
- Treating Ready: False as a crash when the container is actually running fine but failing its readiness probe (classic self-inflicted outage during a deploy).
- Not checking kubectl get events --sort-by=.lastTimestamp -A cluster-wide when the failing pod's own Events are empty — the real cause (e.g., a NetworkPolicy or ResourceQuota) can live outside the pod's own nameserver of events.

**The 'Better Way' (Evolution)**

Two real shifts happening in production troubleshooting: eBPF-based observability (Cilium Hubble, Pixie) gives you network/syscall-level visibility with zero app instrumentation, closing the "it's a CNI problem" blind spot raw kubectl can't see into. And AI-assisted RCA (k8sgpt, Robusta) ingests Events+logs+metrics and proposes a root-cause hypothesis — genuinely useful as a first pass, but treat it as a hypothesis generator, not ground truth, since it can confidently misattribute causes in novel failure modes.

**The Lab Format: Break → Predict → Diagnose → Fix → Debrief**

*Every lab follows five steps, in this order, every time:*

- Break — exact command/manifest to inject a real failure
- Predict — before running any diagnostic command, write down which component (From field) you expect to complain, and why. This is the step almost everyone skips, and it's the one that actually builds the algorithm.
- Diagnose — walk the diagnostic tree from Module 1 with real kubectl output
- Fix — remediate it
- Debrief — name the Reason/exit code, and note the adjacent error it's easy to confuse it with (this is the interview-differentiation muscle)

**Module 1 Lab Pack (7 labs, mapped to the diagnostic tree)**

<img width="890" height="392" alt="image" src="https://github.com/user-attachments/assets/235cf2c5-6263-4c70-b6cd-bd177f9c8820" />

Let's do Lab 1 now as the template — run this on your master:

**1. Break**

```
kubectl create deployment greedy-pod --image=nginx --replicas=1
kubectl set resources deployment greedy-pod --requests=cpu=100,memory=100Gi
```

(100 CPU cores / 100Gi memory — no single EC2 worker in a 2-node kubeadm lab has that.)

**2. Predict (do this before step 3)**
Which component do you expect to log the complaint — kubelet, scheduler, or controller-manager? Write your answer down.

**3. Diagnose**

```
kubectl get pods
kubectl describe pod <greedy-pod-xxxx>
```

Look specifically at the From field and the Events Reason. You should see default-scheduler and FailedScheduling, with a message like 0/2 nodes are available: 2 Insufficient cpu, 2 Insufficient memory.

**4. Fix**

```
kubectl set resources deployment greedy-pod --requests=cpu=100m,memory=128Mi
```

**5. Debrief**

FailedScheduling is always the scheduler, always pre-node-assignment — the pod never even reaches a kubelet, which is why kubectl logs on it will just error with "pod has no containers." That's your tell: if logs fails outright and describe shows no Node: assigned, you're still in the scheduler's territory, not the kubelet's.

**Lab 2 — Taint/Toleration Mismatch**

**Break**

```
bash
kubectl get nodes
kubectl taint nodes <worker-1> dedicated=gpu:NoSchedule
kubectl taint nodes <worker-2> dedicated=gpu:NoSchedule
kubectl create deployment tainted-test --image=nginx --replicas=1
```

**Predict**: Same component as Lab 1 or different? (It's the same — scheduler — but the message will look different. Predict what the message says before you look.)

**Diagnose**

```
bash
kubectl describe pod <tainted-test-xxxx>
```

**Expect**: 0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) had untolerated taint {dedicated: gpu}.

**Fix**

```
kubectl patch deployment tainted-test --type=json -p='[{"op":"add","path":"/spec/template/spec/tolerations","value":[{"key":"dedicated","operator":"Equal","value":"gpu","effect":"NoSchedule"}]}]'
```

**Debrief**: Both Lab 1 and Lab 2 emit FailedScheduling from default-scheduler — same Reason, different Message. The Reason tells you which layer; the Message tells you which sub-cause. Interviewers will push exactly here: "you said FailedScheduling, but which kind?" Also note: the control-plane taint appearing in the message is normal, not a bug — don't chase it.

**Lab 3 — ImagePullBackOff**

**Break**

```
kubectl create deployment bad-image --image=nginx:this-tag-does-not-exist-v99
```

**Predict**: Scheduler or kubelet this time? (Different from Labs 1–2 — the pod does get assigned a node.)

**Diagnose**

```
kubectl get pods
kubectl describe pod <bad-image-xxxx>
kubectl logs <bad-image-xxxx>
```

Expect From: kubelet, cycling through Reason: Failed ("manifest unknown") → Reason: BackOff → container state Waiting: ImagePullBackOff. kubectl logs will error — there's no container to log, since one never started.

```
kubectl set image deployment/bad-image nginx=nginx:1.25
```

**Debrief**: ErrImagePull is the immediate failed attempt; ImagePullBackOff is the state once kubelet starts exponential backoff (up to 5 min between retries) — don't panic and delete the pod, it'll just retry immediately and hit the same wall. If this were a private registry instead of a bad tag, the message says unauthorized/pull access denied instead of manifest unknown — that's your signal it's imagePullSecrets, not the tag.

**Lab 4 — CrashLoopBackOff via OOM (exit 137)**

**Break**

```
kubectl run oom-test --image=polinux/stress --restart=Never \
  --limits=memory=50Mi --requests=memory=50Mi \
  -- stress --vm 1 --vm-bytes 150M --vm-hang 0
```

**Predict**: Which layer kills the process — Kubernetes, or the Linux kernel?

**Diagnose**

```
kubectl get pod oom-test
kubectl describe pod oom-test
```

Look in the Containers → Last State block, not the Events table: Terminated, Reason: OOMKilled, Exit Code: 137. This is the important nuance — OOM kills often don't show up as a nice Warning Event the way scheduling failures do. The kernel's cgroup OOM killer fires, kubelet just reports the exit code after the fact.

**Fix**

```
kubectl delete pod oom-test
kubectl run oom-test --image=polinux/stress --restart=Never \
  --limits=memory=300Mi --requests=memory=300Mi \
  -- stress --vm 1 --vm-bytes 150M --vm-hang 0
```

**Debrief**: Same CrashLoopBackOff container state as Lab 5 below, completely different root cause and layer (kernel, not app code). If you only look at the top-level Waiting: CrashLoopBackOff, Labs 4 and 5 look identical — you have to go one level deeper into Last State to tell them apart. That's the whole point of this 
lab.

**Lab 5 — CrashLoopBackOff via App Error (exit 1)**

**Break**

```
kubectl run app-crash --image=busybox --restart=Never -- /bin/sh -c "exit 1"
```

**Predict**: Will kubectl logs --previous show anything useful here, unlike Lab 3?

**Diagnose**

```
kubectl get pod app-crash
kubectl describe pod app-crash
kubectl logs app-crash --previous
```

Expect Last State: Terminated, Reason: Error, Exit Code: 1. Unlike OOMKilled, Reason: Error is a catch-all — it just means "the process ran and chose to exit non-zero." The real answer lives in logs --previous, not in K8s metadata.

**Fix**

```
kubectl delete pod app-crash
kubectl run app-crash --image=busybox --restart=Never -- sleep 3600
```

**Debrief**: Worth knowing but not reproducing here — if the entrypoint binary itself doesn't exist (vs. exiting from inside a running process), you won't get exit 127 at all. You'll get Reason: CreateContainerError or StartError, because the container runtime failed before a process ever got a PID. Same symptom family, one more layer back — logs will error out entirely, same as Lab 3.

**Lab 6 — Ready: False (Readiness Probe Misconfig)**

**Break**

```
kubectl create deployment probe-fail --image=nginx --replicas=1
kubectl patch deployment probe-fail --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"httpGet":{"path":"/","port":8080},"initialDelaySeconds":2,"periodSeconds":5}}]'
kubectl expose deployment probe-fail --port=80
```

**Predict**: Will the container restart? Check RESTARTS in kubectl get pods before you assume.

**Diagnose**

```
kubectl get pods -o wide
kubectl describe pod <probe-fail-xxxx>
kubectl get endpoints probe-fail
```

Expect: Phase Running, READY 0/1, RESTARTS 0. Events: Warning Unhealthy — Readiness probe failed: dial tcp ...:8080: connection refused. kubectl get endpoints shows <none> — the Service has zero backends because the pod never became Ready.

**Fix**

```
kubectl patch deployment probe-fail --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/port","value":80}]'
kubectl get endpoints probe-fail
```

**Debrief**: RESTARTS: 0 is your tell that this is readiness, not liveness or a crash — the container is genuinely healthy, it's just failing a check. Readiness failures pull a pod out of Service traffic without killing it; liveness failures restart a perfectly fine container. Confusing the two is a classic self-inflicted outage during a rollout.

**Lab 7 — Evicted (DiskPressure)**

This one's node-level, so it starts with SSH, not kubectl.

**Break (SSH into a worker node)**

```
ssh -i <your-key>.pem ubuntu@<worker-node-ip>
df -h /
# fill to under ~10% available — adjust size to your actual free space
sudo fallocate -l <size>G /tmp/bigfile
df -h /
```

**Predict**: Where do you check first — the pod, or the node?

**Diagnose** (back on master)

```
kubectl describe node <worker-node-name>
kubectl get pods -A -o wide --field-selector spec.nodeName=<worker-node-name>
```

Expect Conditions: DiskPressure True on the node, and pods on it moving to Status: Evicted with message The node was low on resource: ephemeral-storage.

**Fix**

```
# on the worker
sudo rm /tmp/bigfile
df -h /
# on master, confirm
kubectl describe node <worker-node-name>
```

If your evicted pod was a bare kubectl run pod (no controller), it stays Evicted forever — delete it manually: kubectl delete pod <name>.

**Debrief**: This is the only lab in the pack where root cause starts at kubectl describe node, not kubectl describe pod — every other failure here was a pod-spec-vs-cluster-state mismatch; this one is a genuine node-health problem. Also: a Deployment-managed pod would've self-healed onto the other worker automatically once evicted — a bare pod won't. That distinction is a favorite interview gotcha ("why didn't it come back on its own").

**Module 2 — Pod & Container Lifecycle Failures (The CRI / Runtime Layer)**

Module 1 stayed at the symptom layer — Reason strings and exit codes as reported by kubelet. Module 2 goes one layer deeper: what actually happens between kubelet deciding to run a container and a process existing on the node, because a lot of "impossible" errors only make sense once you see that pipeline.

**The 'Why' (The Problem)**

Early Kubernetes talked to Docker directly, hardcoded. That coupling was a maintenance nightmare — every new runtime (rkt, containerd standalone) needed bespoke kubelet code. CRI (Container Runtime Interface) was created to decouple kubelet from any specific runtime via a stable gRPC contract. That's great for pluggability, but it means failures can now originate at a layer kubectl has zero visibility into — the CRI shim, the OCI runtime, or the kernel itself. When kubectl describe goes silent or vague ("Error", "ContainerCannotRun"), it's because you've hit a layer kubectl was never designed to explain. Senior engineers know when to stop asking kubectl and start asking the node directly.

**Deep-Dive Mechanics**

**The full creation pipeline:**

```
kubelet --(CRI gRPC, /run/containerd/containerd.sock)--> containerd
   --> containerd-shim-runc-v2 (one per container)
      --> runc (builds namespaces + cgroups + rootfs)
         --> Linux kernel
```

kubelet never touches runc or the kernel directly — it only speaks CRI to containerd. This is why crictl (the CRI-native CLI) sometimes shows you truth that kubectl can't: it talks to containerd on the same socket kubelet uses, bypassing the API server entirely.

**The sandbox concept — this is the part most people misunderstand.** For every Pod, kubelet's first CRI call is RunPodSandbox, which creates a pause container — a tiny infra container whose entire job is to own the Pod's network (and optionally IPC/UTS) namespace. Every app container in that Pod is then started with CreateContainer/StartContainer and joined to the sandbox's namespace, not to each other. That's the real mechanism behind "all containers in a Pod share localhost and one IP" — they're all tenants of the pause container's network namespace.

**Init containers run strictly sequentially, gated on exit 0.** kubelet will not call RunPodSandbox's follow-up container creation for any app container until every init container has exited zero, in order. If init container #2 of 3 fails, app containers are never even created — not stuck, not erroring, just never invoked. Pod status shows Init:Error or Init:CrashLoopBackOff, and Status: PodInitializing for everything downstream.

**PLEG (Pod Lifecycle Event Generator)** is kubelet's internal loop that polls the runtime for container state changes. If the runtime is slow to respond (disk I/O pressure, containerd overloaded, huge number of containers on the node), PLEG relists start timing out, kubelet flags PLEG is not healthy, and — critically — the node can go NotReady even though the kubelet process itself is alive and well. This trips people up constantly: NotReady is instinctively read as "kubelet crashed," and it's frequently not.

**Lifecycle hooks — async vs sync, and why it matters:**

- postStart runs asynchronously right after the container starts — there's no guarantee it completes (or even starts) before your ENTRYPOINT's first instruction runs. If it fails or times out, kubelet kills the container — FailedPostStartHook — even if your main process is perfectly healthy. This looks exactly like an app crash and isn't one.
- preStop runs synchronously before SIGTERM, and blocks pod termination for up to terminationGracePeriodSeconds (default 30s). A hanging preStop is the #1 cause of pods stuck in Terminating — not a broken kubectl delete.

**The Alternative Landscape**

<img width="890" height="572" alt="image" src="https://github.com/user-attachments/assets/e9581260-3e78-4a4f-8467-87c73a36ea6d" />

Why containerd over the isolation-heavy options for most of your target roles: platform teams at Booking/Adyen/Revolut run their own workloads, not arbitrary customer code — namespace isolation is enough, and the performance cost of gVisor/Kata is a bad trade with no matching threat model. You'd reach for runtimeClassName: gvisor specifically when a single cluster runs code from parties you don't fully trust.

**Interview POV & Edge Cases**

A favorite senior-level question: "A node shows NotReady — walk me through it." Weak answers jump straight to "restart kubelet." Strong answers check journalctl -u kubelet first for PLEG is not healthy before touching anything, because restarting kubelet on a node with a genuinely overloaded runtime just repeats the same failure.

**Gotchas**:

- ContainerCreating stuck for 30s+ is almost never an app problem — no container exists yet, so kubectl logs can't help you. Check kubelet logs and CNI plugin state instead, not the app.
- Init container logs need -c <init-container-name> explicitly — running plain kubectl logs <pod> on a multi-init-container pod either errors or grabs the wrong container.
- FailedPostStartHook reads like a crash but the main process may be completely fine — check the hook definition before you touch the app.
- Stuck Terminating → check for a hanging preStop or a stuck finalizer before repeatedly force-deleting, which can orphan processes on the node.

**The 'Better Way' (Evolution)**

**Ephemeral debug containers (kubectl debug)** replaced the old practice of baking a shell into production images "just in case" — you can now attach a debug container with a full toolset into a running Pod's namespaces on demand, even against distroless images with no shell at all. And Evented PLEG (relist-on-change instead of poll-on-interval) reduces the false-NotReady flapping that plagued large, container-dense nodes under the old polling model.

**Module 2 Lab Pack**

<img width="932" height="360" alt="image" src="https://github.com/user-attachments/assets/5de5ebe0-1fa2-4fd9-98dc-226b208b5b50" />

**Lab 1 — Sandbox Anatomy via crictl**

**Break** (nothing broken — this is reconnaissance)

```
kubectl create deployment sandbox-demo --image=nginx --replicas=1
kubectl get pod -o wide   # note which worker it landed on
```

**Predict**: How many entries will crictl ps show for one single-container Pod — one, or two?

**Diagnose** (SSH into that worker node)

```
sudo crictl pods                          # shows the sandbox (pause container)
sudo crictl ps -a --pod <sandbox-id>       # shows the actual nginx container inside it
sudo crictl inspect <container-id> | grep -A3 '"namespaces"'
```

You'll see two CRI-level objects for one Pod: the sandbox and the app container. crictl inspectp <sandbox-id> shows the pause container owns the network namespace; crictl inspect on nginx shows it referencing the same namespace path.

**Fix**: N/A — nothing to fix, just clean up:

```
kubectl delete deployment sandbox-demo
```

**Debrief**: This is the exact structure every later lab in this module builds on. When something goes wrong "inside the pod," you now know there are actually two separate CRI lifecycles to check, not one.

**Lab 2 — Init Container Failure Blocks Everything**

**Break**

```
kubectl run init-fail --image=nginx --restart=Never --dry-run=client -o yaml \
  --overrides='{"spec":{"initContainers":[{"name":"init-check","image":"busybox","command":["sh","-c","exit 1"]}]}}' \
  > init-fail.yaml
kubectl apply -f init-fail.yaml
```

**Predict**: Will the nginx container ever get created at all?

**Diagnose**

```
kubectl get pod init-fail
kubectl describe pod init-fail
kubectl logs init-fail -c init-check
kubectl logs init-fail   # try this too — see what happens
```

Expect Status: Init:Error then Init:CrashLoopBackOff. kubectl logs init-fail (no -c) will error — the main container was never created, so there's nothing to default to.

**Fix**

```
kubectl delete pod init-fail
# edit init-fail.yaml: change command to ["sh","-c","exit 0"], reapply
kubectl apply -f init-fail.yaml
```

**Debrief**: No sandbox-level problem here at all — RunPodSandbox succeeded fine. The block is entirely at the init-container-sequencing layer, one level above CRI. This is the cleanest example of "the pod is Pending-looking but it's not a scheduling issue" — resist the urge to check node resources here.

**Lab 3 — Wrong-Architecture Image (OCI Runtime Layer)**

**Break** (your EC2 nodes are x86_64/amd64 — pull an arm64-only image)

```
kubectl run arch-fail --image=arm64v8/busybox --restart=Never -- sleep 3600
```

**Predict**: Will this fail at image pull, or after?

**Diagnose**

```
kubectl get pod arch-fail
kubectl describe pod arch-fail
```

Expect the image to pull successfully (Docker Hub serves it fine), then: Waiting: CrashLoopBackOff, Last State: Terminated, and buried in the message something like exec: "sleep": executable file not found or, more directly on some runtimes, exec format error. This is your OCI-runtime-layer signal — runc successfully built the container, but the kernel refused to execute a binary compiled for a different CPU architecture.

**Fix**

```
kubectl delete pod arch-fail
kubectl run arch-fail --image=busybox --restart=Never -- sleep 3600
```

**Debrief**: This is a real production failure mode, not a contrived one — it's the classic "built on an Apple Silicon laptop, deployed to x86 nodes" incident. It looks identical to Lab 5 of Module 1 (CrashLoopBackOff, generic Error) at the top level, but the message text is your only differentiator, which is exactly why Module 1 taught you to always read past the Reason into the Message.

**Lab 4 — FailedPostStartHook (Not a Real Crash)**

**Break**

```
kubectl run poststart-fail --image=nginx --restart=Never \
  --dry-run=client -o yaml \
  --overrides='{"spec":{"containers":[{"name":"poststart-fail","image":"nginx","lifecycle":{"postStart":{"exec":{"command":["sh","-c","exit 1"]}}}}]}}' \
  > poststart-fail.yaml
kubectl apply -f poststart-fail.yaml
```

**Predict**: Is nginx itself broken, or something else? Will kubectl logs show an nginx error?

**Diagnose**

```
kubectl describe pod poststart-fail
kubectl logs poststart-fail
```

Events show Warning FailedPostStartHook ... command 'sh -c exit 1' exited with 1. kubectl logs shows a perfectly normal nginx startup — because nginx did start fine. Kubelet killed it anyway, purely because the hook failed.

**Fix**

```
kubectl delete pod poststart-fail
# edit yaml: change hook command to ["sh","-c","exit 0"], reapply
```

**Debrief**: This is the gotcha from the Deep-Dive made concrete — clean, healthy app logs sitting right next to a restart loop. If you only read kubectl logs here you'd conclude "nginx is fine, this must be a fluke" and waste time. The Events table, not the app logs, holds the actual root cause.

**Lab 5 — Stuck Terminating (Hanging preStop)**

**Break**

```
kubectl run stuck-terminate --image=nginx --restart=Never \
  --dry-run=client -o yaml \
  --overrides='{"spec":{"terminationGracePeriodSeconds":20,"containers":[{"name":"stuck-terminate","image":"nginx","lifecycle":{"preStop":{"exec":{"command":["sh","-c","sleep 300"]}}}}]}}' \
  > stuck-terminate.yaml
kubectl apply -f stuck-terminate.yaml
kubectl delete pod stuck-terminate &
```

**Predict**: Will the pod disappear in ~20s (the grace period) or hang far longer? Why 20s and not 300s?

**Diagnose**

```
bash
kubectl get pod stuck-terminate -w   # watch it sit in Terminating
```

It should actually resolve around the 20s mark (the grace period timeout), not the 300s the hook is sleeping for — kubelet sends SIGKILL once terminationGracePeriodSeconds elapses, regardless of whether preStop finished.

**Fix**: nothing to fix — this one resolves itself. If it didn't (grace period set very high), you'd force it:

```
bash
kubectl delete pod stuck-terminate --grace-period=0 --force
```

**Debrief**: This lab is deliberately designed to show you the grace period is a hard ceiling, not a suggestion — a runaway preStop can't hold a pod hostage forever, only up to that ceiling. The real-world version of this bug is usually a preStop doing a slow drain (deregistering from a load balancer, flushing connections) that occasionally exceeds the configured grace period, causing ungraceful kills under load — worth remembering for capacity/rollout design questions.

**Lab 6 — Debugging a Shell-less (Distroless) Container**

**Break**

```
bash
kubectl run distroless-demo --image=gcr.io/distroless/static-debian12 --restart=Never -- sleep 3600
```

**Predict**: Can you kubectl exec into this and get a shell?

**Diagnose**

```
bash
kubectl exec -it distroless-demo -- sh
```

This fails — exec: "sh": executable file not found in $PATH — distroless images ship no shell by design (smaller attack surface).

**Fix / the modern way**

```
kubectl debug -it distroless-demo --image=busybox --target=distroless-demo
```

This attaches a separate ephemeral container into the same Pod, sharing its process namespace, giving you a real shell and the ability to ps, inspect /proc/1/root, etc. — without ever modifying the production image or restarting the workload.

**Debrief**: This is the "Better Way" from the Deep-Dive in practice. The old habit of building a "-debug" variant image with a shell baked in is obsolete — ephemeral debug containers give you the same power on demand, against the exact running Pod, with zero image changes. Expect this to come up directly if an interviewer asks "how do you debug a container with no shell."

**Module 3 — Scheduling & Pending Pod Failures (Scheduler Internals)**

Module 1 gave you the surface-level diagnostic tree for FailedScheduling. This module goes into why the scheduler decides what it decides — because once you've seen enough of these, "insufficient resources" and "didn't match affinity" and "topology spread violated" all start looking the same at a glance, and interviewers specifically probe whether you can tell them apart.

**The 'Why' (The Problem)**

Naive scheduling is just bin-packing: find any node with enough free CPU/memory, place the pod. That breaks down fast in production. If you only bin-pack, you get: all three replicas of a critical service landing on the same node (one node failure = full outage, even though you "have 3 replicas"), a batch job with unbounded resource appetite starving a payment service of capacity, cache pods scheduled far from the services that need low-latency access to them, and stateful workloads scheduled onto nodes that can't actually reach their storage. Kubernetes needed a scheduler that could reason about resource fit and topology and priority and existing cluster state simultaneously — which is why kube-scheduler evolved into a two-phase, plugin-based decision engine rather than a single fit-check.

**Deep-Dive Mechanics**

**Two phases, always in this order: Filter, then Score.**

```
For each pod in the scheduling queue:
  1. Filtering  — eliminate every node that CANNOT run this pod (hard constraints)
  2. Scoring    — rank the surviving nodes 0-100 (soft preferences)
  3. Bind       — pick the highest score (ties broken randomly), write pod.spec.nodeName
```

If zero nodes survive Filtering → Pending, FailedScheduling, and (if PriorityClasses are in play) preemption is attempted. If all nodes survive Filtering but with different scores, the pod goes to the winner — this step never produces an error, so you'll never see a "ScoringFailed" event; scoring only affects which node, never whether.

**What actually runs in the Filter phase (this is the part worth memorizing, because each one produces a distinct message):**

- NodeResourcesFit — enough allocatable CPU/memory/ephemeral-storage? → Insufficient cpu/memory
- NodeUnschedulable / taints-tolerations → untolerated taint {...}
- Node affinity (requiredDuringScheduling...) → didn't match Pod's node affinity/selector
- Pod affinity/anti-affinity (requiredDuringScheduling...) → didn't match pod affinity rules / didn't satisfy existing pods anti-affinity rules
- VolumeBinding (PVC node affinity) → node(s) had volume node affinity conflict
- PodTopologySpread (hard, DoNotSchedule) → didn't match pod topology spread constraints

The scheduler evaluates all of these per node and aggregates failures across nodes into one message — this is why you sometimes see a FailedScheduling event listing two or three different reasons across different node counts, e.g. 1 Insufficient cpu, 2 node(s) didn't match pod topology spread constraints. Read every clause, not just the first one.

**Scoring** — plugins like NodeResourcesBalancedAllocation (prefers nodes where CPU:memory usage ratio stays balanced), ImageLocality (prefers nodes that already have the image cached — pull time avoidance), InterPodAffinity (soft affinity/anti-affinity preferences), PodTopologySpread (soft spread preferences) each emit 0–100, get multiplied by a configurable weight, and summed. This is invisible in describe output — there's no Event for "why this node won over that one." If you need to debug scoring specifically, you're into --v=10 scheduler logs, not kubectl.

**Preemption (PostFilter phase).** If Filtering fails for every node, the scheduler doesn't immediately give up — it checks whether the pod has a PriorityClass and, if so, simulates evicting lower-priority pods on candidate nodes to see if that would let this pod fit. If yes: it picks victims, sets status.nominatedNodeName on the pending pod, and gracefully deletes the victims (respecting their terminationGracePeriodSeconds). Critically — nomination is not a guarantee. The scheduler doesn't hold that node reserved; if another pod (even a lower-priority one that fits without eviction) gets bound to that node first in a later scheduling cycle, your nominated pod goes back to the queue and looks for somewhere else. This trips people up in incident review: "the events show it was nominated to node X, why is it running on node Y?"

**The retry queue architecture.** Pods that fail scheduling don't sit in a hot retry loop — they move to an internal unschedulablePods pool and get retried either on a backoff timer or when a relevant cluster event fires (node added, pod deleted, PVC bound). This is why deleting an unrelated pod can sometimes immediately unstick a Pending one elsewhere — it triggered a requeue, not because you fixed anything directly.

**Volume binding mode matters more than people think.** Immediate binding provisions the PV as soon as the PVC is created — with zero awareness of which node the pod will land on. If that volume is AZ-pinned (EBS is AZ-local) and the pod later gets scheduled to a node in a different AZ, you get a silent, unresolvable volume node affinity conflict — no amount of retrying fixes it, the PVC has to be deleted and recreated. WaitForFirstConsumer binding delays provisioning until a pod actually needs it, so the CSI driver can provision in the same AZ as wherever the scheduler decided to place the pod. Default StorageClass binding mode is a config decision you should always check on someone else's cluster before you trust PVC behavior.

**The Alternative Landscape**

<img width="897" height="523" alt="image" src="https://github.com/user-attachments/assets/28ba9bf1-9cbd-4ce9-bf85-f93b93f4719b" />

<img width="846" height="327" alt="image" src="https://github.com/user-attachments/assets/ccc4d302-4787-485b-8589-43254627d0df" />

Why default kube-scheduler for most of your target roles: platform/SRE work at Booking, Adyen, Revolut is dominated by long-running services, not batch ML training jobs — you want the well-understood, well-instrumented default. You'd reach for Volcano/YuniKorn specifically the moment you're running distributed training where launching half a job's workers (and having them hang waiting for the other half that never got scheduled) wastes GPU-hours — that's a real, common failure mode default kube-scheduler has no concept of, since it schedules pods one at a time with no notion of "this whole group succeeds or none of it starts."

**Interview POV & Edge Cases**

Classic prompt: "Design a scheduling strategy so a 3-replica service survives a single node failure." The answer they want is podAntiAffinity (or topologySpreadConstraints, which is now generally preferred) — and the follow-up they'll push on is: "what if there are only 2 nodes and you ask for requiredDuringScheduling anti-affinity with 3 replicas?" Correct answer: the 3rd replica goes permanently Pending, because required is a hard filter, not a preference — this is the single most common self-inflicted Pending in real clusters, and interviewers use it to check whether you actually understand required vs preferred.

*Gotchas senior engineers are expected to catch:*

- requiredDuringSchedulingIgnoredDuringExecution doesn't evict anything once running — the "IgnoredDuringExecution" suffix means it's enforced only at scheduling time. A node's labels can change after the fact without evicting a pod that would now violate the rule.
- nominatedNodeName in describe output is a hint, not a placement — never assume a nominated pod is guaranteed to land there.
- Static pods (kubelet reads /etc/kubernetes/manifests directly) completely bypass the scheduler — they'll never show FailedScheduling, never respect taints the normal way, and can't be prevented from running on a node via node affinity. This matters directly on your kubeadm cluster: your control-plane static pods (kube-apiserver, etcd, etc.) are exactly this category.
- PreferNoSchedule taints are pure scoring hints (soft) — a pod can and will land there if nothing else fits, unlike NoSchedule which is a hard filter.
- NoExecute taints don't just block new pods — they evict already-running pods that lack a matching toleration, on a timer if tolerationSeconds is set, immediately if not.

**The 'Better Way' (Evolution)**

For batch/ML workloads, Kueue (now a CNCF project many platform teams standardize on) sits above kube-scheduler and adds job-level admission control and fair-share quota — instead of pods trickling in and partially scheduling, whole jobs wait in a queue until their full resource requirement is available. For gang-scheduling specifically (all pods of a job start together or none do), Volcano or YuniKorn replace the scheduling decision itself. And for capacity provisioning reacting to Pending pods, Karpenter (which you've already used in your EKS/Terraform work) watches for unschedulable pods and provisions exactly-fitting nodes just-in-time, rather than the older cluster-autoscaler pattern of scaling a fixed set of pre-defined node group shapes.

**Module 3 Lab Pack**

<img width="937" height="342" alt="image" src="https://github.com/user-attachments/assets/4830a969-0833-47ec-afc1-5355a7546824" />

*First, get your real numbers — don't skip this:*

```
bash
kubectl describe nodes | grep -A5 "Allocatable"
kubectl get nodes -o wide
```

Note your two worker names and rough allocatable CPU/memory — Lab 4 needs you to size requests off your actual capacity, not mine.

**Lab 1 — Impossible Node Affinity**

**Break**

```
kubectl run affinity-fail --image=nginx --restart=Never \
  --dry-run=client -o yaml \
  --overrides='{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"disktype","operator":"In","values":["ssd-nvme-special"]}]}]}}}}}' \
  > affinity-fail.yaml
kubectl apply -f affinity-fail.yaml
````

**Predict**: Same message text as the taint lab back in Module 1, or different wording for the same FailedScheduling Reason?

**Diagnose**

```
kubectl describe pod affinity-fail
```

Expect: 0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.

**Fix**

```
kubectl label node <worker-1> disktype=ssd-nvme-special
kubectl get pod affinity-fail -w
```

**Debrief**: Same Reason: FailedScheduling, completely different message text from an untolerated-taint failure — "didn't match node affinity/selector" is a label mismatch (pod is picky about the node), "untolerated taint" is the node rejecting the pod (node is picky about the pod). They're two different Filter plugins failing; only the Message tells you which. Also worth noting: if you'd used preferredDuringScheduling instead of required, this pod would have scheduled immediately onto any node — preferred constraints only affect the Score phase and can never cause a Pending.

**Lab 2 — Hard Topology Spread Violation (minDomains)**

**Break**

```
kubectl create deployment spread-test --image=nginx --replicas=2
kubectl patch deployment spread-test --type=json -p='[{"op":"add","path":"/spec/template/spec/topologySpreadConstraints","value":[{"maxSkew":1,"minDomains":3,"topologyKey":"kubernetes.io/hostname","whenUnsatisfiable":"DoNotSchedule","labelSelector":{"matchLabels":{"app":"spread-test"}}}]}]'
kubectl scale deployment spread-test --replicas=3
```

(minDomains: 3 requires the constraint to see at least 3 distinct hostname domains before it's satisfied — you only have 2 worker nodes, so this is guaranteed to fail regardless of replica count or resource sizing.)

**Predict**: Will this fail with an "insufficient resources" message or something else entirely?

**Diagnose**

```
kubectl get pods -o wide
kubectl describe pod <spread-test-xxxx-pending-one>
```

**Expect**: 0/3 nodes are available: 3 node(s) didn't match pod topology spread constraints. — nothing about CPU or memory at all, even though resources are plentiful. Note: minDomains requires Kubernetes 1.30+ (GA) — if your kubeadm version predates that, this field is silently ignored; check kubectl version first.

**Fix**

```
kubectl patch deployment spread-test --type=json -p='[{"op":"replace","path":"/spec/template/spec/topologySpreadConstraints/0/minDomains","value":1}]'
```

**Debrief**: This is a pure topology-plugin failure with zero relationship to actual resource availability — a common trap is to see Pending and jump straight to describe nodes checking CPU/memory, when the real constraint has nothing to do with capacity at all. Read the Message before you assume which category of problem you're in.

**Lab 3 — Volume Node Affinity Conflict**

**Break**

```
cat <<'EOF' > bad-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: bad-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  hostPath:
    path: /tmp/bad-pv-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-south-1z-does-not-exist"]
EOF
kubectl apply -f bad-pv.yaml

cat <<'EOF' > bad-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bad-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  volumeName: bad-pv
  storageClassName: ""
EOF
kubectl apply -f bad-pvc.yaml
kubectl run pv-conflict --image=nginx --restart=Never --overrides='{"spec":{"volumes":[{"name":"vol","persistentVolumeClaim":{"claimName":"bad-pvc"}}],"containers":[{"name":"pv-conflict","image":"nginx","volumeMounts":[{"name":"vol","mountPath":"/data"}]}]}}'
```

**Predict: Will this show up as a PersistentVolumeClaim problem, or a pod scheduling problem?**

**Diagnose**

```
kubectl get pvc bad-pvc   # should show Bound — the PVC itself is fine
kubectl describe pod pv-conflict
```
**Expect: 0/3 nodes are available: 3 node(s) had volume node affinity conflict. — note the PVC is Bound, not Pending; the failure is entirely downstream, at pod scheduling, not at volume provisioning.**

```
kubectl delete pod pv-conflict
kubectl delete pvc bad-pvc
kubectl delete pv bad-pv
# recreate bad-pv.yaml with a real node label, or without nodeAffinity at all, then reapply
```
**Debrief**: This is the exact production incident pattern behind "EBS volume in the wrong AZ" — the PVC binds successfully (storage exists, capacity checks out), so if you only check kubectl get pvc you'll see Bound and wrongly conclude storage isn't the problem. The conflict only surfaces one layer up, at the scheduler's Filter phase, which is why VolumeBinding is listed as a scheduler plugin, not a storage-controller check.

**Lab 4 — Priority-Based Preemption**

**Break** — first fill your nodes with low-priority pods close to capacity. Adjust the CPU value below based on what you saw in kubectl describe nodes at the top of this module — aim for ~80% of one node's allocatable CPU per replica pair.

```
kubectl create priorityclass low-priority --value=100 --description="batch workloads"
kubectl create priorityclass high-priority --value=100000 --description="critical services"

cat <<'EOF' > low-pri.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: low-pri-filler
spec:
  replicas: 4
  selector:
    matchLabels: {app: low-pri-filler}
  template:
    metadata:
      labels: {app: low-pri-filler}
    spec:
      priorityClassName: low-priority
      containers:
      - name: filler
        image: nginx
        resources:
          requests:
            cpu: "500m"    # tune this against your real allocatable CPU
            memory: "256Mi"
EOF
kubectl apply -f low-pri.yaml
kubectl get pods -o wide   # confirm these fill up meaningful capacity on both workers
```

**Now submit a high-priority pod asking for more than what's currently free:**

```
cat <<'EOF' > high-pri.yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-pri-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: important
    image: nginx
    resources:
      requests:
        cpu: "1500m"    # sized to require evicting at least one low-pri-filler pod
        memory: "512Mi"
EOF
kubectl apply -f high-pri.yaml
```
**Predict**: Will high-pri-pod go Pending and stay Pending, or will something happen to the existing low-pri-filler pods?

**Diagnose**
```
kubectl get pods -o wide -w
kubectl describe pod high-pri-pod
```






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

Watch for status.nominatedNodeName appearing on high-pri-pod, and one of the low-pri-filler pods transitioning to Terminating shortly after — that's the preemption Reserve→Bind sequence playing out in real time. kubectl describe pod high-pri-pod events should show a Preempted reference.

**Fix**: nothing broken — this is the system working as designed. Clean up:

```
kubectl delete deployment low-pri-filler
kubectl delete pod high-pri-pod
kubectl delete priorityclass low-priority high-priority
```

**Debrief**: This is the lab most people never run in practice, and it's exactly the one interviewers probe on. Note the eviction was graceful — the victim got its full terminationGracePeriodSeconds, it wasn't SIGKILLed instantly. If you'd sized high-pri-pod's request so that evicting one filler still wasn't enough, you'd see multiple simultaneous evictions, all nominated toward making room for the single pending pod.

**Lab 5 — NoExecute Taint Evicts Running Pods**

**Break**

```
kubectl run noexec-victim --image=nginx --restart=Never
kubectl get pod noexec-victim -o wide   # note the node it landed on
kubectl taint nodes <that-node> experiment=true:NoExecute
```

**Predict**: Contrast with Module 1 Lab 2 (NoSchedule) — will this existing, already-running pod be affected at all?

**Diagnose**

```
kubectl get pod noexec-victim -w
```

Unlike NoSchedule (which only blocks new placements), NoExecute actively evicts pods already running on the node that lack a matching toleration — you should see it move to Terminating almost immediately.

**Fix / extended test** — add a toleration with a grace period and see the delayed version:

```
kubectl taint nodes <that-node> experiment-
kubectl run noexec-victim2 --image=nginx --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"experiment","operator":"Equal","value":"true","effect":"NoExecute","tolerationSeconds":30}]}}'
kubectl taint nodes <that-node> experiment=true:NoExecute
kubectl get pod noexec-victim2 -w
```

This one survives ~30 seconds before eviction — tolerationSeconds buys time, it doesn't grant permanence.

**Debrief**: This is the taint-effect distinction interviewers specifically check: NoSchedule = gatekeeper for new arrivals only; PreferNoSchedule = soft scoring hint; NoExecute = active eviction of anyone not tolerating it, with tolerationSeconds as the only lever controlling how long they're allowed to stay. This is also the exact mechanism behind node.kubernetes.io/not-ready:NoExecute and node.kubernetes.io/unreachable:NoExecute — the automatic taints Kubernetes applies when a node fails health checks, which is why pods eventually get rescheduled off a dead node without you doing anything manually.

**Lab 6 — Static Pods Bypass the Scheduler Entirely**

This is exploration, not a break/fix — and it connects directly to the static-pod-corruption chaos lab you've already run.

```
# on the control-plane node
sudo ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A2 "^spec:"

# back on your normal kubectl context
kubectl get pods -n kube-system -o wide | grep kube-apiserver
kubectl get pod -n kube-system kube-apiserver-<control-plane-node-name> -o yaml | grep -A3 "ownerReferences"
```

**Predict**: What happens if you kubectl delete this pod?

```
kubectl delete pod -n kube-system kube-apiserver-<control-plane-node-name>
kubectl get pods -n kube-system -o wide | grep kube-apiserver -w
```

It comes right back — usually within seconds, often before you even finish reading the output.

**Debrief**: ownerReferences on a static pod's mirror object points to Node, not a ReplicaSet or DaemonSet — there's no controller reconciling it via the API server at all. Kubelet watches /etc/kubernetes/manifests directly and recreates from disk the instant it notices the container gone; kubectl delete only removes the mirror object in etcd, which kubelet immediately re-creates once it reconciles. This is exactly why FailedScheduling is structurally impossible for a static pod — it never enters the scheduling queue, nodeName is fixed at manifest-authoring time, and it can't be preempted, cordoned away from, or affected by taints the normal way. If you ever need to actually change one, you edit the YAML file on disk, not kubectl edit.

**Module 4 — Networking & DNS Failures (CNI / kube-proxy / CoreDNS)**

This is usually the module where the "I understand K8s but can't read errors" gap is widest — because unlike a Pod's describe output, network failures often produce no Kubernetes-level error at all. The Service looks fine, the pod looks fine, and the connection just hangs. This module is about knowing which of three completely separate systems (CNI, kube-proxy, CoreDNS) to interrogate when that happens.

**The 'Why' (The Problem)**

Docker's default networking was single-host: containers on one machine could talk via a local bridge, but nothing routed cleanly across hosts, and port-mapping/linking was manual, brittle NAT juggling. Kubernetes needed three things Docker never solved: (1) every pod gets a real, routable IP, reachable from any other pod on any node, with no NAT between pods (the "flat network" model); (2) a stable virtual address for a constantly-churning set of pod replicas, so clients never hardcode a pod IP that might not exist five minutes from now; (3) name-based discovery, so "talk to the payments service" doesn't require knowing an IP at all. CNI solves (1), Services + kube-proxy solve (2), CoreDNS solves (3) — and critically, they are three independent systems that can each fail without touching the other two, which is exactly why "the network is broken" is close to a useless problem statement until you've isolated which of the three it actually is.

**Deep-Dive Mechanics**

**Layer 1 — CNI: how a pod gets an IP and a route.** When kubelet calls RunPodSandbox (Module 2), after the pause container's network namespace is created, kubelet invokes the CNI plugin binary with that namespace path. The plugin: allocates an IP from the node's podCIDR block via its IPAM component, creates a veth pair (one end stays in the pod's netns as eth0, the other end is plumbed into the host root namespace), and wires up routing so traffic can reach other nodes' pod CIDRs. How it reaches other nodes splits CNI plugins into two families:

- Overlay (e.g., Flannel's VXLAN backend): pod-to-pod traffic across nodes gets encapsulated inside a UDP packet (VXLAN, typically UDP/8472) between the nodes' real IPs, then decapsulated on arrival. Simple, works over any underlying network, but adds encapsulation overhead and hides the real pod IP from anything sniffing the physical link.
- Native routing (e.g., Calico in BGP mode): each node advertises its pod CIDR block as a real route via BGP peering with other nodes; packets travel as plain routed IP traffic, no encapsulation. Faster, but depends on the underlying network actually permitting that traffic (a cloud VPC's security groups need to allow it).

**Layer 2 — kube-proxy:** how a Service ClusterIP resolves to a real pod. A ClusterIP is not bound to any interface anywhere — you cannot ping it, ARP for it, or find it with ip addr. It's purely a rule target. kube-proxy watches Service and EndpointSlice objects via the API server and programs the actual traffic redirection, in one of two dataplanes:

- iptables mode (older default): a chain of NAT rules — KUBE-SERVICES matches the ClusterIP:port, jumps to a per-Service chain KUBE-SVC-<hash>, which does probabilistic selection among KUBE-SEP-<hash> chains (one per endpoint) using --probability, each of which does the actual DNAT to a real pod IP:port. This is a linear rule scan — at cluster scale (thousands of Services) this becomes measurably slow.
- IPVS mode: uses the kernel's IPVS load balancer instead — real load-balancing algorithms (round-robin, least-conn, etc.), O(1) lookup via hash tables regardless of Service count. Requires IPVS kernel modules loaded on the node.

The critical shared behavior across both modes: DNAT decisions happen on new connections. Once a connection is established, the kernel's conntrack table remembers the flow and routes subsequent packets on that same socket directly, bypassing the iptables/IPVS decision entirely. This is the root of one of the nastiest network incidents in Kubernetes — see Lab 3.

**Layer 3 — CoreDNS: how names resolve.** Every pod's /etc/resolv.conf (with default dnsPolicy: ClusterFirst) points its nameserver at the kube-dns Service's ClusterIP. CoreDNS pods (a Deployment, typically 2 replicas for HA) read a Corefile that defines the resolution logic: the kubernetes plugin watches the API server directly and answers anything under <service>.<namespace>.svc.cluster.local from live Service/EndpointSlice data (not from a database — it's always current); anything outside cluster.local gets handed to the forward plugin, which sends it to an upstream resolver (often the node's own /etc/resolv.conf, or an explicit list).

The part that catches people off guard: the DNS search list. A pod's resolv.conf doesn't just list one domain — it lists a search path: <namespace>.svc.cluster.local svc.cluster.local cluster.local (plus whatever the node itself adds, e.g. ec2.internal on AWS), and options ndots:5. ndots:5 means: if the name you're looking up has fewer than 5 dots, the resolver tries every entry in the search list first, appending each suffix, before ever trying the name as-is. So looking up api.stripe.com (2 dots, external, nothing to do with the cluster) triggers up to 4 failed internal lookups (api.stripe.com.default.svc.cluster.local, api.stripe.com.svc.cluster.local, api.stripe.com.cluster.local, api.stripe.com.ec2.internal) before the 5th attempt — the bare name — finally succeeds. Every one of those is a real round trip to CoreDNS. This is silent, unlogged-by-default latency that shows up as "external API calls are randomly slow" incidents.

**NetworkPolicy enforcement is not kube-proxy's job** — it's the CNI's. kube-proxy only programs Service load-balancing; it has zero concept of "allow" or "deny." NetworkPolicy objects are enforced by whatever component the CNI provides for it — Calico's Felix agent, Cilium's eBPF programs, etc. If your CNI doesn't implement NetworkPolicy support (plain Flannel, for instance), the API server will happily accept and store NetworkPolicy objects that are silently never enforced by anything. This is a genuinely dangerous gap — a team believes they've locked down traffic and haven't.

**The Alternative Landscape**

<img width="886" height="487" alt="image" src="https://github.com/user-attachments/assets/623663d6-aa36-400c-bf62-de9e6d5bf0c5" />

Why Calico (or similar) over bare Flannel for anything production-bound: NetworkPolicy is a baseline security control at any company handling payments or PII (Adyen, Revolut are exactly this) — shipping a CNI that silently no-ops policy objects is the kind of gap that fails a compliance audit. You'd reach for Cilium specifically when you need L7-aware policy or flow-level observability without adding a service mesh sidecar tax.

**Interview POV & Edge Cases**

The canonical prompt: "A pod can't reach another service — walk me through it." The answer they're grading is layer isolation, in order: (1) is this DNS-specific or all connectivity? — curl the Service's ClusterIP directly to rule DNS in/out; (2) if the IP works but the name doesn't, it's CoreDNS; (3) if neither works, check kubectl get endpoints — are there even any backing pods (readiness gating, Module 1); (4) if Endpoints look right but traffic still fails, check NetworkPolicy; (5) if it's cross-node specifically (same-node works, cross-node doesn't), that's your signature for a CNI/underlay problem, not a Kubernetes-object problem at all.

**Gotchas**:

- You cannot ping a ClusterIP. It has no interface, no ARP entry — ping will just hang or error. Testing Service reachability requires an actual protocol client (curl, nc) on the actual port.
- Stale conntrack entries after a pod is deleted/replaced — new connections route correctly instantly (fresh DNAT decision), but any connection that was already established before the pod died can hang or reset, because conntrack bypasses the iptables/IPVS rule chain entirely for existing flows. "It's fine on retry" is the tell.
- Default-deny NetworkPolicy + forgotten DNS egress rule — the single most common self-inflicted total-outage. The moment you apply an egress default-deny to a namespace, DNS lookups from those pods start failing too, because port 53 to CoreDNS is egress traffic like anything else, and people only think to allow the app's actual destination port.
- NetworkPolicies are additive, never subtractive. Multiple policies selecting the same pod are OR'd together — the union of all their allow rules — not AND'd. You cannot use a second NetworkPolicy to further restrict what a first one already allowed.
- A hang, not an instant refusal, is diagnostic. Connection refused = something's listening on the wrong port or nothing's listening at all (fast TCP RST). A long hang followed by a timeout = something is silently dropping the packet — usually NetworkPolicy, or, on your EC2-based cluster, a Security Group.


**The 'Better Way' (Evolution)**

At scale, iptables' linear rule-chain scanning becomes a real bottleneck — thousands of Services means thousands of sequential chain jumps per packet. Cilium's eBPF dataplane replaces iptables/kube-proxy entirely with O(1) hash-based lookups in the kernel, and Hubble gives you live, per-flow L3–L7 visibility (which pod talked to which pod, on what port, allowed or denied by which policy) without deploying a packet sniffer manually — this closes exactly the blind spot that makes Layer 1/2 failures so hard to see from kubectl alone. (L7/application-layer routing failures — Ingress and Gateway API specifically — are their own dedicated module coming up as Module 8, so we'll go deep on that separately.)

**Module 4 Lab Pack**

<img width="918" height="468" alt="image" src="https://github.com/user-attachments/assets/d2c1a7ce-7ccb-43f4-810c-64cccf1720be" />

**Lab 1 — Identify Your CNI + Baseline Connectivity**

```
bash
kubectl get pods -n kube-system -o wide
kubectl get daemonset -n kube-system
```

Look for calico-node, kube-flannel-ds, cilium, or weave-net — whichever DaemonSet is running your data plane. This determines exactly which port matters in Lab 6, and whether Lab 2/5 will actually enforce (Calico/Cilium — yes; plain Flannel — no, and that's itself the lesson).

```
bash
kubectl run net-a --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl run net-b --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl get pods -o wide   # confirm they landed on DIFFERENT worker nodes
kubectl exec net-a -- curl -sm3 -o /dev/null -w "%{http_code}\n" $(kubectl get pod net-b -o jsonpath='{.status.podIP}')
```

This is your cross-node pod-to-pod baseline — every later lab that breaks connectivity gets compared against this working case.

**Lab 2 — Default-Deny NetworkPolicy Blocks DNS**

**Break**

```
kubectl create ns netpol-test
kubectl run web --image=nginx -n netpol-test --labels=app=web
kubectl expose pod web -n netpol-test --port=80
kubectl run client --image=nicolaka/netshoot -n netpol-test --restart=Never -- sleep 3600

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes: ["Egress"]
EOF
```

**Predict**: Will client fail to resolve DNS, fail to reach web by IP, both, or neither?

**Diagnose**

```
kubectl exec -n netpol-test client -- nslookup web.netpol-test.svc.cluster.local
kubectl exec -n netpol-test client -- curl -sm3 $(kubectl get pod -n netpol-test web -o jsonpath='{.status.podIP}')
```

Both should hang and time out — an empty podSelector: {} with no egress rules blocks everything leaving the pod, including the DNS query itself.

**Fix**

```
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
kubectl exec -n netpol-test client -- nslookup web.netpol-test.svc.cluster.local
```

(kubernetes.io/metadata.name is auto-applied to every namespace since 1.21+ — no manual labeling needed.)

**Debrief**: If your CNI is plain Flannel with no policy engine, both nslookup and curl succeeded the whole time, even with the deny policy applied — confirm that with kubectl describe networkpolicy -n netpol-test (the object exists) versus the actual traffic result (unaffected). That gap is the lesson: NetworkPolicy objects existing in etcd tells you nothing about whether anything is enforcing them. If it's Calico/Cilium and it did block traffic: note that DNS died as pure collateral damage of a policy that was never really about DNS — this exact pattern is the single most common "everything is broken" ticket that turns out to be networking-team-adjacent.

**Lab 3 — Stale Conntrack Entry After Pod Deletion**

**Break**

```
kubectl create deployment conntrack-demo --image=nginx --replicas=1
kubectl expose deployment conntrack-demo --port=80
kubectl get pods -o wide -l app=conntrack-demo   # note pod name + its node
```

Open a persistent connection from a client pod (leave this running in its own terminal):

```
bash
kubectl run conn-client --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl exec -it conn-client -- nc -v $(kubectl get svc conntrack-demo -o jsonpath='{.spec.clusterIP}') 80
```

This opens and holds a raw TCP connection (no HTTP request needed — the handshake alone is enough to create a conntrack entry). Leave it open.

**Predict**: If you delete the backing pod right now, does this already-open connection get transparently redirected to a replacement pod, hang, or reset?

**Diagnose** — in a second terminal, on the worker node hosting the pod:

```
bash
sudo conntrack -L | grep <conntrack-demo-pod-ip>
```

You should see an ESTABLISHED entry mapping the client to that specific pod IP. Now delete the pod:

```
kubectl delete pod <conntrack-demo-pod-name>
kubectl get endpoints conntrack-demo   # new pod IP appears within seconds
```

Check back on the nc session — depending on your CNI's cleanup speed, you'll see one of two outcomes: it hangs with no data until the conntrack entry eventually times out, or it gets an immediate reset once the veth interface is torn down. Either outcome demonstrates the same underlying fact: iptables/IPVS never re-evaluates an already-established flow — only a brand new connection gets a fresh, correct DNAT decision.

**Fix / proof: open a fresh connection right now and confirm it routes correctly on the first try:**

```
kubectl exec -it conn-client -- nc -v -w2 $(kubectl get svc conntrack-demo -o jsonpath='{.spec.clusterIP}') 80
```

**Debrief**: This is the mechanism behind "the error went away when I retried" — a maddening non-error that has no Kubernetes Event, no log line, nothing in describe. It only shows up as application-level symptoms (hung requests, timeouts) during rolling deploys. The diagnostic signature: symptoms correlate with pod churn, and a brand-new connection always works fine while an old one doesn't.

**Lab 4 — CoreDNS Scaled to Zero**

**Break**

```
bash
kubectl -n kube-system scale deployment coredns --replicas=0
```

**Predict**: Will pod-to-pod connectivity by IP still work? Will Services still load-balance correctly if you already know the ClusterIP?

**Diagnose**

```
kubectl exec net-a -- nslookup net-b   # (from Lab 1's pods)
kubectl exec net-a -- curl -sm3 -o /dev/null -w "%{http_code}\n" <net-b's raw pod IP>
```

DNS resolution fails/times out completely; direct-IP connectivity is entirely unaffected — proof that CNI and DNS are fully independent failure domains.

**Fix**

```
kubectl -n kube-system scale deployment coredns --replicas=2
kubectl -n kube-system get pods -l k8s-app=kube-dns -w   # wait for Running + Ready
kubectl exec net-a -- nslookup net-b
```

**Debrief**: This is the cleanest possible demonstration that "the network is down" is too vague a symptom to act on — raw connectivity (CNI) and name resolution (CoreDNS) can fail completely independently of each other. Always test both explicitly rather than assuming one implies the other.

**Lab 5 — Selective NetworkPolicy Ingress (Additive Behavior)**

**Break**

```
kubectl create ns netpol-selective
kubectl run backend --image=nginx -n netpol-selective --labels=app=backend
kubectl expose pod backend -n netpol-selective --port=80
kubectl run allowed-client --image=busybox -n netpol-selective --labels=role=trusted --command -- sleep 3600
kubectl run blocked-client --image=busybox -n netpol-selective --labels=role=untrusted --command -- sleep 3600

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: only-trusted
  namespace: netpol-selective
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: trusted
EOF
```

**Predict**: Which client succeeds, which times out — and will the outcome differ if you add a second NetworkPolicy also selecting backend?

**Diagnose**

```
kubectl exec -n netpol-selective allowed-client -- wget -qO- --timeout=3 backend
kubectl exec -n netpol-selective blocked-client -- wget -qO- --timeout=3 backend
```

allowed-client succeeds, blocked-client hangs and times out (assuming your CNI enforces policy — see Lab 2's caveat if not).

**Fix / prove additive behavior** — add a second, independent policy also allowing traffic on a different label:

```
kubectl label pod blocked-client -n netpol-selective role=untrusted tier=frontend --overwrite
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: also-allow-frontend
  namespace: netpol-selective
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
EOF
kubectl exec -n netpol-selective blocked-client -- wget -qO- --timeout=3 backend
```

blocked-client now succeeds — even though only-trusted (which it still doesn't match) is still applied.

**Debrief**: This is the additive-policy gotcha made concrete — a second policy never narrows what a first one allowed; it only ever widens it. If someone believes they've "further restricted" backend access by adding a second, more specific NetworkPolicy, this lab shows exactly why that belief is wrong.

**Lab 6 — Security Group Blocks CNI Overlay/Peering Port (AWS-Specific)**

This is the layer below Kubernetes entirely — the underlying AWS network refusing to carry CNI traffic between your EC2 instances. 

**Do this carefully; only touch the specific rule you add, and remove it immediately after observing the effect.**

Identify your port first, based on Lab 1's CNI discovery:

- Flannel (VXLAN backend): UDP/8472
- Calico (VXLAN mode): UDP/4789 · Calico (IPIP mode): IP protocol 4 · Calico (BGP mode): TCP/179
- Cilium (VXLAN mode, default): UDP/8472

Break — find your worker nodes' security group and add a temporary deny-equivalent by removing the relevant allow rule (or use a NACL deny if your SG doesn't isolate it cleanly):

```
aws ec2 describe-instances --filters "Name=tag:Name,Values=<your-worker-tag>" \
  --query "Reservations[].Instances[].SecurityGroups"
# note the SG ID, then temporarily revoke the specific CNI port between nodes
aws ec2 revoke-security-group-ingress --group-id <sg-id> \
  --protocol udp --port 8472 --source-group <sg-id>
```

**Predict**: Will same-node pod-to-pod traffic be affected? Cross-node?

**Diagnose**

```
kubectl run same-node-test --image=nicolaka/netshoot --restart=Never --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"<same-node-as-net-a>"}}}' -- sleep 3600
kubectl exec net-a -- curl -sm3 -o /dev/null -w "same-node: %{http_code}\n" $(kubectl get pod same-node-test -o jsonpath='{.status.podIP}')
kubectl exec net-a -- curl -sm3 -o /dev/null -w "cross-node: %{http_code}\n" $(kubectl get pod net-b -o jsonpath='{.status.podIP}')
```

Same-node traffic (never leaves the host, no encapsulation involved) keeps working; cross-node traffic times out.

**Fix**

```
aws ec2 authorize-security-group-ingress --group-id <sg-id> \
  --protocol udp --port 8472 --source-group <sg-id>
kubectl exec net-a -- curl -sm3 -o /dev/null -w "cross-node restored: %{http_code}\n" $(kubectl get pod net-b -o jsonpath='{.status.podIP}')
```

**Debrief**: "Same-node works, cross-node doesn't" is the diagnostic signature that the problem lives below Kubernetes entirely — in the VPC/SG/NACL layer, not in any Kubernetes object. No amount of staring at kubectl describe will surface this; nothing in the cluster's own state is wrong. This is a real, recurring incident category at any company running self-managed kubeadm on EC2 rather than EKS (where AWS manages this wiring for you) — exactly your setup.

**Lab 7 — ndots:5 DNS Query Amplification**

**Break**: nothing to break — this is observing default behavior.

```
kubectl exec net-a -- cat /etc/resolv.conf
```

Note the search line and options ndots:5.

**Predict**: For an external name like example.com (1 dot), how many actual DNS queries do you expect one nslookup to generate?

**Diagnose** — capture on the node running CoreDNS while you trigger one lookup:

```
kubectl get pods -n kube-system -o wide -l k8s-app=kube-dns   # note the node
```

*SSH to that node in one terminal:*

```
sudo tcpdump -i any -n udp port 53 -l
```

In another terminal, trigger a single external lookup:

```
kubectl exec net-a -- nslookup example.com
```

Count the distinct query packets in the tcpdump output — expect 4–5 queries (example.com.default.svc.cluster.local, .svc.cluster.local, .cluster.local, possibly a node search domain, each returning NXDOMAIN, before the final bare example.com. succeeds).

**Fix / comparison** — a fully-qualified name (trailing dot) skips the search list entirely:

```
kubectl exec net-a -- nslookup example.com.
```

Watch the same tcpdump session — this time, one query only.

**Debrief**: This is invisible in every dashboard that only tracks HTTP/application latency — it lives purely at the DNS layer, and by default CoreDNS doesn't even log queries (the log plugin is off by default). At real production scale, this is a genuine multiplier on CoreDNS load and a real source of intermittent external-call latency, and the fix in application code (using trailing-dot FQDNs, or reducing unnecessary search domains) is something very few engineers think to check until they've done exactly this exercise once.

**Module 5 — Storage Failures (CSI Driver Internals & the Attach/Mount Lifecycle)**

This module tends to surprise people because storage failures span the widest range of layers of anything so far — a stuck PVC can be a Kubernetes object problem, a cloud API permissions problem, or a raw block-device problem, and kubectl describe often shows you almost nothing useful for the middle one.

**The 'Why' (The Problem)**

Before CSI, volume plugins were compiled directly into Kubernetes core — kubelet and the controller-manager shipped with hardcoded, vendor-specific code for AWS EBS, GCE PD, Azure Disk, vSphere, and others baked right into the binary (the "in-tree" plugins). This meant a storage vendor couldn't fix a bug or ship a feature without getting a PR merged into upstream Kubernetes and waiting for the next full Kubernetes release — vendor velocity was hostage to the Kubernetes release cycle, and the core binary kept bloating with every new cloud's volume logic. CSI (Container Storage Interface) solved this the same way CRI solved it for runtimes (Module 2): a standard, versioned gRPC contract that lets any storage vendor ship an out-of-tree driver, deployed and upgraded independently, with zero changes to Kubernetes core. By the way — this isn't optional history: in-tree cloud volume plugins were removed from Kubernetes entirely as of 1.27+, so CSI is mandatory for cloud-backed storage on any current cluster, yours included.

**Deep-Dive Mechanics**

**The CSI driver is two separate deployments, not one. Every CSI driver ships as:**

- Controller plugin — a Deployment (leader-elected, one active replica) that talks to the cloud API (EBS CreateVolume, AttachVolume, etc.). It doesn't need to run on every node — it just needs cloud credentials and network access to AWS's control plane.
- Node plugin — a DaemonSet, one pod per node, that does the actual local block-device work: formatting, mounting, unmounting. It needs to run everywhere a pod might need a volume.

Neither talks to the cloud API or the node directly by itself — both wrap a set of sidecar containers that translate Kubernetes-object watches into CSI gRPC calls against the vendor's driver container in the same pod: csi-provisioner (watches PVCs → CreateVolume), csi-attacher (watches VolumeAttachment objects → ControllerPublishVolume), csi-resizer (watches PVC size edits → ControllerExpandVolume), node-driver-registrar (registers the node plugin with kubelet), and a livenessprobe. This sidecar pattern is why a CSI driver pod in kubectl get pods shows 3-4 containers, not one.

**The full lifecycle, in order:**

```
1. PVC created
   → (WaitForFirstConsumer: paused here until a pod needs it — see Module 3)
2. csi-provisioner sees PVC → calls CreateVolume RPC → AWS creates the EBS volume
   → PV object created, bound to the PVC
3. Pod scheduled to a node (VolumeBinding filter already confirmed AZ match)
4. csi-attacher creates a VolumeAttachment object → calls ControllerPublishVolume
   → AWS attaches the EBS volume to that specific EC2 instance
5. kubelet's volume manager on the node calls:
   a. NodeStageVolume  — format (if needed) + mount to a per-node staging path
   b. NodePublishVolume — bind-mount from staging path into the pod's actual mount path
6. Container starts with the volume already mounted
```

Steps 1–2 and 4 happen on the controller (control-plane-adjacent, cloud-API-facing). Step 5 happens on the node the pod landed on. That split is the single most important thing to internalize for troubleshooting — a stuck volume could be failing at a step that has nothing to do with the node your pod is on at all.

**The VolumeAttachment object is the ground truth for "is this volume attached, and where."** It's easy to overlook because nothing in a normal workflow ever has you look at it directly — but it's the API-level record of the controller's attach state, and it persists independently of pod or even node lifecycle. A VolumeAttachment left behind after a node dies ungracefully is the direct cause of the most common storage incident in Kubernetes (Lab 4).

**Access modes are a hard cloud-storage constraint, not a Kubernetes policy choice.** EBS is fundamentally block storage attachable to exactly one instance at a time — it can only ever support ReadWriteOnce. Scaling a Deployment to 2+ replicas that all reference the same EBS-backed PVC doesn't create 2 volumes; it creates 1 volume 2 pods are fighting to attach, and the loser gets stuck. Real ReadWriteMany requires network filesystem storage (EFS/NFS) — an entirely different CSI driver, not a config flag on the EBS one.

**Reclaim policy determines what happens to the underlying cloud resource, not just the K8s object.** Delete (default for most dynamic StorageClasses): deleting the PVC triggers DeleteVolume — the actual EBS volume is destroyed, data gone. Retain: the PV object survives in a Released state, the EBS volume itself is left alone in AWS, but it's not automatically reusable — a human has to either delete it manually or clear claimRef on the PV to rebind it. This is a deliberate data-safety default for StatefulSets and databases, and it's exactly why "I deleted my StatefulSet and my PVCs are still there costing money a week later" is such a common, non-obvious surprise.

**The Alternative Landscape**

<img width="877" height="405" alt="image" src="https://github.com/user-attachments/assets/57d472e1-68ea-4aa5-8f49-25b3d45d2eb8" />

<img width="858" height="256" alt="image" src="https://github.com/user-attachments/assets/653ee69c-18f2-4458-8f8e-64e2e529fa24" />

You'd default to EBS for the overwhelming majority of stateful workloads at companies like Adyen or Revolut (single-writer databases, message queue disks) specifically because RWO is what you actually want most of the time — multi-writer access to the same block volume is rarely what a correctly-designed stateful app needs anyway. You'd reach for EFS specifically when multiple pods genuinely need concurrent write access to the same files, and Local PV specifically when you've deliberately pushed replication up to the application layer (e.g. a Kafka cluster) and want to trade node-loss durability for raw I/O speed.

**Interview POV & Edge Cases**

Classic prompt: "A pod is stuck in ContainerCreating because of a volume — walk me through your triage." The layered answer they want: check PVC status first (Bound vs Pending — tells you if provisioning even succeeded); if Bound, check for a VolumeAttachment object and its state; if attach looks fine, the problem is node-local — check the CSI node plugin pod's logs on that specific node, not the controller's. Weak candidates jump straight to kubectl describe pod and stop there when it just says "waiting for volume" with no further detail — that message is often the symptom, and the real error lives in a completely different pod's logs (the CSI controller), not the failing pod's own Events.

**Gotchas**:

- Multi-Attach error the instant someone scales an EBS-backed Deployment past 1 replica — this is an access-mode limitation, not a bug, and no amount of retrying fixes it.
- A VolumeAttachment surviving an ungracefully terminated node (spot reclaim, hard power-off) blocks any other node from attaching that same volume — AWS still thinks it's attached to a host that no longer exists from Kubernetes' point of view. The dangerous mistake here is force-deleting the VolumeAttachment when you're not actually certain the instance is gone — if that node comes back and still has the volume attached at the OS level while another pod has also mounted it elsewhere, that's real data corruption, not just an inconvenience.
- StatefulSet PVCs are not deleted when the StatefulSet is deleted, by default. This surprises people in both directions — unexpected lingering cost, or unexpectedly getting old data back when a StatefulSet is recreated with the same name. persistentVolumeClaimRetentionPolicy (GA in newer versions) lets you make this explicit instead of relying on the surprising default.
- ImmediateBinding + multi-AZ workers provisions a volume before the scheduler has picked a node — if the pod later lands in a different AZ, you get a permanently stuck volume node affinity conflict (this is exactly Module 3 Lab 3's failure mode, now from the dynamic provisioning side instead of a manually-crafted PV).
- On a self-managed kubeadm cluster (yours), there's no IRSA out of the box. EKS gives pods scoped IAM roles via an OIDC provider automatically; on kubeadm, the CSI controller pod's AWS credentials typically come from the underlying EC2 instance's instance profile role — whatever node the controller pod happens to land on needs that role attached, or every cloud API call fails with an access-denied error visible only in the CSI controller's own pod logs, never in kubectl describe pvc.

**The 'Better Way' (Evolution)**

**Generic ephemeral volumes** (inline in the pod spec) handle scratch-space needs without the full PVC/PV lifecycle overhead at all — useful for anything that doesn't need to survive the pod. CSI VolumeSnapshots give every CSI driver a standardized backup/restore/clone primitive instead of vendor-specific scripts. And architecturally, the biggest "better way" for a lot of workloads is sidestepping the attach/detach lifecycle's entire failure surface by going object-storage-first (S3) wherever the access pattern allows it — no VolumeAttachment, no AZ-affinity, no attach/detach controller involved at all. Worth noting operationally: fast node churn from Karpenter-style autoscaling or spot consolidation makes the stuck-VolumeAttachment failure mode more common, not less — which is exactly why proper node draining before termination matters more, not less, as clusters get more elastic.

**Module 5 Lab Pack**

<img width="900" height="452" alt="image" src="https://github.com/user-attachments/assets/c6b6a760-b4ea-43ee-abe4-5c19fb493ed1" />

**Lab 1 — CSI Driver / IAM Permissions**

**Break / discover**

```
bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl get sc
```

If nothing shows up, install it deliberately before attaching any IAM permissions, so you hit the real gotcha:

```
bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver -n kube-system --create-namespace
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```
**Create a StorageClass and a PVC:**

```
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 5Gi
EOF
kubectl run vol-test --image=nginx --overrides='{"spec":{"containers":[{"name":"vol-test","image":"nginx","volumeMounts":[{"name":"v","mountPath":"/data"}]}],"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"test-claim"}}]}}'
```

**Predict**: Where will the real error message live — kubectl describe pvc, kubectl describe pod, or somewhere else entirely?

**Diagnose**

```
bash
kubectl get pvc test-claim   # stuck Pending
kubectl describe pvc test-claim   # usually just "waiting for a volume to be created" — not useful
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver -c csi-provisioner --tail=50
```

Expect an AccessDenied / UnauthorizedOperation error in the provisioner sidecar's logs — nowhere in the PVC or pod's own description.

**Fix** — find and grant the instance role EBS permissions:

```
# find the role attached to whichever worker the controller pod landed on
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver -o wide
aws ec2 describe-instances --filters "Name=private-dns-name,Values=<that-node-name>" \
  --query "Reservations[].Instances[].IamInstanceProfile.Arn"
aws iam attach-role-policy --role-name <role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
kubectl get pvc test-claim -w
```

**Debrief**: This is the defining lesson of the module — the failing object (test-claim) tells you almost nothing, while the actual error sits in a sidecar container's logs on a completely different pod. This exact gap (controller-side cloud-API failure, invisible at the PVC level) is the reason experienced engineers check CSI controller logs early rather than staring at describe pvc repeatedly hoping for more detail that isn't coming.

**Lab 2 — Immediate Binding + AZ Mismatch (Dynamic Provisioning)**

**Break** — first check whether your two workers are actually in different AZs (this determines whether you'll see the failure at all):

```
bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:'.metadata.labels.topology\.kubernetes\.io/zone'
```

If they're in the same AZ, skip straight to the Debrief — Immediate binding can't mismatch what only has one possible AZ, and that null result is itself worth understanding. If different AZs

```
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-immediate
provisioner: ebs.csi.aws.com
volumeBindingMode: Immediate
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immediate-claim
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ebs-immediate
  resources:
    requests:
      storage: 5Gi
EOF
kubectl get pvc immediate-claim -w   # note which AZ it actually provisioned into
```

Now force the pod onto the other AZ's node:

```
bash
kubectl run az-mismatch --image=nginx \
  --overrides='{"spec":{"nodeSelector":{"topology.kubernetes.io/zone":"<the-OTHER-az>"},"containers":[{"name":"az-mismatch","image":"nginx","volumeMounts":[{"name":"v","mountPath":"/data"}]}],"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"immediate-claim"}}]}}'
```

**Predict**: Is this fixable by waiting, or is it permanent?

**Diagnose**

```
bash
kubectl describe pod az-mismatch
```

Expect volume node affinity conflict — permanent, no amount of retrying resolves it, because the physical EBS volume genuinely cannot move AZs.

**Fix**

```
kubectl delete pod az-mismatch
kubectl delete pvc immediate-claim
# recreate the PVC using the ebs-sc StorageClass from Lab 1 (WaitForFirstConsumer) instead
```

**Debrief**: This is Module 3 Lab 3's manually-constructed scenario happening for real, with dynamic provisioning — and it's the concrete reason WaitForFirstConsumer is the correct default for almost every real StorageClass: it delays provisioning until the scheduler has already committed to a node, so the CSI driver provisions in the right AZ automatically instead of gambling on Immediate.

**Lab 3 — Multi-Attach Error**

**Break**

```
kubectl create deployment multiattach-demo --image=nginx --replicas=1
kubectl set volumes deployment multiattach-demo --add --name=data --type=persistentVolumeClaim \
  --claim-name=multiattach-pvc --claim-size=5Gi --claim-class=ebs-sc --mount-path=/data
kubectl get pvc multiattach-pvc -w   # wait for Bound
kubectl scale deployment multiattach-demo --replicas=2
```

**Predict**: Will the second replica go Pending (scheduler-level) or get scheduled and fail later (attach-level)?

**Diagnose**

```
bash
kubectl get pods -l app=multiattach-demo -o wide
kubectl describe pod <second-replica-pod>
```


Expect the second pod to get scheduled just fine (nothing about the PVC blocks scheduling itself), then sit in ContainerCreating with an Event like Multi-Attach error for volume "..." Volume is already exclusively attached to one node and can't be attached to another.

**Fix**

```
bash
kubectl scale deployment multiattach-demo --replicas=1
```

(The real fix for genuinely needing 2+ replicas with shared storage is switching to an EFS-backed RWX StorageClass, or giving each replica its own PVC via a StatefulSet's volumeClaimTemplates if they don't actually need to share data.)

**Debrief**: Note this failed at the attach step, not the schedule step — the scheduler has no concept of "this PVC is already attached elsewhere," because VolumeBinding (Module 3) only checks AZ/node-affinity compatibility, not exclusivity. Exclusivity is enforced later, by the attacher, which is exactly why this shows up as ContainerCreating + a Multi-Attach Event rather than Pending + FailedScheduling.

**Lab 4 — Stuck VolumeAttachment After Node Goes Unresponsive**

**Break**

```
kubectl create deployment stuck-attach-demo --image=nginx --replicas=1
kubectl set volumes deployment stuck-attach-demo --add --name=data --type=persistentVolumeClaim \
  --claim-name=stuck-attach-pvc --claim-size=5Gi --claim-class=ebs-sc --mount-path=/data
kubectl get pods -l app=stuck-attach-demo -o wide   # note the node
kubectl get volumeattachment   # find the one referencing your PVC's PV
```

Simulate an unresponsive node without actually terminating the EC2 instance (safer, fully reversible) — SSH to that worker and stop kubelet:

```
bash
ssh -i <key>.pem ubuntu@<that-worker-ip>
sudo systemctl stop kubelet
```

**Predict**: Once the node goes NotReady, will the Deployment's replacement pod (rescheduled to the other worker) get its volume attached cleanly?

**Diagnose** — back on the master, wait for NotReady and the pod reschedule:

```
kubectl get nodes -w
kubectl get pods -l app=stuck-attach-demo -o wide
kubectl describe pod <new-pod-on-other-worker>
kubectl get volumeattachment
```

Expect the new pod stuck in ContainerCreating, and the original VolumeAttachment object still present, still pointing at the down node — Kubernetes hasn't detached it because it can't confirm the old node has actually released it.

**Fix** — restart kubelet on the original node rather than force-deleting anything (the safe resolution path):

```
bash
# back on the original worker
sudo systemctl start kubelet
bash
# back on master — watch the attach/detach controller clean up automatically
kubectl get volumeattachment -w
kubectl get pods -l app=stuck-attach-demo -o wide -w
```

**Debrief**: This is deliberately built to demonstrate the safe resolution path, not the dangerous shortcut. You'll see kubectl delete volumeattachment <name> --force mentioned in incident runbooks online — that command exists for genuine node-loss scenarios (terminated instance, confirmed gone) but it bypasses the safety check that prevents double-attachment. If the node were actually still alive and still had the volume mounted at the OS level, force-deleting the VolumeAttachment and letting another node attach the same volume is a real data-corruption path. The correct default instinct is "wait for the attach/detach controller's own timeout-based reconciliation" or "confirm the node is truly gone first" — not "force it immediately because the pod is stuck."

**Lab 5 — Volume Expansion (Controller vs Node Expand)**

**Break** — first confirm your StorageClass allows it:

```
bash
kubectl get sc ebs-sc -o yaml | grep allowVolumeExpansion
```

**If missing:**

```
bash
kubectl patch sc ebs-sc -p '{"allowVolumeExpansion": true}'
```

Now grow the PVC from Lab 1:

```
bash
kubectl patch pvc test-claim -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

**Predict**: Does the filesystem inside the running pod grow immediately, or is another step required?

**Diagnose**

```
bash
kubectl get pvc test-claim -w   # watch conditions
kubectl describe pvc test-claim   # look for Resizing / FileSystemResizePending conditions
kubectl exec vol-test -- df -h /data
```

You'll typically see the PVC's .status.capacity update relatively fast (controller-side ControllerExpandVolume — the EBS API call), but df -h inside the pod may still show the old size briefly, flagged by a FileSystemResizePending condition until kubelet performs NodeExpandVolume on its next sync.

**Fix / confirm** — if it doesn't resolve within a minute or two on its own:

```
bash
kubectl exec vol-test -- df -h /data   # re-check after kubelet's next sync
```

**Debrief**: This two-phase behavior (ControllerExpandVolume grows the cloud volume; NodeExpandVolume grows the filesystem inside it) is exactly the controller/node split from the Deep-Dive playing out again — a resize that "worked" at the cloud level can still show the old size to the application until the node-side step catches up. Depending on your exact Kubernetes/driver version, some combinations require the pod to restart for the filesystem-level growth to apply — worth explicitly checking your version's behavior rather than assuming online expansion always works with zero disruption.

**Lab 6 — StatefulSet PVC Retention + Reclaim Policy**

**Break**

```
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-demo
spec:
  serviceName: sts-demo
  replicas: 1
  selector:
    matchLabels: {app: sts-demo}
  template:
    metadata:
      labels: {app: sts-demo}
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 2Gi
EOF
kubectl get pvc -l app=sts-demo
kubectl exec sts-demo-0 -- sh -c "echo 'important data' > /data/marker.txt"
```

**Predict**: If you delete the StatefulSet right now, does its PVC get deleted too?

**Diagnose / Fix cycle**

```
bash
kubectl delete statefulset sts-demo
kubectl get pvc -l app=sts-demo   # still there
kubectl apply -f - <<'EOF'
# (reapply the same StatefulSet manifest from above)
EOF
kubectl exec sts-demo-0 -- cat /data/marker.txt
```

The recreated pod reattaches the same PVC and your marker file is still there — StatefulSet deletion never touched the PVC.

Now test reclaim policy directly:

```
kubectl get pv $(kubectl get pvc -l app=sts-demo -o jsonpath='{.items[0].spec.volumeName}') -o yaml | grep persistentVolumeReclaimPolicy
kubectl delete statefulset sts-demo
kubectl delete pvc -l app=sts-demo
kubectl get pv   # check whether the PV (and underlying EBS volume) is gone or sitting in Released
```

**Debrief**: If the policy was Delete (typical default for dynamic StorageClasses), the PV and the real EBS volume are both gone now — permanently, along with your marker file. If it were Retain, the PV would sit in Released state indefinitely, costing money, invisible unless someone explicitly checks kubectl get pv. Neither behavior is a bug — but assuming the wrong one in either direction is a genuine production incident category: either "I thought deleting the StatefulSet would clean up storage costs and it didn't," or worse, "I thought deleting the StatefulSet would clean up storage and it silently destroyed data I needed."



















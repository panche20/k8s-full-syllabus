# Kubernetes Troubleshooting MasterClass

## The Roadmap

<img width="912" height="538" alt="image" src="https://github.com/user-attachments/assets/8b86eb22-1332-4194-96b8-90bfedd61cda" />



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

<img width="916" height="550" alt="image" src="https://github.com/user-attachments/assets/cf2eacc0-d585-465e-84d6-41075f595782" />

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

**Module 6 — Control Plane Failures (etcd / API Server / Controller-Manager)**

You've already run real chaos labs against etcd, static pod corruption, network partitions, admission webhook lockout, and cgroup driver mismatch on this exact cluster — so this module builds underneath that hands-on experience rather than repeating it: the theory for why those failures manifested the way they did, plus new labs covering ground you haven't hit yet (including that apiserver TLS cert corruption lab you scoped but never ran).

**The 'Why' (The Problem)**

Distributed cluster state needs consensus — multiple things need to agree on "what is true right now" even when nodes fail or messages get delayed. Before Kubernetes, systems like Google's Borg solved this with a tightly-coupled, purpose-built consensus service (Chubby) baked into the orchestrator itself — every component that needed cluster state had to understand distributed-consensus semantics directly. Kubernetes made a deliberate architectural choice to avoid that: only one component, kube-apiserver, is ever allowed to talk to etcd. Every controller, every scheduler, every kubelet, every kubectl command only ever speaks a uniform REST+watch HTTP API to the API server. This means only one piece of the entire system needs to understand Raft consensus, watch resumption, and MVCC compaction — everything else just lists and watches objects through a simple interface. It's the exact same "isolate the hard distributed-systems problem behind one interface" pattern as CRI (Module 2) and CSI (Module 5), applied to state storage.

**Deep-Dive Mechanics**

**etcd is Raft consensus underneath a key-value store.** Every write goes through leader election and log replication, and requires a quorum — a majority, (N/2)+1 — of members to acknowledge before it's committed. This number matters enormously for you specifically: your troubleshooting cluster runs single-member, stacked etcd (colocated with the one control-plane node) — quorum there is trivially 1, meaning that one etcd instance is the entire consensus group. Lose it, and you have zero availability, instantly, with no failover possible. Your separate [[multi-region-kubeadm]] cluster runs external etcd with multiple members specifically so quorum survives a single node loss (3 members → quorum 2, tolerates 1 failure). Same software, radically different failure tolerance, purely from topology.

**Watch + resourceVersion + compaction.** etcd is MVCC (multi-version concurrency control) — every write bumps a global monotonically-increasing revision, and old revisions are retained until compaction removes them. kube-apiserver's watches (and every controller's informer, and kubectl get --watch) work by listing at some revision, then asking to resume the watch stream from that resourceVersion. If a watcher falls far enough behind that etcd has already compacted past the revision it's asking for, it gets "too old resource version" — this is not data loss or corruption, it's an expected MVCC boundary condition, and the only correct response (which every well-written client does automatically) is a fresh List, not a repair action. Compaction itself only marks old revisions as reclaimable — it doesn't shrink the actual .db file on disk. That requires a separate defrag operation, which is I/O-intensive and, on a single-member cluster, briefly pauses that member entirely (in true multi-member HA, you defrag one member at a time so the others keep serving).

**etcd's NOSPACE alarm is a real, cluster-halting failure mode.** etcd enforces a backend quota (default 2GB) via --quota-backend-bytes. Once the database hits that size, etcd raises a NOSPACE alarm and goes read-only cluster-wide — every write anywhere in the cluster fails, not just etcd-internal ones, because kube-apiserver's writes are etcd writes. Reads keep working (this is often the confusing part — the cluster looks "half-fine"). Recovery is a specific sequence: delete/reduce data → compact → defrag → explicitly disarm the alarm. Miss the disarm step and writes stay blocked even after space is reclaimed — the alarm doesn't self-clear.

**API Priority and Fairness (APF)** replaced the old fixed --max-requests-inflight limit. Requests are bucketed into PriorityLevelConfigurations via FlowSchema rules matching on user/group/verb/resource, each with its own concurrency share and queue — this stops one noisy client (a buggy controller listing everything in a hot loop, a CI pipeline hammering the API) from starving critical traffic like kubelet heartbeats. A 429 Too Many Requests from the API server usually means APF rejected the request due to queue exhaustion at that client's priority level — not general server overload. Important nuance: kubectl running as your normal kubeadm admin identity is typically in the system:masters group, which the default APF config binds to the exempt priority level — meaning your own admin kubectl commands are usually not throttled by APF at all. Real APF incidents are almost always a specific service account or automated client, not admin traffic.

**Static pod failure modes split into two completely different diagnostic paths** (this connects directly to your existing static-pod-corruption chaos lab, one level deeper): if a manifest in /etc/kubernetes/manifests/ is syntactically invalid YAML, kubelet can't even parse it into a Pod object — it logs a parse error to its own journal and never attempts to run anything. crictl ps -a will show nothing at all for that component. If the manifest is valid YAML but semantically wrong (bad flag, wrong path), kubelet successfully creates the Pod object and hands it to the container runtime, which genuinely tries and fails — crictl ps -a will show a container, crash-looping, and crictl logs will have the real error. Same top-level symptom ("this component isn't running"), completely different place to look, and you can tell which case you're in within seconds just by checking whether crictl ps -a shows anything.

**The recursive problem:** if kube-apiserver itself is down (crashed static pod, expired/corrupted serving cert), kubectl cannot help you diagnose it — the tool you'd reach for is unavailable precisely because of the thing you're trying to diagnose. This is the single most senior-differentiating scenario in this whole module: you have to already know the SSH-in, crictl/journalctl-direct path cold, because you cannot discover it live while the system that would normally teach you is the thing that's broken.

**Leader election** for controller-manager and scheduler uses Lease objects (coordination.k8s.io, not the older ConfigMap-annotation mechanism) — kubectl -n kube-system get lease shows holderIdentity for each. On a true multi-control-plane cluster, losing the active holder triggers another instance to acquire the lease within seconds. On your single-control-plane cluster, there is no standby to fail over to — losing controller-manager doesn't cause errors anywhere, it just means nothing gets reconciled (no new ReplicaSets from Deployments, no new Pods from ReplicaSets, no anything) until it comes back, silently.

**The Alternative Landscape**

<img width="872" height="527" alt="image" src="https://github.com/user-attachments/assets/619b8bba-6564-4a0a-8fe4-6112da5b5b08" />

<img width="842" height="287" alt="image" src="https://github.com/user-attachments/assets/3e757049-9c52-4782-9a65-d0454acb7790" />

You're running stacked etcd here specifically because it's the simplest path to a working learning cluster; your separate external-etcd cluster exists specifically to practice the production-grade pattern once blast-radius isolation actually matters. Companies like Booking or Adyen running self-managed clusters would default to external etcd for exactly the isolation reason in the table; most companies today just default to a managed control plane entirely and only manage etcd/apiserver themselves where compliance or cost genuinely demands it.

**Interview POV & Edge Cases**

The signature senior-level prompt here: "kubectl is completely unresponsive — walk me through diagnosing a control plane outage with your normal tooling unavailable." They're checking whether you reflexively reach for SSH + crictl/journalctl rather than freezing because "the tool I'd normally use is the thing that's broken." Expected order: SSH to the control-plane node → check manifest files are valid YAML → check kubelet's own journal → crictl ps -a for the relevant static pod containers → crictl logs on anything crash-looping → check cert validity directly with openssl x509 -in ... -noout -dates if TLS is suspected.

**Gotchas**:

- Reading a 429 as "the cluster needs more capacity" instead of checking apiserver_flowcontrol_rejected_requests_total and the actual FlowSchema involved — scaling nodes fixes nothing here, since it's a fairness/queueing problem, not a resource problem.
- Treating "too old resource version" as data corruption and trying to "fix" etcd, when it's just an expected MVCC boundary the client resolves itself via re-list.
- Forgetting the explicit alarm disarm step after clearing a NOSPACE condition — space reclaimed, writes still blocked, because the alarm doesn't self-clear.
- Assuming system:masters/admin kubectl traffic is subject to the same throttling as everything else — it's typically exempt, which is exactly why real APF incidents trace back to a service account or automated client, not a human running kubectl too fast.
- Assuming controller-manager/scheduler restarts are always harmless because "that's what leader election is for" — true in real HA, false and directly disruptive on any single-control-plane cluster, including this one.

**The 'Better Way' (Evolution)**

Managed control planes (EKS/GKE/AKS) exist specifically to remove this entire failure category from a platform team's plate — multi-AZ etcd, automatic apiserver failover, no one SSHing into a control-plane node ever. That's a genuine trade-off worth naming honestly: you give up deep operational control (and the exact debugging reflexes this module builds) in exchange for eliminating the failure category outright. It's also why this material still matters even if you're targeting EKS-heavy shops — APF throttling, admission webhook failures, and watch-resync behavior all still surface at the API-server level regardless of who's operating etcd underneath, and interviewers know the difference between someone who's only ever seen a managed control plane and someone who's actually broken one on purpose.

**Module 6 Lab Pack**

<img width="887" height="463" alt="image" src="https://github.com/user-attachments/assets/08a3acf7-2c3e-4fd3-aff9-32b8fd4ff697" />

**Before starting: sudo cp -r /etc/kubernetes/pki /etc/kubernetes/pki.bak and sudo cp -r /etc/kubernetes/manifests /tmp/manifests.bak** — every lab in this module touches control-plane internals directly; always have a known-good copy one command away.

**Lab 1 — Corrupted API Server TLS Certificate**

**Break**

```
bash
# ON THE MASTER NODE — you already backed up pki above
sudo truncate -s 0 /etc/kubernetes/pki/apiserver.crt
```

**Predict**: Will any kubectl command still work after this, from anywhere?

**Diagnose** — from wherever your kubeconfig normally works:

```
bash
kubectl get nodes
```

Expect total failure — a TLS handshake error, not a normal Kubernetes error message. Now the recursive part — you have to go around kubectl entirely:

```
bash
# back on the master, over SSH
sudo crictl ps -a | grep apiserver
sudo crictl logs <apiserver-container-id>
```

Expect the container crash-looping, and the log showing a certificate-loading failure.

**Fix**

```
bash
sudo cp /etc/kubernetes/pki.bak/apiserver.crt /etc/kubernetes/pki/apiserver.crt
sudo crictl ps -a | grep apiserver -w
kubectl get nodes
```

**Debrief**: This is the lab your chaos-lab notes scoped but never ran — worth having actually done it now. Note what real-world cert expiry (rather than corruption) looks like preventively: sudo kubeadm certs check-expiration lists every cert's remaining validity, and sudo kubeadm certs renew all regenerates them (static pods need to restart afterward to pick up the new files — moving the manifest out and back, or a kubelet restart, forces that). The failure signature at TLS-handshake level is identical whether the cert is corrupted or genuinely expired — which is exactly why "everything just stopped working, no config changed" is the classic tell of an expired cert, not a Kubernetes bug.

**Lab 2 — Static Pod: Syntax Error vs. Semantic Error**

**Break — Part A (syntax error, guaranteed invalid YAML)**

```
bash
sudo cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
sudo sed -i '1i @@@INVALID_YAML@@@' /etc/kubernetes/manifests/etcd.yaml
```

**Predict**: Will crictl ps -a show an etcd container attempting (and failing) to start, or nothing at all?

**Diagnose**

```
bash
sudo journalctl -u kubelet --no-pager | tail -30
sudo crictl ps -a | grep etcd
```

Expect a parse error in kubelet's own journal, and nothing in crictl ps -a — kubelet never got far enough to attempt a container. Since etcd is now down, watch the API server start failing shortly after too (cascading dependency).

**Fix**

```
bash
sudo cp /tmp/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml
sudo crictl ps -a | grep etcd -w
kubectl get nodes
```

**Break — Part B (semantically wrong, valid YAML)**

```
bash
sudo cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak2
sudo sed -i 's#--data-dir=/var/lib/etcd#--data-dir=/nonexistent/path#' /etc/kubernetes/manifests/etcd.yaml
```

**Predict**: Same absence in crictl ps -a as Part A, or different?

**Diagnose**

```
bash
sudo crictl ps -a | grep etcd
sudo crictl logs <etcd-container-id>
```

This time crictl ps -a does show a container — crash-looping — with a real startup error about the bad path in its logs.

**Fix**

```
bash
sudo cp /tmp/etcd.yaml.bak2 /etc/kubernetes/manifests/etcd.yaml
```

**Debrief:** This pair is the whole lesson from the Deep-Dive made concrete: identical top-level symptom ("etcd isn't running"), and crictl ps -a alone tells you instantly which diagnostic path you're on — empty means go to the kubelet journal, populated-and-crash-looping means go to crictl logs. Checking crictl ps -a first, before anything else, is the single fastest triage step for any static pod failure.

**Lab 3 — etcd NOSPACE Alarm**

Work as root for this lab and set up etcdctl once:

```
bash
sudo -i
export ETCDCTL_API=3
export ETCDCTL_ARGS="--endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key"
etcdctl $ETCDCTL_ARGS endpoint status --write-out=table
```

**Break** — lower the quota so you don't have to wait for 2GB of real growth:

```
bash
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak3
sed -i '/- etcd/a\    - --quota-backend-bytes=16777216' /etc/kubernetes/manifests/etcd.yaml
# wait for etcd to restart with the new flag, then confirm:
crictl ps | grep etcd
```

Now flood it (exit root shell only for the kubectl part, or run kubectl with --kubeconfig=/etc/kubernetes/admin.conf while still root):

```
bash
for i in $(seq 1 300); do
  kubectl --kubeconfig=/etc/kubernetes/admin.conf create configmap flood-$i \
    --from-literal=data="$(head -c 50000 /dev/urandom | base64)" -n default >/dev/null 2>&1
done
```

**Predict**: Once the quota trips, will reads still work? Writes to totally unrelated objects?

**Diagnose**

```
bash
etcdctl $ETCDCTL_ARGS alarm list
kubectl --kubeconfig=/etc/kubernetes/admin.conf get configmap flood-1   # reads
kubectl --kubeconfig=/etc/kubernetes/admin.conf create configmap unrelated-test --from-literal=x=1   # writes
```

Expect alarm list showing NOSPACE; reads succeed; every write cluster-wide fails, including this totally unrelated ConfigMap, with an mvcc: database space exceeded error.

**Fix**

```
bash
for i in $(seq 1 300); do
  kubectl --kubeconfig=/etc/kubernetes/admin.conf delete configmap flood-$i -n default >/dev/null 2>&1
done
REV=$(etcdctl $ETCDCTL_ARGS endpoint status --write-out=json | grep -o '"revision":[0-9]*' | head -1 | cut -d: -f2)
etcdctl $ETCDCTL_ARGS compact $REV
etcdctl $ETCDCTL_ARGS defrag
etcdctl $ETCDCTL_ARGS alarm disarm
# restore the real quota
cp /tmp/etcd.yaml.bak3 /etc/kubernetes/manifests/etcd.yaml
exit   # leave root shell
```

**Debrief**: Notice writes stayed blocked even right after compact — because compact only marks old revisions reclaimable, it doesn't shrink the file, and more importantly the alarm itself doesn't clear on its own even once space genuinely is reclaimed. disarm is a separate, easy-to-forget step, and skipping it is a real "why is my cluster still read-only, I already cleaned everything up" incident.

**Lab 4 — Forcing "too old resource version" Deterministically**

```
bash
sudo -i
export ETCDCTL_API=3
export ETCDCTL_ARGS="--endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key"
CURRENT_REV=$(etcdctl $ETCDCTL_ARGS endpoint status --write-out=json | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Status']['header']['revision'])")
echo "Watching from revision: $CURRENT_REV"
```

**Predict**: If you compact etcd past this revision and then ask it to resume a watch from $CURRENT_REV, what happens?

**Break**

```
bash
etcdctl $ETCDCTL_ARGS put demo-key demo-value
NEW_REV=$(etcdctl $ETCDCTL_ARGS endpoint status --write-out=json | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Status']['header']['revision'])")
etcdctl $ETCDCTL_ARGS compact $NEW_REV
```

**Diagnose**

```
bash
etcdctl $ETCDCTL_ARGS watch --rev=$CURRENT_REV demo-key
```

Expect an immediate error: mvcc: required revision has been compacted.

**Fix:** nothing to fix — clean up and relate it back to the K8s layer:

```
bash
etcdctl $ETCDCTL_ARGS del demo-key
exit
bash
kubectl get pods --watch   # this is the exact client-side mechanism — if it ever falls this far behind, it just re-lists automatically
```

**Debrief:** You just reproduced, deterministically and at the source, the exact condition behind "too old resource version" from a stalled kubectl get --watch or a controller's informer resync. It's not corruption, not data loss — it's MVCC's compaction boundary doing exactly what it's designed to do. The correct reaction to seeing this in the wild is "the watcher will re-list, this is expected," never "something is wrong with etcd."

**Lab 5 — Killing controller-manager on a Single Control Plane**

```
bash
kubectl -n kube-system get lease
kubectl -n kube-system get lease kube-controller-manager -o yaml | grep holderIdentity
```

**Predict:** With only one control-plane node, what happens to a brand-new Deployment if controller-manager isn't running?

**Break**

```
bash
sudo crictl ps | grep kube-controller-manager
sudo crictl stop <container-id>
```

**Diagnose**

```
bash
kubectl create deployment leader-test --image=nginx --replicas=3
kubectl get deployment leader-test
kubectl get replicaset -l app=leader-test
kubectl get pods -l app=leader-test
```

The Deployment object itself is created instantly (that's just an API server write). No ReplicaSet ever appears — that reconciliation is specifically controller-manager's job, and it isn't running.

**Fix**

```
bash
sudo crictl ps -a | grep kube-controller-manager   # kubelet should already be restarting the static pod
kubectl get replicaset -l app=leader-test -w   # watch it appear the instant controller-manager is back
```

**Debrief:** Deployment → ReplicaSet → Pod normally happens so fast you never see the gap between the steps — this lab makes it visible by holding one step open. In a true multi-control-plane cluster, holderIdentity on that Lease would flip to a standby within seconds and you'd likely never notice. Here, on this exact single-master setup, everything downstream of controller-manager simply stops — silently, with zero error surfaced anywhere — until it restarts. This is precisely the gap your external-etcd, presumably-eventually-multi-control-plane cluster exists to close.

**Lab 6 — APF Throttling via a Constrained ServiceAccount**

```
bash
kubectl get flowschemas
kubectl get prioritylevelconfigurations
```

**Break** — create a low-privileged identity (your admin kubeconfig is almost certainly exempt from APF, so testing as yourself would show nothing):

```
bash
kubectl create serviceaccount apf-test
kubectl create clusterrolebinding apf-test-view --clusterrole=view --serviceaccount=default:apf-test
TOKEN=$(kubectl create token apf-test --duration=1h)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
```

**Predict**: Firing a large burst of concurrent raw requests as this constrained identity — what response code do you expect once its priority level's concurrency share is exhausted?

```
bash
for i in $(seq 1 100); do
  curl -sk -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $TOKEN" \
    "$APISERVER/api/v1/namespaces/default/pods" &
done
wait | sort | uniq -c
```

(Using raw curl in parallel deliberately bypasses kubectl's own client-side rate limiting, which would otherwise smooth this out.)

**Diagnose**

```
bash
kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
```

Expect a mix of 200s and 429s in the burst, and a non-zero, incrementing rejection counter — direct confirmation this is APF fairness throttling, not general server overload.

**Fix / cleanup**

```
bash
kubectl delete clusterrolebinding apf-test-view
kubectl delete serviceaccount apf-test
```

**Debrief**: This is the accurate version of a scenario people often get wrong in interviews — your own admin kubectl traffic is typically exempt from APF entirely via the system:masters group binding, so a real 429 incident is almost never "the cluster is overloaded, add nodes." It's a specific service account, operator, or CI client hammering the API within its own priority level's queue. The fix lives in FlowSchema/PriorityLevelConfiguration tuning or fixing the offending client's request pattern — not capacity.

**Module 7 — Node-Level Failures (kubelet Internals & OS/cgroup Pressure)**

Everything so far has been Kubernetes-object-level troubleshooting. This module goes one floor further down — to the actual Linux node underneath. This is also where your cgroup driver mismatch chaos lab lives conceptually; we'll go deeper into why that failure class exists at all.

**The 'Why' (The Problem)**

Early container orchestration mostly assumed nodes were healthy, roughly-infinite resource pools — a pod just "stopped," with no systematic signal distinguishing "this pod's manifest is wrong" from "the node it's on is dying." That's a dangerous gap: a single leaking pod could exhaust a node's memory or disk and silently take every other pod on that node down with it, with nothing in any pod's own describe output hinting at the real cause. Kubernetes needed two things: a systematic way for kubelet to observe and report actual OS-level health (Node Conditions), and a systematic way to protect a node from any one workload's misbehavior before it becomes a total node failure (the eviction manager, driven by QoS classes). Node-level failures are exactly the failures that don't show up when you describe a pod — you have to know to look at the node instead.

**Deep-Dive Mechanics**

**Node Conditions and the two-tier heartbeat.** kubelet continuously polls OS-level metrics (via its embedded cAdvisor) and reports them as Node Conditions: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable. There are actually two separate heartbeat mechanisms: a full NodeStatus update (relatively infrequent — expensive, since it's a full object write to etcd) and a lightweight Lease object heartbeat (frequent, cheap, roughly every 10s) that the node-lifecycle-controller actually watches to decide node health quickly. This split exists purely for cost — writing a full node status object to etcd every few seconds across a large fleet would be wasteful; the cheap Lease heartbeat gives fast failure detection without that cost.

**QoS class is computed, not declared** — you never set it directly, kubelet derives it from your resource spec:

- Guaranteed: every container has requests == limits for both CPU and memory.
- Burstable: has requests, but they don't match limits (or only some containers/resources specify them).
- BestEffort: no requests or limits at all, anywhere.

This classification directly sets each pod's oom_score_adj and eviction priority — BestEffort is evicted/OOM-killed first, Guaranteed is protected most, Burstable scales in between based on how far actual usage exceeds its requested amount.

**The eviction manager is a kubelet decision, distinct from scheduler preemption (Module 3)**. Preemption happens before placement, on the scheduler, deciding who gets a node. Eviction happens on an already-running pod, decided locally by the kubelet on the node that's under pressure, once a Node Condition trips a configured threshold (soft thresholds have a grace period; hard thresholds evict immediately). kubelet ranks victims by: is usage exceeding requests, then QoS class, then priority — not by which pod actually caused the pressure. This is the source of a very common, very confusing incident: an innocent BestEffort pod gets evicted while the real memory hog (a Burstable pod using far more than it requested) survives, simply because QoS class outranks "who's responsible" in the eviction algorithm.

**cgroup driver mismatch, one level deeper than your chaos lab.** The kernel's cgroup subsystem needs exactly one manager. If kubelet is configured for cgroupfs while the container runtime (containerd) or the OS init system (systemd) expects systemd to own cgroups, you get two systems independently trying to manage the same resource hierarchy — inconsistent accounting, resource limits silently not applying correctly, and on cgroup v2-only kernels (the current default on modern Ubuntu), systemd as the driver is effectively required, not optional. This is exactly the mismatch your existing chaos lab reproduced; the underlying reason it matters is that kubelet and the runtime have to agree on who is authoritative for the exact same kernel data structures.

**Automatic taint-based node eviction — the real mechanism behind Module 3's manual NoExecute lab**. When a node misses its Lease heartbeats past node-monitor-grace-period (default ~40s), the node-lifecycle-controller automatically applies node.kubernetes.io/not-ready:NoExecute (or ...unreachable:NoExecute) — the exact same taint type you applied by hand in Module 3. Pods don't get evicted instantly even then: the built-in default tolerationSeconds for these specific taints is 300 seconds (5 minutes), unless a pod explicitly overrides it. This is the real answer to "why does failover take 5 minutes on a perfectly healthy multi-node cluster" — it's not a bug or slow detection, it's this specific, deliberate default giving a flapping node a grace window to recover before Kubernetes starts evicting everything on it.

**PLEG health is itself a node-readiness signal,** not just a pod-lifecycle detail (Module 2 mentioned this briefly — here's the full picture). If kubelet's PLEG relist loop can't get timely responses from the container runtime, kubelet marks itself unhealthy internally and the node goes NotReady — even though the kubelet process is fully alive. ps aux | grep kubelet showing a running process tells you nothing about whether kubelet can actually do its job.

**Kernel-level failure surfaces beyond cgroups,** each with its own non-obvious signature:

- Inode exhaustion is distinct from disk space exhaustion — a filesystem can show plenty of free bytes (df -h) while being 100% out of inodes (df -i) if it's littered with huge numbers of tiny files. DiskPressure actually monitors both independently.
- Clock skew breaks TLS certificate validity windows and Raft timing assumptions — and the resulting failure looks exactly like a PKI problem (Module 6), not a clock problem, unless you specifically check the system clock.
- conntrack table exhaustion (nf_conntrack_max) causes the kernel to silently drop packets once the connection-tracking table fills — no Kubernetes object anywhere reflects this; it only shows up in dmesg.

**The Alternative Landscape**

<img width="901" height="581" alt="image" src="https://github.com/user-attachments/assets/ac72f852-76af-460b-ad9c-ff2e9ab2bf8f" />

None of these replace kubelet's own eviction manager — it's the always-on last line of defense. You'd add a descheduler when workloads correctly start well-placed but drift into imbalance over time; VPA when the actual root cause is chronically wrong resource requests rather than genuine unpredictable spikes; and NPD when you need to catch hardware-level node rot before it manifests as an actual Kubernetes-visible symptom at all — this is standard on GKE and increasingly common as a self-managed DaemonSet elsewhere.

**Interview POV & Edge Cases**

Classic prompt: "A node goes NotReady intermittently under load — walk me through it." The layered answer: check Node Conditions first (kubectl describe node) — which one tripped; check kubelet's own journal for PLEG is not healthy or similar internal loop errors, not just "is the process running"; check actual OS-level pressure directly (free -h, and critically df -i alongside df -h, and conntrack -C against nf_conntrack_max); and factor in the ~40s detection grace period plus the 300s default toleration before assuming something's abnormally slow.

**Gotchas**:

- df -h showing free space while the node is genuinely under DiskPressure from inode exhaustion — checking only one of the two is a real, common miss.
- Assuming the pod that got evicted during a MemoryPressure event is the pod that caused it — eviction order follows QoS class and usage-over-request ratio, not culpability; the actual offender (often Burstable, using more than requested but less than its generous limit) can survive while an innocent BestEffort pod nearby gets killed.
- Treating the 300-second default failover delay as a symptom of something broken, when it's a deliberate built-in tolerationSeconds default protecting against flapping nodes.
- "kubelet is running" ≠ "node is healthy" — PLEG failures or a stuck container runtime can leave the process alive while kubelet is functionally unable to do its job.
- Chasing a confusing TLS/certificate error for several minutes (Module 6 territory) when the actual root cause is a clock five minutes off, discoverable in ten seconds with timedatectl status.

**The 'Better Way' (Evolution)**

Autoscaler-integrated node health checks (Karpenter and similar) increasingly terminate and replace visibly unhealthy nodes proactively rather than waiting for kubelet's own slow-to-surface conditions to fully develop. Node Problem Detector plus an automatic remedy system (GKE's automatic node repair is the canonical example) watches kernel/journald signals directly and drains+replaces before a Kubernetes-level condition even trips. VPA addresses the underlying resource-misestimation that causes most pressure events in the first place, rather than reacting after the fact. And at the observability layer, node-level eBPF tooling (Cilium/Pixie again — this keeps coming up because it genuinely is the modern answer to "invisible to kubectl") gives direct visibility into kernel-level exhaustion — conntrack fill, socket exhaustion — that no kubectl command surfaces at all.

**Module 7 Lab Pack**

<img width="873" height="430" alt="image" src="https://github.com/user-attachments/assets/474530aa-97c8-4324-aa17-0b3e7924a2d2" />

**Lab 1 — Inode Exhaustion (DiskPressure Without Low Disk Space)**

**Break (SSH to a worker)**

```
df -h /   # note free space — should look fine throughout this lab
df -i /   # note free inodes
sudo mkdir -p /tmp/inode-flood
i=0
while true; do
  i=$((i+1))
  sudo touch /tmp/inode-flood/f$i
  if [ $((i % 10000)) -eq 0 ]; then
    USE=$(df -i / | awk 'NR==2{print $5}' | tr -d '%')
    echo "created $i files, inode use: $USE%"
    [ "$USE" -ge 90 ] && break
  fi
done
```

**Predict**: Will df -h and df -i on this node tell the same story?

**Diagnose** (from master)

```
bash
kubectl describe node <that-worker> | grep -A5 Conditions
```

**Expect DiskPressure**: True even though df -h on the node itself still shows plenty of free bytes — kubelet monitors nodefs.inodesFree as an entirely separate signal from nodefs.available.

**Fix**

```
bash
# on the worker
sudo rm -rf /tmp/inode-flood
df -i /
bash
kubectl describe node <that-worker> | grep -A5 Conditions   # confirm DiskPressure clears
```

**Debrief**: Checking df -h alone and concluding "disk isn't the problem" is a genuinely common near-miss — this scenario (huge counts of tiny files: logs, temp lease files, small cache entries) is realistic, not contrived, and the fix requires checking -i specifically, which most people don't reach for by habit.

**Lab 2 — MemoryPressure + QoS-Based Eviction Ordering**

First check the target node's real capacity so you size the "hog" correctly:

```
bash
kubectl describe node <worker-1> | grep -A3 Allocatable
kubectl label node <worker-1> pressure-test=true
```

**Break** — three pods, three QoS classes, one node. Adjust vm-bytes below to roughly 70-80% of that node's allocatable memory:

```
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  nodeSelector: {pressure-test: "true"}
  containers:
  - name: g
    image: nginx
    resources:
      requests: {cpu: "100m", memory: "200Mi"}
      limits: {cpu: "100m", memory: "200Mi"}
---
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  nodeSelector: {pressure-test: "true"}
  containers:
  - name: b
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: burstable-hog
spec:
  nodeSelector: {pressure-test: "true"}
  containers:
  - name: hog
    image: polinux/stress
    resources:
      requests: {memory: "100Mi"}
      limits: {memory: "1500Mi"}
    args: ["stress", "--vm", "1", "--vm-bytes", "1200M", "--vm-hang", "0"]
EOF
```

**Predict**: When MemoryPressure trips, which pod gets evicted first — the hog that's actually causing it, the innocent bystander, or neither?

**Diagnose**

```
bash
kubectl describe node worker-1 | grep -A5 Conditions
kubectl get pods -o wide -w
```

Expect besteffort-pod evicted first (Status: Evicted), while burstable-hog — the actual cause — and guaranteed-pod both survive, at least initially.

**Fix**

```
kubectl delete pod burstable-hog guaranteed-pod besteffort-pod
kubectl label node worker-1 pressure-test-
```

**Debrief**: This is exactly the "who gets blamed vs. who's actually guilty" gap from the Deep-Dive, made visible. In a real incident, seeing an evicted BestEffort pod and assuming it was the problem — rather than checking which pod's actual usage was driving the pressure — sends you investigating the wrong workload.

**Lab 3 — PID Exhaustion**

```
bash
kubectl label node worker-1 pressure-test=true
cat /proc/sys/kernel/pid_max   # on worker-1 — total system PID space
```

**Break**

```
bash
kubectl run pid-hog --image=busybox --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"pressure-test":"true"},"containers":[{"name":"pid-hog","image":"busybox","command":["sh","-c","for i in $(seq 1 20000); do sleep 1000 & done; wait"]}]}}'
```

**Predict:** Will this hit a per-pod limit first, or exhaust the node's entire PID space?

**Diagnose**

```
kubectl describe node worker-1 | grep -A5 Conditions
kubectl describe pod pid-hog | tail -20
```

One of two outcomes, both instructive: if a per-pod PID limit (PodPidsLimit/--pod-max-pids) is configured, pid-hog alone fails once it hits its own cgroup's pids.max — the node stays fine, exactly as designed. If no such limit is configured (common in a vanilla kubeadm install), you may see PIDPressure: True on the node itself.

**Fix**

```
kubectl delete pod pid-hog
```

**Debrief**: PID space is finite in a way most engineers don't intuitively track the way they track memory or disk — and unlike those, exhausting it affects every process on the node, including kubelet and the container runtime itself. Whether this lab stayed contained to one pod or affected the whole node tells you directly whether your cluster has per-pod PID limiting configured — worth checking explicitly rather than assuming.

**Lab 4 — kubelet Stopped → Automatic NoExecute Taint → 300s Default Failover**

```
bash
kubectl label node worker-1 pressure-test-   # clean up from earlier labs
kubectl create deployment failover-test --image=nginx --replicas=1
kubectl get pod -l app=failover-test -o wide   # note the node
```

**Break (SSH to that node)**

```
sudo systemctl stop kubelet
```

**Predict**: Time how long each of these takes, from the moment kubelet stops — node NotReady, the automatic taint appearing, and the pod actually being rescheduled.

**Diagnose** (from master, with timestamps)

```
bash
date; kubectl get nodes -w   # note the moment it flips to NotReady
kubectl describe node <that-node> | grep -A3 Taints   # watch for node.kubernetes.io/not-ready:NoExecute
kubectl get pod -l app=failover-test -o wide -w   # watch for eviction + reschedule onto the other worker
```

Expect roughly: NotReady ~40s after the last heartbeat, the NoExecute taint applied automatically at essentially the same moment, but the pod itself doesn't actually move until ~300s later.

**Fix**

```
# on the original node
sudo systemctl start kubelet
```

**Debrief**: This connects Module 3's manual NoExecute taint lab and Module 6's Lease-based leader election lab into one real, automatic incident: the node-lifecycle-controller is watching the exact same Lease heartbeats controller-manager's own election relies on, applying the exact same taint type you applied by hand earlier, gated by the exact same 300-second default toleration. "Why did failover take 5 minutes" stops being mysterious once you've watched all three pieces fire in sequence with your own timestamps.

**Lab 5 — Clock Skew Masquerading as a PKI Problem**

**Break** (SSH to a worker — restore promptly, this affects everything else running on the node while active)

```
timedatectl status   # note current sync state before you start
sudo timedatectl set-ntp false
sudo date -s "2027-01-01 00:00:00"
```

**Predict**: Will the resulting error, wherever it surfaces, mention "time" or "clock" anywhere — or will it look like an entirely different category of problem?

**Diagnose**

```
kubectl get nodes   # from master — this node may show NotReady or a communication error
sudo journalctl -u kubelet --no-pager | tail -30   # on the worker
```

Expect kubelet's client certificate to fail validation against the API server — the error text reads like a TLS/certificate problem (echoing Module 6 Lab 1), with nothing pointing at the clock directly.

**Fix**

```
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
timedatectl status   # confirm "System clock synchronized: yes"

kubectl get nodes -w
```

**Debrief**: This is deliberately one of the most disorienting failures in the whole masterclass — the symptom actively misdirects you toward PKI troubleshooting when the real fix is one date command away. timedatectl status is worth being a genuinely early, cheap check whenever you see unexplained cert/auth failures with no corresponding config change anywhere.

**Lab 6 — conntrack Table Exhaustion (Kernel-Level, Invisible to kubectl)**

```
kubectl label node worker-1 pressure-test=true
# on worker-1
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

**Break** — lower the ceiling so it's easy to exhaust without generating enormous real traffic (note the original value above so you can restore it):

```
sudo sysctl -w net.netfilter.nf_conntrack_max=256

kubectl run conntrack-flood --image=nicolaka/netshoot --restart=Never \
  --overrides='{"spec":{"nodeSelector":{"pressure-test":"true"}}}' -- sleep 3600
kubectl exec conntrack-flood -- sh -c 'for i in $(seq 1 500); do (curl -s -o /dev/null -m1 http://1.1.1.1 &); done; sleep 5'
```

**Predict**: Once the table fills, do new connections get a clear error, or something else?

**Diagnose**

```
bash
# on worker-1
dmesg | grep -i conntrack   # look for "nf_conntrack: table full, dropping packet"
kubectl exec conntrack-flood -- curl -sm2 -o /dev/null -w "%{http_code} %{time_total}\n" http://1.1.1.1
```

Expect a silent hang/timeout with no informative error, and the real cause visible only in dmesg — nowhere in any Kubernetes object.

**Fix**

```
sudo sysctl -w net.netfilter.nf_conntrack_max=262144   # or your originally noted value
kubectl delete pod conntrack-flood
kubectl label node worker-1 pressure-test-
```

**Debrief**: This is the deepest-in-the-stack failure in the entire masterclass so far — completely invisible to kubectl, to CoreDNS logs, to kube-proxy, to every Node Condition. At genuinely high pod density or connection churn (a busy proxy, a high-QPS service), nf_conntrack_max is a real, recurring node-level sysctl tuning knob that has nothing to do with any Kubernetes object at all — the kind of thing that only shows up once you already know to check kernel logs on the node itself.

**Module 8 — Ingress/Gateway API & App-Layer Failures**

This is where every earlier module's failures ultimately surface — a scheduling problem, a broken readiness probe, a stuck CSI volume, a resource-starved node all eventually show up here as the same-looking generic HTTP error at the edge. This module is as much about disciplined triage direction as new mechanics. And since you've already got Gateway API + Envoy Gateway + cert-manager running for real on this cluster, we'll teach Gateway API primary throughout, with Ingress alongside it for contrast — not the other way around.

**The 'Why' (The Problem)**

Before Ingress existed, exposing an HTTP service externally meant Service type: LoadBalancer — and every single service needing external access provisioned its own cloud load balancer. No shared L7 routing, no path-based fan-out, no centralized TLS termination — just cost and complexity multiplying linearly with service count. Ingress fixed that: one object type describing host/path routing rules, implemented by a shared controller fronting many backend Services behind a single IP and a single place to manage certificates.

But Ingress's core spec only really standardizes basic host+path matching. Anything beyond that — weighted traffic splitting, header-based routing, cross-namespace routing restrictions — required proprietary, controller-specific annotations: nginx.ingress.kubernetes.io/... means nothing to the AWS Load Balancer Controller, which has its own entirely different annotation vocabulary. Ingress manifests that look "standard" are quietly non-portable, and complex routing logic devolves into annotation soup nothing type-checks. Gateway API was built specifically to fix this: a properly structured, typed, role-oriented API — GatewayClass / Gateway / HTTPRoute (and GRPCRoute, TCPRoute, etc.) — where the things Ingress could only express via magic strings are now real, validated fields. It also explicitly models something Ingress never did: who owns which layer of the config, matching how platform teams and app teams actually divide responsibility.

**Deep-Dive Mechanics**

**The object chain, and the single most common early mistake.** Neither Ingress nor Gateway API objects do anything by themselves — they're pure declarative intent, sitting inert in etcd until a controller watches and implements them. Ingress: Ingress → matched by IngressClass → an actual Ingress Controller (not part of core Kubernetes — ingress-nginx, ALB controller, etc., installed separately) → backend Service → pod endpoints. Gateway API: HTTPRoute → parentRef → Gateway → GatewayClass → the controller that implements that class (Envoy Gateway, in your case). Creating either object with no matching class/controller installed is accepted silently by the API server — no error, no rejection, it just sits there permanently doing nothing. This is the Gateway-API-equivalent of Module 1's "no controller watching" gap, and it's the very first thing to rule out whenever routing "isn't working" with no obvious error anywhere.

**Gateway API's three-tier role separation is the actual architectural point, not just extra objects:**

- GatewayClass — cluster-scoped, infra/platform-team owned, declares which controller implements it (Envoy Gateway).
- Gateway — a specific listener instance: ports, protocols (HTTP/HTTPS/TLS/TCP), hostnames, TLS cert references. Typically one or a few per cluster or team, platform-owned.
- HTTPRoute — app-team owned, lives in the app's own namespace, references a Gateway via parentRefs, and defines the actual matching rules and backendRefs — including native weighted traffic splitting, no annotations required.

This maps directly onto real org RBAC boundaries in a way Ingress structurally couldn't: with plain Ingress, anyone with create-permission on Ingress objects in a namespace could effectively influence a shared load balancer's routing. Gateway API lets a platform team own the Gateway and grant app teams only HTTPRoute permissions in their own namespaces — routing intent stays properly scoped.

**ReferenceGrant — an explicit, deliberate cross-namespace security control.** If an HTTPRoute in namespace A wants to reference a backendRef Service in namespace B, or a Gateway wants to reference a TLS Secret in a different namespace, that reference is rejected by default unless a ReferenceGrant object in the target namespace explicitly permits it. No implicit cross-namespace trust — a real security improvement over some of the looser cross-namespace tricks that existed in the Ingress-annotation world. This is also a very real "why does everything look configured correctly but traffic still 404s" gotcha.

**Status conditions are the actual diagnostic surface — and there are two of them that get conflated.** Both HTTPRoute and Gateway report controller-written status back onto the object itself: Accepted (is the route syntactically valid and successfully attached to its parent Gateway) and, separately, ResolvedRefs (do the objects it actually points to — backend Services, TLS Secrets — exist and is access to them permitted). A route can be Accepted: True and still be completely non-functional because ResolvedRefs: False. Checking only the first condition and seeing "True" is a genuinely common false-confidence trap — always read the reason field on ResolvedRefs specifically (BackendNotFound vs RefNotPermitted are different problems with different fixes, both showing up as the same False status).

**cert-manager's ACME flow has a real chicken-and-egg failure mode.** cert-manager watches for TLS configuration (a Certificate resource, or automatically via a Gateway listener's TLS config) and requests a cert from an Issuer/ClusterIssuer — commonly Let's Encrypt via ACME. HTTP-01 challenge validation requires the ACME server to reach a specific path, from the public internet, through the exact same routing infrastructure you're simultaneously trying to configure. If your Gateway/HTTPRoute setup is broken, the HTTP-01 challenge can't complete, so the cert never issues, so TLS never comes up — and the resulting symptom (no HTTPS) looks like a cert problem when the actual root cause is the underlying routing, unresolved. (DNS-01 sidesteps this — it validates via a DNS TXT record instead of inbound HTTP traffic, at the cost of needing DNS provider API credentials configured in cert-manager, but it does support wildcard certs, which HTTP-01 cannot.)

**The 502/503/504 triad each points to a genuinely different layer — collapsing them wastes real triage time:**

- 502 Bad Gateway — the proxy successfully reached the backend, but the connection failed or reset (wrong port configured, nothing listening, pod crashed mid-request).
- 503 Service Unavailable — the proxy has zero healthy endpoints to send traffic to at all (ties directly back to Module 1/4's readiness-gating: a Service with no Ready backends).
- 504 Gateway Timeout — the backend was reachable and accepted the connection, but didn't respond within the proxy's configured timeout (a slow app, or resource pressure from Module 6/7 upstream).

Because this is the layer where everything eventually surfaces, the correct triage instinct is always to work backward — check the gateway/ingress controller's own access/error logs first (Envoy Gateway's pod logs, in your setup) to see exactly which of the three it logged, which immediately narrows which earlier module's failure category you're actually chasing.

**The Alternative Landscape**

<img width="863" height="516" alt="image" src="https://github.com/user-attachments/assets/2e217fa5-a81d-423e-98b0-85e81d113b73" />

<img width="872" height="211" alt="image" src="https://github.com/user-attachments/assets/2dd4aa56-7384-45b2-958b-8dc147b072a7" />

You'd choose a Gateway-API-native controller like Envoy Gateway (what you're already running) specifically to avoid ever writing another controller-specific annotation again — the config is portable across any Gateway-API-conformant implementation by design. You'd reach for Istio specifically when the actual requirement extends past ingress into service-mesh territory (mTLS between every internal service, fine-grained east-west traffic policy) — genuine extra operational weight that isn't worth carrying just to expose a few HTTP routes.

**Interview POV & Edge Cases**

Classic prompt: "Users report intermittent 502s on one route — walk me through it." The layered answer: check the gateway/ingress controller's own logs first for the specific upstream error (this tells you 502 vs 503 vs 504 definitively, rather than guessing from the browser's generic error page); check backend Service endpoints for zero-ready-backend (that's actually 503, a different bug entirely); check the actual backend pod's logs and resource state (Module 6/7 territory) if the connection genuinely is failing/timing out.

**Gotchas**:

- An Ingress/HTTPRoute created with no matching IngressClass/GatewayClass installed — silently inert forever, always check kubectl get ingressclass / kubectl get gatewayclass before debugging routing rules themselves.
- Accepted: True read as "this is working" when ResolvedRefs: False is sitting right next to it — always check both conditions, not just the first one you see.
- Missing ReferenceGrant for a cross-namespace backendRef or a cross-namespace TLS certificateRef on a Gateway — same underlying mechanism, two different places it bites.
- The ACME HTTP-01 chicken-and-egg trap — debugging a stuck Certificate in isolation without first confirming the underlying route it depends on was ever actually working.
- Copy-pasting a controller-specific Ingress annotation from a blog post into a cluster running a different controller (or worse, into an HTTPRoute, where annotations like that have zero effect at all) — a very real mistake when following examples without checking which controller they assumed.

**The 'Better Way' (Evolution)**

Gateway API itself already is the "better way" relative to plain Ingress — but the frontier past it is the GAMMA initiative (Gateway API for Mesh Management and Administration), which extends the same HTTPRoute/GatewayClass model to service-mesh east-west traffic, not just north-south ingress — the goal being one unified routing API instead of Ingress/Gateway for external traffic and a completely separate mesh-specific CRD set for internal traffic. Separately, eBPF-native L7 implementations (Cilium's Envoy-based Gateway API support, or Cilium's own native L7 policy) are collapsing CNI and L7 gateway into a single dataplane, reducing the number of distinct proxy hops a request has to cross.

**Module 8 Lab Pack**

These labs create disposable test objects attached to your existing GatewayClass/Gateway rather than touching your real Envoy Gateway + cert-manager setup — find your actual names first:

```
kubectl get gatewayclass
kubectl get gateway -A
kubectl get clusterissuer
```

<img width="880" height="485" alt="image" src="https://github.com/user-attachments/assets/3daf79c4-47de-44a8-815e-899b62eca553" />

**Lab 1 — Ingress With No Matching Controller**

**Break**

```
kubectl get ingressclass   # confirm — likely empty on your Gateway-API-first cluster
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orphan-ingress
spec:
  rules:
  - host: orphan.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nonexistent-svc
            port:
              number: 80
EOF
```

**Predict**: Does the API server reject this at creation time, or accept it?

**Diagnose**

```
bash
kubectl get ingress orphan-ingress
kubectl describe ingress orphan-ingress
```

It's accepted without complaint — the API server never validates that a controller exists to implement it. describe shows no address ever assigned, no meaningful events, ever. Now contrast with a real, working object:

```
bash
kubectl get httproute -A -o wide
kubectl describe httproute <one-of-your-real-routes> -n <its-namespace>
```

Your real HTTPRoute shows populated status conditions (Accepted: True) — direct proof Envoy Gateway is actually reconciling it, unlike the orphaned Ingress.

**Fix**

```
kubectl delete ingress orphan-ingress
```

**Debrief**: Both objects look identically "successfully created" from kubectl apply's perspective — the entire difference is whether anything is watching. kubectl get ingressclass / kubectl get gatewayclass should always be your first check, before you spend any time debugging routing rules that might never even be evaluated.

**Lab 2 — HTTPRoute Backend Not Found**

```
bash
kubectl create deployment httproute-test --image=nginx
kubectl expose deployment httproute-test --port=80
```

**Break**

```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: broken-backend-route
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["broken-backend.test.local"]
  rules:
  - backendRefs:
    - name: totally-wrong-svc-name
      port: 80
EOF
```

**Predict**: Will Accepted be True or False?

**Diagnose**

```
kubectl describe httproute broken-backend-route
```

Expect Accepted: True (nothing wrong with the route's own syntax or its attachment to the Gateway) but ResolvedRefs: False, reason BackendNotFound.

**Fix**

```
kubectl patch httproute broken-backend-route --type=json \
  -p='[{"op":"replace","path":"/spec/rules/0/backendRefs/0/name","value":"httproute-test"}]'
kubectl describe httproute broken-backend-route
```

**Debrief**: This is the two-condition trap made concrete — checking only Accepted and seeing True would have you believing everything's fine while every request 404s. ResolvedRefs is a separate check on a separate concern (does what this route points to actually exist), and you have to read it explicitly.

**Lab 3 — Cross-Namespace Backend, No ReferenceGrant**

**Break**

```
kubectl create namespace route-ns
kubectl create namespace backend-ns
kubectl create deployment cross-ns-backend --image=nginx -n backend-ns
kubectl expose deployment cross-ns-backend --port=80 -n backend-ns

cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cross-ns-route
  namespace: route-ns
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["cross-ns.test.local"]
  rules:
  - backendRefs:
    - name: cross-ns-backend
      namespace: backend-ns
      port: 80
EOF
```

**Predict**: Same failure reason as Lab 2, or something different — remember, the Service genuinely exists this time.

**Diagnose**

```
bash
kubectl describe httproute cross-ns-route -n route-ns
```

Expect ResolvedRefs: False again, but reason RefNotPermitted — a distinctly different reason string than Lab 2's BackendNotFound, because this time the object exists, it's just not permitted to be referenced.

**Fix**

```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-route-ns
  namespace: backend-ns
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: route-ns
  to:
  - group: ""
    kind: Service
EOF
kubectl describe httproute cross-ns-route -n route-ns
```

**Debrief**: Two different failure reasons living under the exact same False status condition — you have to read the reason field to know whether you're fixing a typo (Lab 2) or granting a genuine cross-namespace permission (this lab). Clean up:

```
bash
kubectl delete namespace route-ns backend-ns
```

**Lab 4 — cert-manager ACME Chicken-and-Egg**

**Break** — a disposable test Certificate for a domain that can't actually complete validation:

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: broken-acme-test
spec:
  secretName: broken-acme-test-tls
  issuerRef:
    name: <your-clusterissuer-name>
    kind: ClusterIssuer
  dnsNames:
  - some-domain-you-dont-actually-control.example.com
EOF
```

**Predict**: Will this cert ever reach Ready: True? Where will the actual stuck point show up?

**Diagnose**

```
kubectl describe certificate broken-acme-test
kubectl get order
kubectl describe order <order-name>
kubectl get challenge
kubectl describe challenge <challenge-name>
```

Expect the Certificate stuck Ready: False, an Order created, and a Challenge stuck with a validation error — the ACME server genuinely cannot reach that domain to complete HTTP-01 verification.

**Fix**

```
kubectl delete certificate broken-acme-test
kubectl delete secret broken-acme-test-tls --ignore-not-found
```

**Debrief**: This is the chicken-and-egg trap directly — in your real project, HTTP-01 issuance depends entirely on the routing to cert-manager's temporary ACME solver pod already working correctly. If you're ever debugging a stuck Certificate at the same time as a broken route, always fix the routing first and re-attempt cert issuance second — debugging both simultaneously means you can't tell which one is actually causing the symptom you're looking at.

**Lab 5 — 502 vs 503 vs 504, Deliberately Produced**

```
kubectl create deployment http-demo --image=nginx
kubectl expose deployment http-demo --port=80
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-demo-route
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["http-demo.test.local"]
  rules:
  - backendRefs:
    - name: http-demo
      port: 80
EOF
```

(Point curl at whatever address/hostname your Gateway is actually reachable on — adjust the Host header or DNS as your existing project setup requires.)

**Part A — 503 (zero ready endpoints)**

```
bash
kubectl scale deployment http-demo --replicas=0
curl -H "Host: http-demo.test.local" -sI http://<your-gateway-address>/
```

**Predict** before running: which of the three codes do you expect with zero backends?

**Part B — 502 (backend exists, wrong port)**

```
kubectl scale deployment http-demo --replicas=1
kubectl patch service http-demo --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":9999}]'
curl -H "Host: http-demo.test.local" -sI http://<your-gateway-address>/
```

**Predict**: same code as Part A, or different — the Service now has a "ready" endpoint, it's just pointing at nothing listening.

**Part C — 504 (backend slow to respond)**

```
kubectl patch service http-demo --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":80}]'
kubectl set image deployment/http-demo nginx=kennethreitz/httpbin
curl -H "Host: http-demo.test.local" -sI --max-time 10 http://<your-gateway-address>/delay/30
```

**Predict**: will this time out at the gateway's own configured timeout, or wait the full 30 seconds?

**Diagnose across all three:** check the Envoy Gateway pod's own logs at each stage —

```
bash
kubectl logs -n <envoy-gateway-namespace> -l <envoy-gateway-pod-label> --tail=20
```

The proxy's own logs name the exact upstream failure category each time, which is far more reliable than inferring from the HTTP status code alone.

**Fix**

```
kubectl delete deployment http-demo
kubectl delete service http-demo
kubectl delete httproute http-demo-route
```

**Debrief**: Three genuinely different root causes, three genuinely different codes, and — critically — none of them tell you which earlier module's failure is actually behind them in a real incident. A 502 could be Module 2's crashed container, a 503 could be Module 1's failing readiness probe, a 504 could be Module 7's resource-starved node. The gateway/ingress layer is where all seven prior modules' failures ultimately collapse into a generic-looking HTTP error — which is exactly why reading the proxy's own access/error logs first, rather than guessing from the status code shown to the user, is the fastest way back to the real cause.

**Lab 6 — Gateway TLS Listener, Cross-Namespace Secret, No ReferenceGrant**

**Break**

```
kubectl create namespace tls-secret-ns
openssl req -x509 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -days 1 -nodes -subj "/CN=test.local"
kubectl create secret tls fake-tls --cert=/tmp/tls.crt --key=/tmp/tls.key -n tls-secret-ns

cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: test-tls-gateway
spec:
  gatewayClassName: <your-existing-gatewayclass-name>
  listeners:
  - name: https
    protocol: HTTPS
    port: 8443
    tls:
      certificateRefs:
      - name: fake-tls
        namespace: tls-secret-ns
EOF
```

**Predict**: Same underlying mechanism as Lab 3, applied to a Gateway instead of an HTTPRoute — will this listener come up?

**Diagnose**

```
bash
kubectl describe gateway test-tls-gateway
```

Expect the listener rejected — Programmed: False or a listener-specific condition citing the cross-namespace cert reference isn't permitted.

**Fix**

```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-cert
  namespace: tls-secret-ns
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: default
  to:
  - group: ""
    kind: Secret
EOF
kubectl describe gateway test-tls-gateway
```

**Fix / cleanup**

```
bash
kubectl delete gateway test-tls-gateway
kubectl delete namespace tls-secret-ns
```

**Debrief**: Same ReferenceGrant mechanism as Lab 3, but this time gating Gateway → Secret instead of HTTPRoute → Service. This exact pattern is directly relevant to your own project the moment TLS certs live in a shared, dedicated namespace separate from app namespaces — a common, sensible real-world layout that requires exactly this grant to actually function.

**Module 9 — Observability-Driven Triage**

Modules 1–8 each gave you a layer-specific diagnostic tree. But a real incident never announces its layer up front — you get "checkout is slow" or "some requests are failing," and the actual skill is figuring out which of those eight modules you're even in, fast, before you touch a single kubectl describe. That's what this module is for: using metrics, logs, traces, and Events together to narrow the search space in minutes instead of guessing your way through eight modules sequentially.

**The 'Why' (The Problem)**

Without a systematic observability layer, engineers facing a vague symptom have exactly two bad options: guess based on what broke last time (fast, biased, often wrong), or manually check every layer in order (thorough, but far too slow under real incident pressure with people watching a dashboard turn red). The three pillars — metrics (is something wrong, and roughly where), logs (what exactly happened, once you've narrowed), traces (which specific hop in a multi-service request chain was the actual problem) — exist specifically to compress that search. Kubernetes' own Events (used constantly throughout Modules 1–8) are actually the fourth, most granular signal — but they're short-lived by default, which is itself a real operational gap this module addresses directly.

**Deep-Dive Mechanics**

**Golden signals first — before anything else.** Google's SRE framework (latency, traffic, errors, saturation) gives you a fast first pass at which module you're probably in:

- Error rate spike → likely app-layer (Module 8) or a crash loop (Module 2).
- Latency spike, normal error rate → likely resource pressure (Module 7), DNS amplification (Module 4), or slow storage (Module 5).
- Saturation climbing (CPU/memory/disk approaching limits) → Module 7 territory, and importantly a leading indicator — it predicts an incoming eviction or OOM kill before it happens, rather than explaining one after the fact.
- Traffic drops to zero → could be DNS (Module 4), could be a scheduling failure preventing pods from ever running at all (Module 3), could be a Gateway/Ingress misconfiguration (Module 8).

**Two complementary lenses, for two different kinds of things. The USE method** (Utilization, Saturation, Errors) is for resources — nodes, disks, CPUs — Module 6/7 territory. The RED method (Rate, Errors, Duration) is for request-driven services — Module 8 territory. Reaching for the wrong one wastes time: USE-style dashboards won't tell you which specific route is erroring; RED-style dashboards won't tell you a node is about to hit DiskPressure.

**kube-state-metrics vs. cAdvisor/node-exporter answer genuinely different questions** — this is the single most senior-differentiating point in this module. kube-state-metrics exposes what Kubernetes thinks the state is — Deployment replica counts, Pod phase, PVC status — essentially kubectl get/describe turned into scrapeable metrics, so you can alert on "available replicas < desired" without polling. cAdvisor (embedded in kubelet, per-pod/container cgroup stats) and node-exporter (raw OS/kernel metrics) tell you what's actually happening underneath. A divergence between the two is itself a diagnostic signal — a Pod reporting Ready: True via kube-state-metrics while node-exporter shows that node's available memory collapsing is a ticking clock, not a healthy state, and a purely kubectl-based investigation (Modules 1–8 in isolation) would show nothing wrong until the eviction has already happened.

**Kubernetes Events are TTL'd — and this is a real, common operational gap.** As covered back in Module 1, Events are namespace-scoped and expire (default ~1 hour, via kube-controller-manager's --event-ttl). Without shipping them somewhere durable, any postmortem written the next day cannot see a single Event from the actual incident window — the exact signal you'd want most for root-cause review is gone by the time anyone sits down to write it up. Tools like kubernetes-event-exporter exist specifically to ship Events into Loki/Elasticsearch/whatever durable store you already run, turning a 1-hour-lifespan signal into a permanently queryable one.

**Distributed tracing solves a problem log correlation genuinely cannot scale to solve**. In a real microservices chain — directly relevant to your e-commerce platform's Gateway → auth → cart → payment → inventory flow — a single slow user request might touch five services. Without a trace ID propagated through every hop, "checkout is slow" can only be investigated by manually eyeballing timestamps across five separate log streams — slow, error-prone, and it gets worse as service count grows. A trace turns that into "the payment service's call to the bank's API accounted for 4.2 of the total 4.5 seconds" — directly actionable, no manual correlation required. OpenTelemetry is the vendor-neutral instrumentation standard for this — it's not a backend itself, it's the propagation/collection layer that can export to whatever backend you choose (Tempo, Jaeger, a commercial APM), which matters because it decouples your app's instrumentation from any single vendor's product.

**Log architecture, and why logs --previous isn't enough at scale.** kubelet redirects every container's stdout/stderr to files under /var/log/pods (via the CRI LogPath). A node-level shipper DaemonSet (Promtail, Fluent Bit) tails those files and ships them to a central store. This matters because Module 1's kubectl logs --previous trick only works if the old container instance still technically exists on the node — once a pod is truly gone (deleted, or the node itself replaced), only centrally-shipped logs survive. If you haven't shipped logs off-node, you've lost them the moment the pod's garbage collected.

**The Alternative Landscape**

<img width="857" height="465" alt="image" src="https://github.com/user-attachments/assets/320757ec-1148-4227-9fc5-2be07bf27043" />

<img width="888" height="262" alt="image" src="https://github.com/user-attachments/assets/d3357b38-10a8-42ac-844c-a37b9d7bb3be" />

**Worth naming explicitly:** your monitoring stack can fall victim to the exact same Module 1–8 failure modes it's meant to help you diagnose — if self-hosted Prometheus runs out of disk (Module 5/7) or gets evicted (Module 7) during the very incident you need it for, you've lost visibility exactly when it matters most. This is a genuine reason larger orgs pay for managed observability specifically for their alerting path, even while self-hosting everything else. You'd reach for eBPF-based tooling specifically for services like your FastAPI microservices where hand-instrumenting every single one with OpenTelemetry SDK calls is real, ongoing engineering cost you might rather avoid for a first pass at visibility.

**Interview POV & Edge Cases**

**Signature senior prompt:** "You're paged for elevated p99 latency on checkout — walk me through your first five minutes, before you touch a single pod." They're testing whether you reach for dashboards first or start kubectl describe-ing things at random. Expected order: check the RED signal for the specific service (which of rate/errors/duration actually moved, and when); check whether it's isolated to that service or cluster-wide (cross-reference against node-level USE metrics); check for a trace pinpointing the actual slow hop; only then drop into the specific module's targeted kubectl commands the metrics/trace pointed you toward.

**Gotchas**:

- Treating "no alert fired" as "nothing is wrong" — alerting coverage gaps are real and common; absence of an alert is not evidence of absence, especially for a genuinely novel failure mode.
- The "who watches the watchmen" trap — trying to diagnose an observability stack outage using the very stack that's down, directly paralleling Module 6's recursive "kubectl is down because of the thing you're debugging" problem.
- Trusting kube-state-metrics' Ready: True without cross-checking actual node saturation — the divergence point above, made concrete.
- Chasing a metrics-visible symptom while the actual root cause is something standard exporters don't capture well by default — Module 4's conntrack exhaustion or Module 7's clock skew are both classic examples that need specific instrumentation to show up in Prometheus at all.
- Manually eyeballing timestamps across multiple services' logs instead of reaching for a trace ID — doesn't scale past a couple of hops and produces wrong conclusions under time pressure.

**The 'Better Way' (Evolution)**

AI-assisted RCA tools (k8sgpt, Robusta — full circle from Module 1) now ingest metrics, logs, Events, and traces together and propose a starting hypothesis, collapsing "which of the eight modules is this" from minutes to seconds for common, previously-seen patterns. Still a hypothesis generator that needs verification against the diagnostic trees you've actually built through this masterclass — not a replacement for them, especially for genuinely novel failures. And eBPF-based always-on flow/syscall visibility is shifting the whole field from "you had to know what to instrument in advance" toward "it's captured by default, queryable after the fact" — closing exactly the kind of blind spot Module 4 and Module 7 kept surfacing.

**Module 9 Lab Pack**

```
Lab	Focus	Trains
1	Install a real metrics stack	The three metric sources, in one Helm chart
2	kube-state-metrics vs node-exporter divergence	Spotting a ticking-clock state before it becomes an incident
3	RED signal through your real Envoy Gateway	Closing the loop with Module 8, edge metrics without app instrumentation
4	Multi-layer chained failure — the capstone	Metrics-first triage across two modules at once
5	Event retention gap	Why Events alone aren't enough for postmortems
```

**Lab 1 — Install a Real Metrics Stack**

**Break (nothing to break — building the tool you'll use for the rest of the module)**

```
bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get pods -n monitoring
```

**Predict**: How many distinct scrape targets will this one chart set up, and which of the three metric sources from the Deep-Dive will they cover?

**Diagnose**

```
bash
kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9090:9090 &
curl -s localhost:9090/api/v1/targets | grep -o '"job":"[^"]*"' | sort -u
```

Expect targets covering kubelet/cAdvisor, kube-state-metrics, node-exporter, the API server, and the Prometheus/Grafana/Alertmanager components themselves.

```
bash
kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d
kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80 &
```

**Fix**: nothing broken — this is your working baseline for the rest of the module.

**Debrief**: One Helm install gets you exactly the three metric sources from the Deep-Dive, plus query/visualization/alerting on top. This is the actual gap between "I've learned Kubernetes" and "I operate Kubernetes" that most tutorials skip entirely.

**Lab 2 — kube-state-metrics vs node-exporter Divergence**

```
bash
kubectl label node worker-1 pressure-test=true
kubectl describe node worker-1 | grep -A3 Allocatable   # size the hog against real capacity
```

**Break** — a pod that stays healthy from Kubernetes' point of view while genuinely straining the node:

```
bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quiet-pressure
spec:
  nodeSelector: {pressure-test: "true"}
  containers:
  - name: hog
    image: polinux/stress
    resources:
      requests: {memory: "100Mi"}
    args: ["stress", "--vm", "1", "--vm-bytes", "<~70% of allocatable>", "--vm-hang", "0"]
EOF
```

**Predict**: Will kube-state-metrics show anything alarming about this pod at all?

**Diagnose** — via Prometheus (Lab 1's port-forward), PromQL only, no kubectl describe yet:

```
kube_pod_status_ready{pod="quiet-pressure"}
```

Expect 1 — perfectly Ready, nothing alarming. Now check the node level:

```
kube_node_info{node="worker-1"}
```

(use this to find the matching instance label for node-exporter metrics on that node, then)

```
node_memory_MemAvailable_bytes{instance="<worker-1's instance label>"}
```

Watch this collapse toward zero — a real divergence: the Kubernetes-object-level view says fine, the OS-level view says nearly exhausted.

**Fix**

```
bash
kubectl delete pod quiet-pressure
kubectl label node worker-1 pressure-test-
```

**Debrief**: This is the divergence from the Deep-Dive made visible with real numbers. A purely kubectl-based investigation (everything from Modules 1–8) would show a perfectly healthy Ready: True pod right up until the eviction actually happens — the metrics-level view is what lets you see it coming instead of explaining it after the fact.

**Lab 3 — RED Signal Through Your Real Envoy Gateway**

```
bash
kubectl create deployment red-demo --image=kennethreitz/httpbin --replicas=2
kubectl expose deployment red-demo --port=80
```

```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: red-demo-route
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["red-demo.test.local"]
  rules:
  - backendRefs:
    - name: red-demo
      port: 80
EOF
```

**Break** — generate a mix of success and error traffic through the gateway, not directly at the pod, so Envoy's own metrics capture it:

```
bash
kubectl run traffic-gen --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl exec traffic-gen -- sh -c 'for i in $(seq 1 100); do curl -s -H "Host: red-demo.test.local" -o /dev/null http://<gateway-address>/status/200; curl -s -H "Host: red-demo.test.local" -o /dev/null http://<gateway-address>/status/500; done'
```

**Predict**: Without instrumenting httpbin itself with any Prometheus client library, will you still get real request-rate and error-rate metrics for this route?

**Diagnose** — find Envoy's actual metric names first (naming varies by version, so discover rather than guess):

```
bash
kubectl get pods -n <envoy-gateway-namespace>
kubectl exec -n <envoy-gateway-namespace> <envoy-proxy-pod> -- curl -s localhost:19000/stats/prometheus | grep -i upstream_rq | head -20
```

Use whatever request-count/response-code-class metrics you find there in Prometheus's query UI to isolate the 5xx count and request duration for this specific route.

**Fix / cleanup**

```
bash
kubectl delete deployment red-demo
kubectl delete service red-demo
kubectl delete httproute red-demo-route
kubectl delete pod traffic-gen
```

**Debrief:** This closes the loop between Module 8 and Module 9 directly — Envoy Gateway, like any real gateway/ingress controller, exposes its own Prometheus metrics for exactly the RED signals, with zero changes to the backend application. This is genuinely how production teams get golden-signal visibility at the edge without hand-instrumenting every single microservice — a real advantage for something like your FastAPI-based platform, where you'd otherwise need per-service OpenTelemetry instrumentation just to get this baseline view.

**Lab 4 — Multi-Layer Chained Failure (Capstone)**

This lab deliberately combines a Module 7 failure with a Module 8 symptom — the whole point is finding the real root cause via metrics before touching any per-pod kubectl describe.

```
bash
kubectl label node worker-1 pressure-test=true
kubectl create deployment chained-app --image=nginx --replicas=4
kubectl expose deployment chained-app --port=80
```

```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: chained-app-route
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["chained-app.test.local"]
  rules:
  - backendRefs:
    - name: chained-app
      port: 80
EOF
```

**Break**

```
bash
kubectl run memory-hog --overrides='{"spec":{"nodeSelector":{"pressure-test":"true"}}}' \
  --image=polinux/stress -- stress --vm 1 --vm-bytes <~70% of worker-1's allocatable> --vm-hang 0
```

**Predict:** Before running a single kubectl get pod or describe, what should your first metrics check be, and what do you expect it to show?

**Diagnose — metrics-first, in this order:**

```
bash
# 1. Edge signal — is latency/error rate on chained-app elevated at all? (Lab 3's technique)
# 2. If yes, check whether the Deployment is actually at full capacity:
kube_deployment_status_replicas_available{deployment="chained-app"}
kube_deployment_spec_replicas{deployment="chained-app"}
bash
# 3. If available < spec, check saturation on the node those missing replicas were on:
node_memory_MemAvailable_bytes{instance="<worker-1's instance>"}
```

Only after this metrics-driven narrowing, confirm with targeted kubectl:

```
bash
kubectl get pods -l app=chained-app -o wide
kubectl describe node worker-1 | grep -A5 Conditions
```

**Fix**

```
bash
kubectl delete pod memory-hog
kubectl label node worker-1 pressure-test-
kubectl get pods -l app=chained-app -w   # confirm self-healing back to 4/4
bash
kubectl delete deployment chained-app
kubectl delete service chained-app
kubectl delete httproute chained-app-route
```

**Debrief**: This is the whole masterclass's thesis in one exercise. The symptom was felt two layers away (Module 8's edge) from where it actually originated (Module 7's node pressure), with Module 1/3-style replica-count and pod-status as the connecting middle layer. kubectl describe pod on one of the surviving, perfectly healthy replicas would have shown nothing wrong at all — it's the metrics-first sequence (edge signal → replica count → node saturation) that actually finds a multi-hop root cause efficiently, instead of randomly describe-ing objects hoping to stumble onto the right one.

**Lab 5 — The Event Retention Gap**

```
bash
kubectl create configmap event-test-trigger --from-literal=x=1
kubectl run event-test --image=nginx:this-tag-does-not-exist-v99 --restart=Never
kubectl get events --field-selector involvedObject.name=event-test --sort-by=.lastTimestamp
```

**Predict**: If you came back to check this Event in two hours, would it still be here?

**Diagnose** — check your actual configured retention rather than waiting two hours:

```
bash
grep event-ttl /etc/kubernetes/manifests/kube-controller-manager.yaml || echo "no explicit --event-ttl set — default 1h applies"
```

**Fix:** nothing to break/repair here — the real fix is architectural: ship Events somewhere durable. kubernetes-event-exporter (the resmoio project) is the standard tool for this — it watches the Events API and forwards to Loki/Elasticsearch/whatever backend you already run, rather than letting this signal vanish after an hour. Worth installing alongside the stack from Lab 1 if you want Events to survive past their default TTL — check the project's own repo for current install instructions, since exact chart coordinates shift over time.

**Cleanup**

```
bash
kubectl delete pod event-test
kubectl delete configmap event-test-trigger
```

**Debrief**: The default 1-hour TTL is genuinely short relative to real incident-review timelines — a postmortem written the next morning cannot see a single Event from the actual incident window unless it was already shipped somewhere durable. This is a gap teams almost always discover after needing an Event that's already gone, never before.

**Module 10 — Live Incident Simulations**

Nine modules of diagnostic trees are only worth something if you can pull the right one out under pressure with no hints. This final module drops that scaffolding — no "Module 4 territory" labels, no roadmap telling you which tree to use. Just what you'd actually get: a page.

This module runs differently from the last nine. I'll give you an incident briefing — exactly the amount of information you'd have in the first sixty seconds of getting paged, nothing more. You tell me what you'd check first and why, I respond as the system would (command output, logs, whatever you ask for), and we work it like a real incident until you've found root cause. No peeking at "which module this is" — that's the whole point. We'll run through a few of these.

**📟 Incident 1**

**#incident-4471 — P2 — Checkout error rate elevated**

Paged 14 minutes ago. Dashboard shows checkout HTTP error rate at ~8%, up from a baseline near 0%. Latency (p50/p99) looks normal for the requests that do succeed. No deploys in the last 6 hours. This has been going on for at least 15 minutes and isn't trending up or down — it's holding steady around 8%.

You're on call. What's the first thing you check, and what are you expecting to see?

**Read the clues like evidence, not just facts:**

- Only checkout is affected, everything else is fine. This rules out anything cluster-wide — no shared control-plane issue, no CoreDNS problem, no node-wide resource crisis. Whatever this is, it's local to checkout's own pods or its specific route.
- Steady 8%, flat for 15 minutes — not climbing, not shrinking. This is the clue people miss. A slow memory leak or building resource pressure would show worsening errors or climbing latency over time. A one-off blip would spike and recover. A perfectly flat, non-trending percentage is the signature of something structural — a fixed portion of capacity is broken, and the rest is fine, and nothing is actively getting worse.
- Latency is completely normal on the requests that succeed. If the whole service were under shared resource pressure (CPU throttling, GC pauses, a saturated node), you'd expect degraded latency even on the successful requests. Normal latency on successes means the healthy replicas are genuinely, fully healthy.

Put together: this pattern — flat error percentage + normal latency on success + isolated to one service — points at one unhealthy replica out of several, still absorbing a share of traffic, while its siblings are completely fine. Not a service-wide problem. A specific-pod problem hiding inside a healthy-looking Deployment.

**So the next command isn't "check the logs" — it's check replica-level health, not just replica count:**

```
kubectl get pods -l app=checkout -o wide
```

Imagine this comes back: 5 replicas, 4 with RESTARTS: 0, one with RESTARTS: 14 — but its current STATUS still says Running. This is the trap: if you'd only glanced at STATUS, everything looks fine. The restart count is what tells you one replica has been dying repeatedly.

```
kubectl describe pod <that-one-pod> | grep -A3 "Last State"
```

Say this shows Terminated, Reason: OOMKilled, Exit Code: 137 — straight out of Module 1's exit-code table. That pod is getting memory-killed on a cycle, probably every couple of minutes, likely because its memory limit is sized too tight for some request payloads it occasionally handles.

Here's the part that explains the errors, not just the restarts — a healthy Deployment usually pulls a crashing pod out of Service traffic via readiness (Module 1), so a crash alone shouldn't cause client-facing errors, just reduced capacity. But right after each restart, there's a brief window where kube-proxy's endpoint update and any already-open connections (Module 4's conntrack lesson) can still route a request or two at that pod before it's fully marked unready again. Multiply that small window by "this happens every couple of minutes, 24/7" and you get a small, steady, non-trending error percentage — exactly your 8%, exactly flat, exactly isolated to this one service.

Root cause: one replica's memory limit is undersized for real traffic, causing periodic OOM kills, and the brief post-restart routing gap is what's actually producing client-visible errors. Fix: raise that container's memory limit (or investigate why that specific replica sees larger payloads — sharding, uneven load balancing, etc.), same move as Module 1 Lab 4.

The transferable skill here, more than this specific answer: a flat, non-trending, partial-failure rate is almost always a per-replica problem, not a systemic one — go straight to individual pod health (restart counts, not just status) before you look anywhere else.

**📟 Incident 2**

**#incident-4502 — P3 — Inventory service latency spikes**

- Ops has noticed a pattern over the last few days: inventory service p99 latency spikes to roughly 3-4x its normal baseline every day, consistently around 12:00–13:00 and again 18:00–19:00 — right around lunch and dinner order peaks for the mobile store.
- 
- Error rate stays flat at 0% the entire time — nothing is failing, requests just get slow and then recover once the peak passes.
- Restart count across all inventory replicas: 0, the whole time.
- Node-level CPU utilization during these windows looks moderate — nowhere near maxed out.
- No deploys or config changes correlate with when this started.

You're investigating. What's the first thing you'd check, and what are you hoping it tells you?

**Read the clues like evidence:**

- Recurs at exact, predictable traffic peaks (lunch/dinner) — not random. This immediately says "load-correlated," not "flaky infra." Whatever this is, it only manifests under concurrency, not idle.
- Latency only, zero errors, zero restarts. This rules out OOM (Incident 1's answer), rules out crash loops, rules out anything that kills a process. Requests are getting slow, not failing — the pod is alive and functioning the entire time.
- Node CPU "moderate" during the spike. This is the clue that trips people up — the natural assumption is "if it were CPU, the node would show maxed-out CPU." It doesn't. That's the tell, not a reason to rule CPU out.

**Here's the trap in that last clue:** node-level aggregate CPU utilization and per-container CPU throttling are two completely different signals. Kubernetes enforces resources.limits.cpu via the kernel's CFS quota mechanism — each container gets a fixed slice of CPU time per 100ms scheduling period. If a container's actual work during a burst exceeds its slice, the kernel pauses it for the rest of that period, even if the node has plenty of idle CPU sitting right next to it on another core. The node-wide dashboard averages across everything and looks fine; the one throttled container experiences real, repeated pauses that add up to exactly the kind of latency spike you're seeing — with zero errors, zero restarts, because throttling delays execution, it never kills anything.

And it explains the timing perfectly: inventory presumably does more actual CPU-bound work per request at lunch/dinner peaks (more concurrent inventory checks), pushing usage against a limits.cpu value that's been sized the same the whole time — nothing changed in config, the peak traffic is what's newly colliding with an existing ceiling.

**The check that actually confirms it** — not raw CPU%, but the throttling-specific cAdvisor metric from Module 9's toolkit:

```
rate(container_cpu_cfs_throttled_periods_total{pod=~"inventory-.*"}[5m])
```

against

```
rate(container_cpu_cfs_throttled_seconds_total{pod=~"inventory-.*"}[5m])
```

If these spike in lockstep with the 12–13h/18–19h windows, that's your confirmation — and cross-check kubectl get deployment inventory -o jsonpath='{.spec.template.spec.containers[0].resources}' to see how tight that CPU limit actually is relative to what the pods are trying to use during peak.

Fix: raise limits.cpu (or, if the team's comfortable with the trade-off, drop the CPU limit entirely and rely on requests.cpu plus node-level capacity planning — a genuinely debated but common production pattern specifically to avoid this exact throttling behavior).

**Worth naming honestly:** I don't have DB/downstream metrics in this scenario, and CPU throttling isn't the only hypothesis that fits — connection-pool exhaustion against inventory's database under peak concurrency produces an almost identical signature (latency up, zero errors, zero restarts, load-correlated). If the throttling metric above came back flat, that's exactly where I'd look next: pool wait-time metrics on the DB side, or a trace (Module 9) showing whether the time is being spent in inventory's own CPU work or waiting on a downstream call. Keeping two live hypotheses and using one specific metric to eliminate one is normal, correct practice here — not a sign you guessed wrong the first time.

**Simulating Incident 2's Mechanism — CPU Throttling, With Real Numbers**

You already have Prometheus running from Module 9 Lab 1, so let's use it for real instead of just talking about container_cpu_cfs_throttled_periods_total abstractly.

**Break** — a container demanding far more CPU than its limit allows:

```
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: throttle-demo
spec:
  replicas: 1
  selector:
    matchLabels: {app: throttle-demo}
  template:
    metadata:
      labels: {app: throttle-demo}
    spec:
      containers:
      - name: stress
        image: polinux/stress
        resources:
          requests: {cpu: "100m"}
          limits: {cpu: "100m"}
        args: ["stress", "--cpu", "2", "--timeout", "600s"]
EOF
```

This container is capped at 0.1 CPU core but is actively trying to burn 2 full cores worth of work — a guaranteed, heavy throttling case.

**Predict**: Will node-level CPU utilization look alarming, or will it look basically fine — same as Incident 2's clue?

**Diagnose**

```
kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9090:9090 &
```

In Prometheus's query UI (localhost:9090):

```
rate(container_cpu_cfs_throttled_periods_total{pod=~"throttle-demo.*"}[1m])
```

Expect this climbing steadily, non-zero — real, ongoing throttling. Now check the node itself — pick worker-1's (or whichever node it landed on) instance label and query:

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance="<that node's instance>"}[1m])) * 100)
```

Expect this to look moderate, unremarkable — exactly like the node CPU in Incident 2's briefing, because one container's 100m ceiling has nothing to do with how busy the rest of the node's cores are.

**Fix**

```
bash
kubectl patch deployment throttle-demo --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/cpu","value":"1500m"}]'
```

Requery the throttled-periods rate — it should fall toward zero within a minute or two as the new limit gives the container enough quota to actually run.

**Cleanup**

```
kubectl delete deployment throttle-demo
```

**Debrief**: This is Incident 2's exact mechanism, with real numbers proving it rather than inference. container_cpu_cfs_throttled_periods_total is the one metric that reliably catches this pattern — node-level CPU% will mislead you every single time it's a per-container limit problem rather than genuine node-wide load.

**📟 Incident 3**

**#incident-4519 — P2 — New notifications service unreachable**

- notifications was deployed for the first time about 20 minutes ago as part of a planned rollout. All 3 pods show Running, 3/3 Ready, zero restarts — deployment itself looks completely healthy in every dashboard.
- 
- But every single request to notifications fails immediately — not slow, not intermittent, instant failure, 100% of the time, since the moment it deployed.
- No other service is affected.
- No relevant Events on the pods themselves — they've been quietly healthy the whole time.

You're investigating. What's the first thing you'd check, and what would you expect it to tell you?

**Read the clues like evidence:**

- Pods are 3/3 Ready, zero restarts, no Events at all — genuinely, boringly healthy. This is a strong signal to stop looking at the pods entirely. Whatever's wrong isn't inside them.
- Instant failure, not slow. This rules out anything resembling a timeout — no upstream connection attempt is even happening. Compare this to a 504 (backend reachable but slow) or 502 (connection attempted, then failed/reset) — both of those take some time to fail. An instant failure is the signature of the proxy rejecting the request before it ever tries to reach a pod — which is exactly what a 503 is (Module 8: zero healthy endpoints, the proxy doesn't even attempt an upstream call).
- 100% failure rate, since the very first request, on a brand-new service. Not intermittent, not degrading — it's never worked. That points away from anything that develops over time (leaks, throttling, drift) and toward something that was simply misconfigured from the moment it was created.

Put together: healthy pods + instant + total + brand-new deploy points at the routing layer having nowhere to send traffic, not the pods themselves. The single most common cause of exactly this pattern: a Service selector that doesn't actually match the pod labels — pods run fine, Kubernetes just never wired the Service up to them.

**Diagnose**

```
bash
kubectl get endpoints notifications
```

Expect <none> — this is the confirming moment. The pods are Ready, but the Service has zero endpoints, meaning nothing is actually behind it as far as any proxy is concerned.

```
bash
kubectl get service notifications -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels | grep notifications
```

This is almost always where you find it — say the Service selector is app: notifications but the Deployment's pod template actually labels pods app: notification-service (or similar near-miss typo). Kubernetes doesn't warn you about this anywhere — a Service with a selector matching zero pods is a completely valid, silently-accepted object.

**Fix**

```
bash
kubectl patch service notifications --type=json \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"notification-service"}]'
kubectl get endpoints notifications   # confirm pod IPs now populate
```

**Root cause:** label mismatch between the Service selector and the Deployment's pod template, present since the very first deploy — the pods were never actually behind the Service at all, so every request hit an empty backend set and got an instant 503 at the gateway.

**Worth being honest about the other live hypothesis** I'd have kept open until that first kubectl get endpoints came back: if endpoints had shown pod IPs, the next place to look would've been Module 8's HTTPRoute/Ingress status — Accepted: True but ResolvedRefs: False from a typo'd backendRef name produces an extremely similar instant-100%-failure signature. The endpoints check is exactly the one command that cleanly splits those two hypotheses apart, which is why it's the right first move here rather than jumping straight to describe httproute.

**Simulating Incident 3 — Selector Mismatch, Zero Endpoints, Instant 503**

**Break**

```
bash
kubectl create deployment notif-demo --image=nginx --replicas=3
kubectl expose deployment notif-demo --port=80 --name=notif-svc
kubectl get pods -l app=notif-demo   # confirm 3/3 Running, Ready — genuinely healthy
```

Now introduce the exact mismatch from the incident:

```
bash
kubectl patch service notif-svc --type=json \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"wrong-label"}]'
```

**Predict**: Will the pods' own status change at all? What will kubectl get endpoints show?

**Diagnose**

```
bash
kubectl get pods -l app=notif-demo   # still 3/3 Ready — untouched, exactly like the incident
kubectl get endpoints notif-svc      # <none>
kubectl get service notif-svc -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pods --show-labels | grep notif-demo
```

Line the two up side by side — the Service is asking for app=wrong-label, the pods are labeled app=notif-demo. Zero overlap, zero endpoints, and nothing about it is visible from the pods' own perspective at all.

Route it through your real Gateway to see the actual instant failure a client would hit, tying back to Module 8:

```
bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: notif-demo-route
spec:
  parentRefs:
  - name: <your-gateway-name>
    namespace: <your-gateway-namespace>
  hostnames: ["notif-demo.test.local"]
  rules:
  - backendRefs:
    - name: notif-svc
      port: 80
EOF

curl -sI -H "Host: notif-demo.test.local" --max-time 3 http://<gateway-address>/
```

Expect an immediate 503 — no delay, because Envoy never even attempts an upstream connection when it has zero endpoints to choose from. Compare that instant response time against Module 8 Lab 5's 504 timeout — the speed of the failure alone tells you which one you're looking at, before you read a single status code.

**Fix**

```
bash
kubectl patch service notif-svc --type=json \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"notif-demo"}]'
kubectl get endpoints notif-svc   # pod IPs populate
curl -sI -H "Host: notif-demo.test.local" http://<gateway-address>/   # 200 now
```

**Cleanup**

```
bash
kubectl delete deployment notif-demo
kubectl delete service notif-svc
kubectl delete httproute notif-demo-route
```

**Debrief**: You just watched the fastest, most silent failure mode in this whole masterclass — no restart, no Event, no unhealthy condition anywhere, just one string not matching another. kubectl get endpoints on any suspiciously "healthy but unreachable" service should be close to a reflex at this point.

**📟 Incident 4**

**#incident-4538 — P1 — Order processing completely stalled**

- orders Deployment shows 5/5 Ready, all healthy, zero restarts. kubectl get pods looks completely normal.
- But orders placed in the last ~25 minutes have not progressed past "pending" — nothing downstream is happening at all.
- Metrics dashboard: request rate into orders is normal. Error rate: 0%. Latency: normal.
- Someone mentions a teammate ran kubectl scale deployment orders-worker --replicas=0 about 30 minutes ago "to test something" and then got pulled into another meeting.

What's the first thing you'd check, and — given that last detail — what's your working theory already?

**What you said, evaluated:**

- "Check if pods are running" — you checked orders' pods specifically, and they're already reported 5/5 Ready. That's a dead end on its own — nothing there points anywhere.
- "Scale back to 5" — correct fix, correct target. But notice why you knew to target orders-worker: the ticket told you outright. In a real page, that detail often doesn't show up as a tidy confession — you'd have to find it yourself.

**So here's the piece worth building as a reflex**: the ticket shows you a dashboard for orders — 5/5 Ready, 0% errors, normal latency — and it's telling the truth. orders genuinely is healthy. The mistake would be assuming that dashboard represents the whole system. "Orders stall at pending, nothing progresses downstream" is the signature of a missing consumer, not a broken API — something has to be pulling items off a queue/pending-state and moving them forward, and that something is architecturally a different deployment than the one accepting the request. The diagnostic move, without being told the answer, is:

```
bash
kubectl get deployments
```

— not just orders, all of them — specifically looking for anything sitting at 0/0 that shouldn't be. That's how you'd spot orders-worker cold, the same way you'd spot Incident 3's selector mismatch: by checking the thing nobody mentioned yet, not just the thing the dashboard already flagged as unhealthy (here, nothing was flagged at all).

**Confirm before you fix:**

```
bash
kubectl get deployment orders-worker
```

Expect 0/0 — confirms the theory before you act on it, rather than scaling blind based on secondhand info from a teammate's Slack message.

**Fix**

```
bash
kubectl scale deployment orders-worker --replicas=5
kubectl get pods -l app=orders-worker -w
```

Watch the "pending" backlog actually start draining once workers come back — that drain is your confirmation, not just the replica count going green.

The transferable lesson: a healthy dashboard for the component that's named in the complaint doesn't mean the system is healthy — it means that specific component is healthy. "Nothing progresses" with zero errors anywhere is almost always a missing worker/consumer, not a broken producer, and kubectl get deployments (the whole list, not the one you were pointed at) is the cheap first move that finds it.

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

SOLUTIONS:

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
















# Day 14 — Runtime Security: Falco, Trivy & Supply Chain

*Days 13 covered preventing bad things from being admitted. Today covers detecting bad things that are already running. 
This is the difference between a firewall and an IDS — you need both.*

## 🧠 Part 1: The Security Layers

```
Layer 1: Supply chain    — is the image safe before it runs?     (Trivy, Cosign)
Layer 2: Admission       — does the pod spec meet policy?         (PSA, Gatekeeper)
Layer 3: Runtime         — is the container behaving at runtime?  (Falco)
Layer 4: Network         — is the traffic pattern normal?         (NetworkPolicy, Cilium)
```

*Day 16 = Layer 2. Today = Layers 1 and 3. Layer 4 = Day 4 (NetworkPolicy).*

A container can pass all admission checks and still be compromised at runtime — a valid nginx image that gets exploited via a CVE, then runs a reverse shell. 
Falco catches that. Trivy would have warned you about the CVE before deploy.

## 🦅 Part 2: Falco — Runtime Security

*Falco is a CNCF project that uses Linux kernel syscalls (via eBPF or kernel module) to detect anomalous behavior in running containers. 
It watches what processes are actually doing — not what they're configured to do.*

**How Falco works**

```
Container process makes syscall (open file, exec, connect)
         ↓
Falco's eBPF probe intercepts at kernel level
         ↓
Falco evaluates against rules
         ↓
Rule matches → alert to stdout / file / Slack / Falcosidekick
```

*This is why Falco catches things PSA can't — PSA only sees the pod spec at admission time. Falco sees every syscall at runtime, even if the container was admitted cleanly.*

**Install Falco**

```
# Add Falco Helm repo
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# Install Falco with Disabled Redis Persistence
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.type=modern-ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true \
  --set falcosidekick.webui.redis.storageEnabled=false \
  --set tty=true

# Watch Falco come up — takes 2-3 minutes (compiles eBPF probe)
kubectl get pods -n falco -w

# Check Falco is running and loaded rules
kubectl logs -n falco \
  $(kubectl get pod -n falco -l app.kubernetes.io/name=falco \
    -o jsonpath='{.items[0].metadata.name}') \
  | grep -E "Loading|Enabled|error" | head -20

# Access Falcosidekick UI
kubectl port-forward -n falco svc/falco-falcosidekick-ui --address 0.0.0.0 2802:2802 &
# http://localhost:2802

# Access Falco UI : Username : admin, password : admin

# Retrieve Current Settings
helm get values falco -n falco | grep -A 5 "webui"

# Set Custom Credentials
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --reuse-values \
  --set falcosidekick.webui.user="admin:your_new_password"
```

## 📋 Part 3: Falco Rules

*Rules define what "anomalous" means. Falco ships with hundreds of default rules. You write custom ones for your environment.*

**Rule anatomy**

```
# A Falco rule has five parts:
- rule: Shell spawned in container          # human-readable name
  desc: Detect shell execution in a container  # description
  condition: >                              # when to fire (uses Falco fields)
    spawned_process
    and container
    and shell_procs
  output: >                                 # what to log
    Shell spawned in container
    (user=%user.name container=%container.name
     image=%container.image.repository
     shell=%proc.name parent=%proc.pname
     cmdline=%proc.cmdline)
  priority: WARNING                         # DEBUG/INFO/NOTICE/WARNING/ERROR/CRITICAL/ALERT/EMERGENCY
  tags: [container, shell, mitre_execution]
```

**Key Falco fields**

```
# Process fields
proc.name          — process name (bash, curl, nc)
proc.cmdline       — full command line
proc.pname         — parent process name
proc.pid           — process ID
proc.args          — process arguments

# Container fields
container.name     — container name
container.id       — container ID
container.image.repository — image name
container.image.tag

# User fields
user.name          — username inside container
user.uid           — user ID

# File fields
fd.name            — file/socket path
fd.directory       — directory of file

# Network fields
fd.sip             — server IP
fd.sport           — server port
fd.cip             — client IP
```

**Default rules that fire most often**

```
# These come built-in — understand what they detect:

# 1. Terminal shell in container
rule: Terminal shell in container
# Fires when: someone kubectl exec -it pod -- bash
# Why matters: attacker or insider running commands interactively

# 2. Write below binary dir
rule: Write below binary dir
# Fires when: process writes to /bin, /usr/bin, /sbin
# Why matters: malware installing itself

# 3. Read sensitive file untrusted
rule: Read sensitive file untrusted
# Fires when: non-system process reads /etc/passwd, /etc/shadow
# Why matters: credential harvesting

# 4. Contact K8s API Server From Container
rule: Contact K8s API Server From Container
# Fires when: container connects to API server IP:6443
# Why matters: compromised pod trying to escalate via K8s API

# 5. Outbound Connection to C2 Servers
rule: Outbound Connection to C2 Servers
# Fires when: connection to known C2 IPs/domains
# Why matters: malware phoning home

# 6. Launch Privileged Container
rule: Launch Privileged Container
# Fires when: container with privileged:true starts
# Why matters: immediate container escape risk
```

**Writing custom rules**

```
# Custom rule: detect curl/wget exfiltration attempt
- rule: Data Exfiltration Attempt
  desc: Detect curl or wget used inside a container
  condition: >
    spawned_process
    and container
    and proc.name in (curl, wget, nc, ncat, netcat)
    and not proc.pname in (apt, apt-get, yum, pip)
  output: >
    Possible data exfiltration
    (user=%user.name proc=%proc.name
     args=%proc.args container=%container.name
     image=%container.image.repository:%container.image.tag)
  priority: WARNING
  tags: [network, exfiltration, container]

---
# Custom rule: detect crypto mining
- rule: Crypto Mining Activity
  desc: Detect processes associated with crypto mining
  condition: >
    spawned_process
    and container
    and (
      proc.name in (xmrig, minerd, cpuminer, ethminer)
      or proc.cmdline contains "stratum+tcp"
      or proc.cmdline contains "mining.pool"
    )
  priority: CRITICAL
  tags: [cryptomining, container]

---
# Custom rule: detect sensitive env var access
- rule: Access to Secret Environment Variables
  desc: Detect reading /proc/<pid>/environ which exposes env vars
  condition: >
    open_read
    and container
    and fd.name glob "/proc/*/environ"
    and not proc.name in (ps, top, htop)
  output: >
    Environment variables accessed
    (proc=%proc.name container=%container.name
     file=%fd.name user=%user.name)
  priority: WARNING

---
# Macro to whitelist your own app
- macro: url_shortener_process
  condition: >
    container.image.repository = "ghcr.io/youruser/url-shortener"
    and proc.name = "python"

# Custom rule with whitelist
- rule: Unexpected Network Connection
  desc: Container making unexpected outbound connections
  condition: >
    outbound
    and container
    and not url_shortener_process
    and not proc.name in (curl, wget)  # if your app legitimately uses these
    and fd.sport != 443
    and fd.sport != 80
  priority: NOTICE
```

**Deploy custom rules**

```
# Add to Helm values as a ConfigMap
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set-file falco.rules_file[0]=/path/to/custom-rules.yaml

# Or via ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco
data:
  custom-rules.yaml: |
    - rule: Data Exfiltration Attempt
      desc: Detect curl or wget inside containers
      condition: >
        spawned_process and container
        and proc.name in (curl, wget, nc)
      output: >
        Exfiltration attempt
        (container=%container.name proc=%proc.name args=%proc.args)
      priority: WARNING
      tags: [network, exfiltration]
EOF
```

## 🔫 Part 4: Triggering and Reading Falco Alerts

```
# Watch Falco alerts in real time
kubectl logs -n falco \
  $(kubectl get pod -n falco -l app.kubernetes.io/name=falco \
    -o jsonpath='{.items[0].metadata.name}') \
  -f | grep -v "^{" | head -50

# Trigger the "shell in container" rule
kubectl run trigger-test --image=nginx:1.25 -- sleep 3600
kubectl exec -it trigger-test -- bash
# ← Falco fires: "Terminal shell in container"

# In another terminal watch the alert:
kubectl logs -n falco -l app.kubernetes.io/name=falco -f | grep -i shell

# Trigger "write below binary dir"
kubectl exec trigger-test -- touch /bin/evil
# ← Falco fires: "Write below binary dir"

# Trigger "read sensitive file"
kubectl exec trigger-test -- cat /etc/shadow
# ← Falco fires: "Read sensitive file untrusted"

# Trigger "contact K8s API server"
kubectl exec trigger-test -- sh -c \
  'curl -sk https://kubernetes.default.svc/api'
# ← Falco fires: "Contact K8s API Server From Container"
```

## 🔍 Part 5: Trivy — Image and Cluster Scanning

*Trivy scans for vulnerabilities (CVEs), misconfigurations, secrets, and license issues. Three modes relevant to CKS:*

```
trivy image    — scan a container image
trivy k8s      — scan the entire cluster
trivy fs       — scan a filesystem or git repo
trivy config   — scan K8s manifests / Helm charts / Terraform
```

**Install Trivy**

```
# On Ubuntu (your EC2 instance)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin

trivy --version
```

**Image scanning**

```
# Scan an image — shows CVEs by severity
trivy image nginx:1.25

# Output:
# nginx:1.25 (debian 12.4)
# ==========================
# Total: 147 (UNKNOWN: 0, LOW: 96, MEDIUM: 42, HIGH: 8, CRITICAL: 1)
#
# ┌──────────────┬────────────────┬──────────┬───────────────┬───────────────┐
# │   Library    │ Vulnerability  │ Severity │ Installed Ver │   Fixed Ver   │
# ├──────────────┼────────────────┼──────────┼───────────────┼───────────────┤
# │ openssl      │ CVE-2023-XXXX  │ CRITICAL │ 3.0.8         │ 3.0.9         │

# Scan only CRITICAL and HIGH — ignore noise
trivy image --severity HIGH,CRITICAL nginx:1.25

# Exit with non-zero if CRITICAL found (use in CI)
trivy image --severity CRITICAL --exit-code 1 nginx:1.25

# Scan your own image
trivy image ghcr.io/youruser/url-shortener:latest

# Scan with JSON output (parse in CI)
trivy image --format json --output trivy-report.json nginx:1.25

# Scan for secrets accidentally baked into image
trivy image --scanners secret nginx:1.25

# Scan for misconfigs in Dockerfile layers
trivy image --scanners config nginx:1.25

# Ignore specific CVEs (with justification in .trivyignore)
cat <<EOF > .trivyignore
# Accepted risk — no fix available, not exploitable in our context
CVE-2023-12345
CVE-2023-67890
EOF

trivy image --ignorefile .trivyignore nginx:1.25
```

**Kubernetes cluster scanning**

```
# Scan entire cluster — all running workloads
trivy k8s --report summary --disable-node-collector

# Output:
# Summary Report for kubernetes
# ======================================
# Workload Assessment
# ┌──────────────┬────────────┬──────────────┬───────────┬──────────┐
# │  Namespace   │  Resource  │     Name     │ Misconfig │  Secret  │
# ├──────────────┼────────────┼──────────────┼───────────┼──────────┤
# │ production   │ Deployment │ url-shortener│     3     │    0     │
# │ monitoring   │ DaemonSet  │ node-exporter│     5     │    0     │

# First examine the summary
trivy k8s --report summary --disable-node-collector

# Scan only HIGH and CRITICAL findings
trivy k8s --severity HIGH,CRITICAL --report summary --disable-node-collector

# Then get detailed findings
trivy k8s \
  --severity HIGH,CRITICAL \
  --report all \
  --disable-node-collector

# Think of it as:
summary
   ↓
Where are my problems?

all
   ↓
What exactly are those problems?

HIGH/CRITICAL
   ↓
What should I investigate first?

# Scan specific namespace:
trivy k8s \
  --include-namespaces psa-restricted-test \
  --report summary \
  --disable-node-collector

# Scan kube-system namespace
trivy k8s --include-namespaces kube-system --report summary

# Detailed scan of namespace
trivy k8s \
  --include-namespaces psa-restricted-test \
  --severity HIGH,CRITICAL \
  --report all \
  --disable-node-collector

# Drill into CRITICAL vulnerabilities
trivy k8s \
  --include-namespaces kube-system \
  --severity CRITICAL \
  --scanners vuln \
  --report all

# Investigate the CRITICAL CVEs
trivy k8s \
  --include-namespaces kube-system \
  --severity CRITICAL \
  --scanners vuln \
  --report all

# Scan specific resource
trivy k8s --report all deployment/url-shortener -n production

# Check for misconfigs in your manifests (before applying)
trivy config ./k8s/
# Scans YAML files for:
# - Missing resource limits
# - Running as root
# - Privileged containers
# - hostPath mounts
# - Missing NetworkPolicies

# Scan Helm chart
trivy config ./charts/url-shortener/
```

**Trivy in CI/CD — Gate deployments on CVEs**

```
# .github/workflows/security-scan.yaml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Build image
      run: docker build -t url-shortener:${{ github.sha }} .

    - name: Scan image for CRITICAL CVEs
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: url-shortener:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: 1              # fail build on critical CVEs

    - name: Upload results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: trivy-results.sarif
      if: always()               # upload even if scan failed

    - name: Scan K8s manifests
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: config
        scan-ref: ./k8s/
        severity: HIGH,CRITICAL
        exit-code: 1
```

## 🔏 Part 6: Supply Chain Security — Cosign & Image Signing

*Supply chain attacks (like SolarWinds, Log4Shell) compromise software before it reaches you. Image signing proves an image came from a trusted source and wasn't tampered with.*

**How Cosign works**

```
Build         → sign the image digest with your private key
Push          → signature stored alongside image in registry
Deploy        → Kubernetes webhook verifies signature before admission
```

```
# Install Cosign
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

cosign version

# Generate a key pair
cosign generate-key-pair
# Creates: cosign.key (private — keep secret) + cosign.pub (public — share)

# Sign an image (after pushing to registry)
cosign sign \
  --key cosign.key \
  ghcr.io/youruser/url-shortener:v2.0.0

# Verify a signature
cosign verify \
  --key cosign.pub \
  ghcr.io/youruser/url-shortener:v2.0.0

# Keyless signing (uses Sigstore Fulcio CA — OIDC-based, no key management)
cosign sign \
  --identity-token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
  ghcr.io/youruser/url-shortener:v2.0.0

# Attach SBOM (Software Bill of Materials) to image
trivy image \
  --format cyclonedx \
  --output sbom.json \
  ghcr.io/youruser/url-shortener:v2.0.0

cosign attach sbom \
  --sbom sbom.json \
  ghcr.io/youruser/url-shortener:v2.0.0
```

**Enforce image signing with Kyverno (alternative to Gatekeeper)**

```
# Kyverno policy — only allow signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production]
    verifyImages:
    - imageReferences:
      - "ghcr.io/youruser/*"      # only your org's images
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <contents of cosign.pub>
              -----END PUBLIC KEY-----
```

## 🔐 Part 7: Seccomp and AppArmor

*These are kernel-level security mechanisms that restrict what syscalls a container can make. CKS tests both.*

**Seccomp — restrict syscalls**

```
# RuntimeDefault — uses container runtime's built-in seccomp profile
# Blocks ~44 dangerous syscalls including: ptrace, kexec, mount, umount2

spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault    # most apps work fine with this

# Localhost — use a custom profile you provide
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json  # relative to /var/lib/kubelet/seccomp/
```

**Create a custom seccomp profile**

```
# Create profile directory on the node
docker exec k8s-mastery-control-plane bash -c \
  "mkdir -p /var/lib/kubelet/seccomp/profiles"

# Write a profile that logs all syscalls (audit mode)
docker exec k8s-mastery-control-plane bash -c "
cat <<EOF > /var/lib/kubelet/seccomp/profiles/audit.json
{
  \"defaultAction\": \"SCMP_ACT_LOG\"
}
EOF"

# Write a restrictive profile — block everything except what nginx needs
docker exec k8s-mastery-control-plane bash -c "
cat <<EOF > /var/lib/kubelet/seccomp/profiles/nginx-restricted.json
{
  \"defaultAction\": \"SCMP_ACT_ERRNO\",
  \"architectures\": [\"SCMP_ARCH_X86_64\"],
  \"syscalls\": [
    {
      \"names\": [
        \"accept4\", \"access\", \"bind\", \"brk\", \"capget\",
        \"capset\", \"chdir\", \"clone\", \"close\", \"connect\",
        \"dup2\", \"epoll_create1\", \"epoll_ctl\", \"epoll_wait\",
        \"exit\", \"exit_group\", \"fcntl\", \"fstat\", \"futex\",
        \"getdents64\", \"getpid\", \"getuid\", \"ioctl\", \"listen\",
        \"lseek\", \"mmap\", \"mprotect\", \"munmap\", \"nanosleep\",
        \"open\", \"openat\", \"pipe2\", \"read\", \"recvfrom\",
        \"recvmsg\", \"rt_sigaction\", \"rt_sigprocmask\",
        \"sendfile\", \"sendmsg\", \"sendto\", \"set_tid_address\",
        \"setgid\", \"setgroups\", \"setuid\", \"socket\",
        \"stat\", \"sysinfo\", \"uname\", \"wait4\", \"write\",
        \"writev\"
      ],
      \"action\": \"SCMP_ACT_ALLOW\"
    }
  ]
}
EOF"

# Use the custom profile
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-test
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/nginx-restricted.json
  containers:
  - name: nginx
    image: nginx:1.25
EOF
```

**AppArmor — restrict file/network access**

```
# Check if AppArmor is enabled on nodes
docker exec k8s-mastery-worker bash -c \
  "cat /sys/module/apparmor/parameters/enabled"
# Y = enabled

# List loaded profiles
docker exec k8s-mastery-worker bash -c "aa-status 2>/dev/null | head -20"

# Load a custom profile onto the node
docker exec k8s-mastery-worker bash -c "
cat <<EOF > /tmp/nginx-apparmor
#include <tunables/global>
profile nginx-restricted flags=(attach_disconnected) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  network inet tcp,
  network inet udp,

  /usr/sbin/nginx mr,
  /etc/nginx/** r,
  /var/log/nginx/** w,
  /var/run/nginx.pid w,
  /tmp/** rw,

  deny /etc/passwd r,
  deny /etc/shadow r,
  deny /proc/** rw,
}
EOF
apparmor_parser -r /tmp/nginx-apparmor"

# Apply to a pod via annotation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-test
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/nginx-restricted
spec:
  containers:
  - name: nginx
    image: nginx:1.25
EOF

# Verify AppArmor profile is applied
kubectl exec apparmor-test -- cat /proc/1/attr/current
# nginx-restricted (enforce)
```

## 🖥️ Part 8: Hands-On Exercises

**Exercise 1: Trigger and capture Falco alerts**

```
# Make sure Falco is running
kubectl get pods -n falco

# Watch alerts in one terminal
kubectl logs -n falco \
  -l app.kubernetes.io/name=falco \
  -f --tail=0 &

# Trigger 5 different rules:

# 1. Shell in container
kubectl run attack-sim --image=nginx:1.25 -- sleep 3600
kubectl exec -it attack-sim -- bash -c "id && whoami"

# 2. Read sensitive file
kubectl exec attack-sim -- cat /etc/passwd

# 3. Write to binary directory
kubectl exec attack-sim -- touch /bin/backdoor 2>/dev/null || true

# 4. Outbound network connection (contact API server)
kubectl exec attack-sim -- sh -c \
  "curl -sk https://kubernetes.default.svc/api | head -5" 2>/dev/null || true

# 5. Create a setuid binary (privilege escalation attempt)
kubectl exec attack-sim -- sh -c \
  "cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash" 2>/dev/null || true

# Read the alerts
kubectl logs -n falco -l app.kubernetes.io/name=falco \
  --tail=50 | grep -E "Warning|Critical|Error" | head -20
```

**Exercise 2: Trivy full scan workflow**

```
# Scan three images of different risk levels
echo "=== Scanning old nginx ==="
trivy image --severity HIGH,CRITICAL nginx:1.20

echo "=== Scanning latest nginx ==="
trivy image --severity HIGH,CRITICAL nginx:1.25

echo "=== Scanning alpine (minimal) ==="
trivy image --severity HIGH,CRITICAL alpine:3.19

# Scan your cluster's running images
trivy k8s --report summary cluster 2>/dev/null

# Scan your K8s manifests for misconfigs
mkdir -p /tmp/manifests-scan

kubectl get deployment -A -o yaml > /tmp/manifests-scan/deployments.yaml
kubectl get pod -A -o yaml > /tmp/manifests-scan/pods.yaml

trivy config /tmp/manifests-scan/

# Focus: how many misconfigs in monitoring namespace?
kubectl get all -n monitoring -o yaml > /tmp/manifests-scan/monitoring.yaml
trivy config /tmp/manifests-scan/monitoring.yaml
```

**Exercise 3: Apply seccomp RuntimeDefault cluster-wide**

```
# Update your production namespace pods to use RuntimeDefault
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-runtime-default
  namespace: default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginx:1.25
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
  - name: tmp
    emptyDir: {}
EOF

# Verify seccomp is active
kubectl get pod seccomp-runtime-default \
  -o jsonpath='{.spec.securityContext.seccompProfile}'
# {"type":"RuntimeDefault"}

# Try to make a forbidden syscall from inside
kubectl exec seccomp-runtime-default -- sh -c \
  "unshare --pid echo test" 2>&1 || true
# Operation not permitted — seccomp blocked it
```

**Exercise 4: Full security audit of a running pod**

```
# Pick any running pod and audit it completely
POD=kube-prometheus-stack-grafana-xxx
NS=monitoring

# 1. Get full security context
kubectl get pod $POD -n $NS \
  -o jsonpath='{.spec.securityContext}' | jq .

# 2. Check container security contexts
kubectl get pod $POD -n $NS \
  -o jsonpath='{.spec.containers[*].securityContext}' | jq .

# 3. Check volumes for hostPath
kubectl get pod $POD -n $NS \
  -o jsonpath='{.spec.volumes}' | jq .

# 4. Check capabilities
kubectl get pod $POD -n $NS \
  -o jsonpath='{.spec.containers[*].securityContext.capabilities}' | jq .

# 5. Trivy scan the image it uses
IMAGE=$(kubectl get pod $POD -n $NS \
  -o jsonpath='{.spec.containers[0].image}')
trivy image --severity HIGH,CRITICAL $IMAGE

# 6. Check what Falco would say about running it
# (watch Falco logs while accessing the pod)
kubectl exec -n $NS $POD -- sh -c "id" 2>/dev/null || true
kubectl logs -n falco -l app.kubernetes.io/name=falco \
  --tail=5 | grep $NS
```

## 🎯 Part 9: Interview Questions — Day 17

**Q1: How does Falco detect threats that admission controllers miss?**

Admission controllers only see the pod spec at creation time — they validate configuration, not behavior. A perfectly well-configured nginx pod can be exploited via a CVE at runtime, spawning a reverse shell or reading /etc/passwd. Falco intercepts actual Linux syscalls via eBPF at the kernel level — it sees every process execution, file access, and network connection regardless of what the pod spec says. You can have a container with perfect security context that Falco still alerts on because it executed bash or connected to an unexpected IP.

**Q2: A pod is flagged by Trivy with a CRITICAL CVE but you can't update the image right now. What do you do?**

Short term: check if the CVE is actually exploitable in your context — many CVEs in base OS packages aren't reachable. Add it to .trivyignore with a documented justification and review date. Apply compensating controls: NetworkPolicy to restrict egress, seccomp RuntimeDefault, drop all capabilities, run as non-root. Add Falco rules to detect the specific exploit pattern. Long term: set a deadline to update the image. Add the scan to CI with --exit-code 1 so future builds catch new criticals before deploy.

**Q3: What is a Software Bill of Materials (SBOM) and why does it matter for K8s security?**

An SBOM is a complete inventory of all components, libraries, and dependencies that make up a container image — like an ingredients list. When a new CVE is announced (like Log4Shell), you can query your SBOMs across all running images to instantly know which pods are affected, instead of manually checking each image. Trivy can generate SBOMs in CycloneDX or SPDX format. Cosign can attach them to images in the registry. This is the foundation of supply chain security at scale.

**Q4: Seccomp vs AppArmor — what does each restrict and when do you use each?**

Seccomp restricts which Linux syscalls a process can make — it operates at the system call layer. If a process calls ptrace or kexec and your seccomp profile blocks it, the call returns EPERM. AppArmor restricts access to files, networks, and capabilities by process path — it's a MAC (Mandatory Access Control) system. Use RuntimeDefault seccomp on everything as a baseline (blocks ~44 dangerous syscalls with zero config). Use AppArmor for finer-grained control when you know exactly which files and networks an app needs. Both can be applied simultaneously.

**Q5: How do you prevent a compromised container from reaching the K8s API server?**

Multiple layers: NetworkPolicy blocking egress to the API server IP/port 6443. Disable ServiceAccount token automounting with automountServiceAccountToken: false on pods that don't need API access. Even if a pod has a token, RBAC should give the default SA zero permissions. Falco rule to alert on Contact K8s API Server From Container. NodeRestriction admission controller limits what kubelet can do even if compromised. In CKS terms this is "minimizing microservice footprint" — pods should only have what they explicitly need.

**Q6: Walk me through a supply chain attack and how you'd detect and prevent it.**

Attack: attacker compromises a dependency in your base image (or the registry itself), injecting a backdoor that phones home on container start. Prevention: sign all images with Cosign and enforce signature verification at admission — unsigned images can't run. Scan images in CI before push and block on CRITICAL CVEs. Use distroless or minimal base images to reduce attack surface. Pin image digests (not tags — tags are mutable). Detection: Falco alerts when the container makes unexpected outbound connections or spawns unexpected processes. Network egress policies block connections to unknown IPs. SBOM lets you trace which images contain the compromised component.













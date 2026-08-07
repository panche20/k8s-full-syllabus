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























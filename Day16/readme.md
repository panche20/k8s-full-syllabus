# Day 16 — Full CKS Mock Exam

*Week 3 boss battle. CKS is the hardest Kubernetes certification — it assumes CKA knowledge as a baseline and tests active security thinking under time pressure. Real exam: 2 hours, ~15-20 tasks, passing score 67%.*

## ⚡ Exam Setup — Do This First

```
# Aliases
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Verify cluster
kubectl get nodes
kubectl config current-context

# Check what security tools are available
which falco trivy kubeseal kubectl helm
falco --version 2>/dev/null || echo "falco: use kubectl logs"
trivy --version

# Useful for CKS — check admission plugins
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml \
  | grep admission
```

## Task 1 — Pod Security Admission (6%)

In namespace cks-1:

- Apply Pod Security Standards so that:
  - enforce mode: baseline
  - warn mode: restricted
  - audit mode: restricted
  - All at latest version
- Create a Pod named compliant-pod image nginx:1.25 that passes baseline enforce but triggers a restricted warning
- Create a Pod named violating-pod that would fail restricted — document exactly which fields cause the violation by writing them to /tmp/psa-violations.txt
- Write the namespace labels to /tmp/psa-labels.txt

**Solution:**

### Step 1: Label the namespace

```
kubectl label ns cks-1 \
pod-security.kubernetes.io/enforce=baseline \
pod-security.kubernetes.io/enforce-version=latest \
pod-security.kubernetes.io/warn=restricted \
pod-security.kubernetes.io/warn-version=latest \
pod-security.kubernetes.io/audit=restricted \
pod-security.kubernetes.io/audit-version=latest \
--overwrite
```

### Step 2: Create compliant-pod

The pod must:

- Pass baseline
- Trigger restricted warning

The easiest way is to omit all restricted security settings.

```
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: cks-1
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

**Apply it:**

```
kubectl apply -f compliant-pod.yaml
```

**You should see something similar to:**

```
Warning: would violate PodSecurity "restricted:latest":
allowPrivilegeEscalation != false
runAsNonRoot != true
seccompProfile not set
capabilities.drop not set
```

### Step 3: Create violating-pod

```
apiVersion: v1
kind: Pod
metadata:
  name: violating-pod
  namespace: cks-1
spec:
  hostNetwork: true
  containers:
  - name: nginx
    image: nginx:1.25
    securityContext:
      privileged: true
```

```
kubectl apply -f violating-pod.yaml
```

### Step 4: Document the restricted violations

```
cat <<EOF >/tmp/psa-violations.txt
privileged: true
allowPrivilegeEscalation not set to false
runAsNonRoot not set to true
seccompProfile not set
capabilities.drop not set
EOF
```

### Step 5: Save namespace labels

```
kubectl get ns cks-1 --show-labels > /tmp/psa-labels.txt
kubectl get ns cks-1 -o jsonpath='{.metadata.labels}' >/tmp/psa-labels.txt
```

**Verification**

```
kubectl get ns cks-1 --show-labels
kubectl get pods -n cks-1
kubectl describe pod compliant-pod -n cks-1
```

**Files required**

*/tmp/psa-labels.txt*
Contains namespace labels.
*/tmp/psa-violations.txt*

```
privileged: true
allowPrivilegeEscalation not set to false
runAsNonRoot not set to true
seccompProfile not set
capabilities.drop not set
```

**CKS Exam Tip (Fastest Commands)**

```
kubectl label ns cks-1 \
pod-security.kubernetes.io/enforce=baseline \
pod-security.kubernetes.io/enforce-version=latest \
pod-security.kubernetes.io/warn=restricted \
pod-security.kubernetes.io/warn-version=latest \
pod-security.kubernetes.io/audit=restricted \
pod-security.kubernetes.io/audit-version=latest --overwrite
```

*************************************************************************************************************************************************************

## Task 2 — RBAC Hardening (7%)

In namespace cks-2:

- A ServiceAccount over-privileged-sa currently has cluster-admin ClusterRoleBinding — find and remove it
- Create a new minimal Role app-role that allows only: get, list on pods and get on pods/log
- Bind over-privileged-sa to app-role in cks-2 only
- Verify before: SA can delete secrets cluster-wide
- Verify after: SA can only get/list pods in cks-2, nothing else

```
# Setup for this task — run first
kubectl create namespace cks-2
kubectl create serviceaccount over-privileged-sa -n cks-2
kubectl create clusterrolebinding over-privileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=cks-2:over-privileged-sa
```

**Solution**

### Step 1: Execute Setup Script

*First, run the provided setup commands to establish the starting environment:*

```
kubectl create namespace cks-2
kubectl create serviceaccount over-privileged-sa -n cks-2
kubectl create clusterrolebinding over-privileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=cks-2:over-privileged-sa
```

### Step 2: Verify Initial Over-Privileged Access

*Confirm that the ServiceAccount currently has cluster-wide administrative rights (e.g., ability to delete secrets):*

```
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -A
```

**Expected Output: yes**

### Step 3: Remove Over-Privileged ClusterRoleBinding

*Identify and delete the ClusterRoleBinding linking over-privileged-sa to cluster-admin:*

```
kubectl delete clusterrolebinding over-privileged-binding
```

### Step 4: Create Minimal Role app-role

*Create the Role in namespace cks-2 granting permissions to get and list pods, as well as get pod logs:*

```
kubectl create role app-role \
  --namespace=cks-2 \
  --verb=get,list --resource=pods \
  --verb=get --resource=pods/log
```

### Step 5: Bind ServiceAccount to app-role

*Bind over-privileged-sa to app-role within namespace cks-2 using a RoleBinding:*

```
kubectl create rolebinding app-role-binding \
  --namespace=cks-2 \
  --role=app-role \
  --serviceaccount=cks-2:over-privileged-sa
```

### Step 6: Verification

**1. Verify secret deletion is blocked cluster-wide:**

```
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -A
```
**Expected Output: no**

**2. Verify Pod access in cks-2 namespace:**

```
# Can list pods in cks-2
kubectl auth can-i list pods \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -n cks-2

# Can get pod logs in cks-2
kubectl auth can-i get pods/log \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -n cks-2

# Cannot delete pods in cks-2
kubectl auth can-i delete pods \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -n cks-2
```

**Expected Outputs: yes, yes, no**

**3. Verify isolation outside cks-2:**

```
kubectl auth can-i list pods \
  --as=system:serviceaccount:cks-2:over-privileged-sa \
  -n default
```

**Expected Output: no**

*************************************************************************************************************************************************************

## Task 3 — Falco Runtime Detection (8%)

Falco is running on your cluster.

- Find the Falco pod and confirm it is loading rules
- Trigger the following Falco rules and capture the alert output to /tmp/falco-alerts.txt:
    - Terminal shell in container
    - Read sensitive file (read /etc/passwd from a container)
    - Contact K8s API Server from container
- Write a custom Falco rule to /tmp/custom-rule.yaml that:
    - Detects any process named nc, ncat, or netcat running inside a container
    - Priority: CRITICAL
    - Output includes: container name, image, process name, command line
- Apply the custom rule to Falco and prove it fires when you run nc inside a pod

**Solution**

### Step 1: Confirm Falco Pod and Verify Rules are Loading

Check that the Falco pod is running and inspect its startup logs:

```
# Get the running Falco pod name
FALCO_POD=$(kubectl get pod -n falco -l app.kubernetes.io/name=falco -o jsonpath='{.items[0].metadata.name}')

# Verify rule loading in pod logs
kubectl logs -n falco $FALCO_POD -c falco | grep -i "rules loaded"
```

### Step 2: Trigger Falco Rules and Capture Alerts

Stream the Falco output into /tmp/falco-alerts.txt in the background:

```
kubectl logs -n falco $FALCO_POD -c falco -f > /tmp/falco-alerts.txt &
```

Trigger the three required rules from inside compliant-pod in namespace cks-1:

```
# 1. Terminal shell in container
kubectl exec -it -n cks-1 compliant-pod -- /bin/sh -c "exit"

# 2. Read sensitive file
kubectl exec -it -n cks-1 compliant-pod -- cat /etc/passwd

# 3. Contact K8s API Server from container
kubectl exec -it -n cks-1 compliant-pod -- curl -k https://kubernetes.default.svc
```

Verify that the events were captured in /tmp/falco-alerts.txt:

```
cat /tmp/falco-alerts.txt | grep -E "Terminal shell|sensitive file|K8s API"
```

### Step 3: Write Custom Falco Rule File

Create /tmp/custom-rule.yaml on the host:

```
cat <<'EOF' > /tmp/custom-rule.yaml
- rule: Netcat Executed in Container
  desc: Detect any process named nc, ncat, or netcat running inside a container
  condition: container.id != host and proc.name in (nc, ncat, netcat)
  output: "Netcat process executed inside container (container_name=%container.name image=%container.image.repository process=%proc.name cmdline=%proc.cmdline)"
  priority: CRITICAL
  tags: [container, network, custom]
EOF
```

### Step 4: Append Custom Rule and Force Falco Rule Reload

Append /tmp/custom-rule.yaml directly to the active /etc/falco/falco_rules.yaml file inside the running container, then signal Falco to reload its rules engine:

```
# 1. Append custom rule to falco_rules.yaml
kubectl exec -i -n falco $FALCO_POD -c falco -- sh -c 'cat >> /etc/falco/falco_rules.yaml' < /tmp/custom-rule.yaml

# 2. Send SIGHUP (signal 1) to force Falco to reload the rule file
kubectl exec -n falco $FALCO_POD -c falco -- kill -1 1
```

Alternative (if kill is restricted inside the container):

```
kubectl rollout restart daemonset falco -n falco
```

### Step 5: Verify Rule Reload and Test Execution

- Verify rule reload in Falco logs:

```
# Re-fetch FALCO_POD name if a rollout restart was performed
FALCO_POD=$(kubectl get pod -n falco -l app.kubernetes.io/name=falco -o jsonpath='{.items[0].metadata.name}')

kubectl logs -n falco $FALCO_POD -c falco --tail=30
```

(Look for log confirming: Loading rules from: /etc/falco/falco_rules.yaml | schema validation: ok)

- Trigger the rule with nc:

```
kubectl exec -it -n cks-1 compliant-pod -- nc -h
```

- Confirm the alert fired with CRITICAL priority:

```
kubectl logs -n falco $FALCO_POD -c falco | grep "Netcat process executed"
```

*************************************************************************************************************************************************************

## Task 4 — Image Security (7%)

- Scan the image nginx:1.20 with Trivy — write CRITICAL and HIGH CVEs only to /tmp/nginx-120-cves.txt
- Scan the image nginx:1.25 — write summary to /tmp/nginx-125-summary.txt
- Find which image has fewer HIGH+CRITICAL CVEs — write the answer and count to /tmp/safer-image.txt
- Scan your cluster's running workloads — write the namespace and deployment names that have HIGH or CRITICAL CVEs to /tmp/cluster-vulns.txt
- Create an OPA Gatekeeper ConstraintTemplate that blocks pods using images from any registry other than ghcr.io and registry.k8s.io

**Solution**

### Step 1: Scan nginx:1.20 for HIGH and CRITICAL CVEs

*Use Trivy's --severity flag to filter for HIGH,CRITICAL severity levels and redirect the output:*

```
trivy image --severity HIGH,CRITICAL nginx:1.20 > /tmp/nginx-120-cves.txt
```

### Step 2: Scan nginx:1.25 for Summary Output

*By default, standard trivy image includes the vulnerability summary table. To generate the summary file:*

```
trivy image nginx:1.25 > /tmp/nginx-125-summary.txt
```

### Step 3: Compare Vulnerabilities and Write Comparison Result

*Count the total HIGH and CRITICAL vulnerabilities for both images using Trivy's quiet/json or filtered output:*

```
# Count HIGH + CRITICAL for 1.20
trivy image --severity HIGH,CRITICAL nginx:1.20 --quiet | grep -E "HIGH|CRITICAL" | wc -l

# Count HIGH + CRITICAL for 1.25
trivy image --severity HIGH,CRITICAL nginx:1.25 --quiet | grep -E "HIGH|CRITICAL" | wc -l
```

Assuming nginx:1.25 has fewer vulnerabilities (since it is newer):

```
cat <<EOF > /tmp/safer-image.txt
Image: nginx:1.25
Count: <insert_counted_number>
EOF
```

### Step 4: Scan Running Cluster Workloads

*Scan the container images deployed across your running pods using Trivy, or fetch running pod images directly using kubectl and scan them:*

```
# Clear file if present
> /tmp/cluster-vulns.txt

# Iterate over running pods, extract image, and scan
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | while read -r ns pod img; do
  
  # Scan image and count HIGH/CRITICAL CVEs
  vuln_count=$(trivy image --severity HIGH,CRITICAL --quiet "$img" 2>/dev/null | grep -c -E "HIGH|CRITICAL")
  
  if [ "$vuln_count" -gt 0 ]; then
    echo "Namespace: $ns | Workload/Pod: $pod" >> /tmp/cluster-vulns.txt
  fi
done
```

### Step 5: OPA Gatekeeper ConstraintTemplate & Constraint

Gatekeeper requires two resources: a ConstraintTemplate (defining the Rego logic) and a Constraint (specifying parameters and enforcement scope).

**1. Create the ConstraintTemplate**

Apply this manifest to allow only images starting with ghcr.io/ or registry.k8s.io/:

```
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not valid_image(container.image)
          msg := sprintf("container <%v> has an invalid image registry <%v>, allowed registries are %v", [container.name, container.image, input.parameters.repos])
        }

        valid_image(image) {
          repo := input.parameters.repos[_]
          startswith(image, repo)
        }
EOF
```

**2. Create the Constraint**

Enforce the rule across all Pods in the cluster:

```
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allow-specified-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "ghcr.io/"
      - "registry.k8s.io/"
EOF
```

**Step 3: Verify Test Pod Creation**
Test Allowed Pod (Should Pass):

```
kubectl run test-allowed-pod --image=registry.k8s.io/pause:3.9 --restart=Never
```

Test Blocked Pod (Should Fail):
```
kubectl run test-blocked-pod --image=nginx:latest --restart=Never
```

Clean Up:

```
kubectl delete pod test-allowed-pod --ignore-not-found
```
*************************************************************************************************************************************************************

## Task 5 — Seccomp Profiles (6%)

- Verify seccomp is available on your nodes
- Create a Pod seccomp-default in namespace cks-5 using RuntimeDefault seccomp profile
- Create a Pod seccomp-unconfined in namespace cks-5 using Unconfined seccomp profile (no restrictions)
- Write to /tmp/seccomp-diff.txt:
    - Which syscalls RuntimeDefault blocks that Unconfined allows
    - The security implication of using Unconfined
- From inside seccomp-default, try to run unshare --pid echo test — write the result to /tmp/seccomp-block.txt

**Solution**

### Step 1: Create Namespace and Verify Seccomp Availability

*Seccomp support is enabled in Linux kernels 3.5+ and built into containerd/cri-o by default. Verify seccomp is supported on your node kernel:*

```
# Create namespace cks-5
kubectl create namespace cks-5

# Verify kernel has seccomp enabled (should return CONFIG_SECCOMP=y)
grep CONFIG_SECCOMP= /boot/config-$(uname -r)
```

### Step 2: Create Pod seccomp-default (RuntimeDefault Profile)

*Create a pod in namespace cks-5 with securityContext.seccompProfile.type: RuntimeDefault:*

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default
  namespace: cks-5
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
EOF
```

### Step 3: Create Pod seccomp-unconfined (Unconfined Profile)

*Create a pod in namespace cks-5 with securityContext.seccompProfile.type: Unconfined:*

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-unconfined
  namespace: cks-5
spec:
  securityContext:
    seccompProfile:
      type: Unconfined
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
EOF
```

### Step 4: Write Diff and Security Implication to /tmp/seccomp-diff.txt

*Create the file explaining blocked syscalls and the security risks of Unconfined:*

```
cat <<'EOF' > /tmp/seccomp-diff.txt
Syscalls blocked by RuntimeDefault but allowed by Unconfined:
RuntimeDefault blocks dangerous system calls including unshare, ptrace, reboot, kexec_load, acct, add_key, bpf, io_pgetevents, sysfs, and personality.

Security implication of using Unconfined:
Using Unconfined disables all kernel syscall filtering for the container. If a container is compromised, an attacker can execute sensitive kernel syscalls to perform container escape, access host memory/processes, load kernel modules, or disrupt the host kernel and neighboring pods.
EOF
```

### Step 5: Test unshare Inside seccomp-default and Capture Output

*Execute unshare inside seccomp-default. Because RuntimeDefault filters out the unshare syscall, the system will reject the request (Operation not permitted):*

```
kubectl exec -n cks-5 seccomp-default -- unshare --pid echo test > /tmp/seccomp-block.txt 2>&1
```

### Verification

*Verify both files were written correctly:*

```
cat /tmp/seccomp-diff.txt
echo "---"
cat /tmp/seccomp-block.txt
```

*************************************************************************************************************************************************************

## Task 6 — Network Policy Hardening (8%)

*In namespace cks-6, you have a 3-tier app:*

```
# Setup
kubectl create namespace cks-6
kubectl run frontend --image=nginx:1.25 --labels="tier=frontend" -n cks-6
kubectl run backend  --image=nginx:1.25 --labels="tier=backend"  -n cks-6
kubectl run database --image=nginx:1.25 --labels="tier=database" -n cks-6
kubectl expose pod frontend --port=80 -n cks-6
kubectl expose pod backend  --port=80 -n cks-6
kubectl expose pod database --port=80 -n cks-6
```

Apply NetworkPolicies that enforce:

- frontend accepts ingress from anywhere, egress only to backend
- backend accepts ingress only from frontend, egress only to database + DNS
- database accepts ingress only from backend, no egress except DNS
- Default deny all ingress and egress for the namespace

Verify the full matrix:

- frontend → backend: ✅
- backend → database: ✅
- frontend → database: ❌
- database → backend: ❌
- external pod → database: ❌

Write your verification commands and results to */tmp/netpol-verify.txt*

**Solution**

### Step 1: Deploy Application Workloads

*Run these commands to delete any modified or broken resources and deploy a clean 3-tier setup:*

```
# Create namespace
kubectl create namespace cks-6

# Deploy pods with required labels
kubectl run frontend --image=nginx:1.25 --labels="tier=frontend" -n cks-6
kubectl run backend  --image=nginx:1.25 --labels="tier=backend"  -n cks-6
kubectl run database --image=nginx:1.25 --labels="tier=database" -n cks-6

# Expose services on port 80
kubectl expose pod frontend --port=80 -n cks-6
kubectl expose pod backend  --port=80 -n cks-6
kubectl expose pod database --port=80 -n cks-6
```

### Step 2: Apply All NetworkPolicies in One Manifest

**This single manifest defines:**

- default-deny-all: Blocks all traffic by default in cks-6.
- frontend-netpol: Ingress from anywhere; Egress to backend (port 80) + DNS (port 53).
- backend-netpol: Ingress from frontend (port 80); Egress to database (port 80) + DNS (port 53).
- database-netpol: Ingress from backend (port 80); Egress to DNS (port 53).

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: cks-6
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: cks-6
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: cks-6
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 80
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-netpol
  namespace: cks-6
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

### Step 3: Verify the Matrix & Generate Output File

*Run the test matrix script using curl (targeting specific container instances to avoid ephemeral pod interference):*

```
# Clear existing verification file
> /tmp/netpol-verify.txt

echo "=== NetworkPolicy Verification Matrix ===" >> /tmp/netpol-verify.txt

# 1. frontend -> backend (Allowed)
kubectl exec -n cks-6 frontend -c frontend -- curl -s --connect-timeout 2 http://backend >/dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "frontend -> backend: SUCCESS (Allowed)" >> /tmp/netpol-verify.txt
else
  echo "frontend -> backend: FAILED" >> /tmp/netpol-verify.txt
fi

# 2. backend -> database (Allowed)
kubectl exec -n cks-6 backend -c backend -- curl -s --connect-timeout 2 http://database >/dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "backend -> database: SUCCESS (Allowed)" >> /tmp/netpol-verify.txt
else
  echo "backend -> database: FAILED" >> /tmp/netpol-verify.txt
fi

# 3. frontend -> database (Blocked)
kubectl exec -n cks-6 frontend -c frontend -- curl -s --connect-timeout 2 http://database >/dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "frontend -> database: SUCCESS (Blocked)" >> /tmp/netpol-verify.txt
else
  echo "frontend -> database: FAILED" >> /tmp/netpol-verify.txt
fi

# 4. database -> backend (Blocked)
kubectl exec -n cks-6 database -c database -- curl -s --connect-timeout 2 http://backend >/dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "database -> backend: SUCCESS (Blocked)" >> /tmp/netpol-verify.txt
else
  echo "database -> backend: FAILED" >> /tmp/netpol-verify.txt
fi

# 5. external pod -> database (Blocked)
kubectl run test-ext --image=nginx:1.25 -n default --rm -i --restart=Never -- curl -s --connect-timeout 2 http://database.cks-6 >/dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "external pod -> database: SUCCESS (Blocked)" >> /tmp/netpol-verify.txt
else
  echo "external pod -> database: FAILED" >> /tmp/netpol-verify.txt
fi
```

### Step 4: Validate Final Output

*Print the contents of /tmp/netpol-verify.txt:*

```
cat /tmp/netpol-verify.txt
```

**Expected Output:**

```
=== NetworkPolicy Verification Matrix ===
frontend -> backend: SUCCESS (Allowed)
backend -> database: SUCCESS (Allowed)
frontend -> database: SUCCESS (Blocked)
database -> backend: SUCCESS (Blocked)
external pod -> database: SUCCESS (Blocked)
```

*************************************************************************************************************************************************************

## Task 7 — Pod Hardening (7%)

*In namespace cks-7, create a Pod hardened-app with image nginx:1.25 that meets ALL of these requirements:*

- Runs as non-root user (UID 101)
- allowPrivilegeEscalation: false
- Root filesystem is read-only
- Seccomp: RuntimeDefault
- Drops ALL capabilities
- Adds only NET_BIND_SERVICE
- Has no hostPID, hostNetwork, hostIPC
- Resource limits: cpu 200m, memory 128Mi
- Resource requests: cpu 100m, memory 64Mi
- Mounts writable emptyDir volumes at /tmp, /var/cache/nginx, /var/run
- Passes a readiness probe on / port 80

Write the pod YAML to */tmp/hardened-pod.yaml* and apply it.
Verify it is Running and passes the readiness probe.

**Solution**

### Step 1: Create namespace

```
kubectl create ns cks-7
```

### Step 2: Create a ConfigMap

*Create a custom nginx configuration:*

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: cks-7
data:
  default.conf: |
    server {
        listen 8080;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

### Step 3: Create POD

```
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
  namespace: cks-7
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: nginx
    image: nginx:1.25

    command:
    - nginx
    - -g
    - daemon off;

    securityContext:
      runAsNonRoot: true
      runAsUser: 101
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true

      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE

    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"

    ports:
    - containerPort: 8080

    readinessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 5

    volumeMounts:
    - name: tmp
      mountPath: /tmp

    - name: cache
      mountPath: /var/cache/nginx

    - name: run
      mountPath: /var/run

    - name: nginx-config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: default.conf
      readOnly: true

  volumes:
  - name: tmp
    emptyDir: {}

  - name: cache
    emptyDir: {}

  - name: run
    emptyDir: {}

  - name: nginx-config
    configMap:
      name: nginx-config
```

*************************************************************************************************************************************************************

## Task 8 — OPA Gatekeeper Policy (8%)

Install Gatekeeper if not already installed.

*Create and apply the following policies:*

- Policy 1 — K8sRequiredLabels: All Deployments in namespace cks-8 must have labels team and environment
- Policy 2 — K8sNoLatestTag: No Pod in any namespace may use :latest image tag or an image with no tag
- Policy 3 — K8sRequiredResources: All containers must have CPU and memory limits set

Test each policy:

- For Policy 1: try creating a Deployment without labels → should fail
- For Policy 2: try kubectl run test --image=nginx → should fail
- For Policy 3: try creating a Pod with no resource limits → should fail

Write each policy violation message to */tmp/gatekeeper-violations.txt*

**Solution**

### Step 1: Install Gatekeeper (if not already installed)

```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

### Policy 1: K8sRequiredLabels

**ConstraintTemplate**

```
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        required := input.parameters.labels[_]
        not input.review.object.metadata.labels[required]
        msg := sprintf("Missing required label: %v", [required])
      }
```

**Apply**::

```
kubectl apply -f required-labels-template.yaml
```

**Constraint**

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: deployment-required-labels
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces:
    - cks-8
  parameters:
    labels:
    - team
    - environment
```

**Apply:**

```
kubectl apply -f required-labels-constraint.yaml
```

### Policy 2: K8sNoLatestTag

**ConstraintTemplate**

```
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snolatesttag
spec:
  crd:
    spec:
      names:
        kind: K8sNoLatestTag
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snolatesttag

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        endswith(container.image, ":latest")
        msg := sprintf("Image %v uses latest tag", [container.image])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not contains(container.image, ":")
        msg := sprintf("Image %v has no explicit tag", [container.image])
      }
```

**Apply**

```
kubectl apply -f no-latest-template.yaml
```

**Constraint**

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoLatestTag
metadata:
  name: deny-latest
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

### Policy 3: K8sRequiredResources

**ConstraintTemplate**

```
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredresources

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.cpu
        msg := sprintf("Container %v missing CPU limit", [container.name])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.memory
        msg := sprintf("Container %v missing memory limit", [container.name])
      }
```

**Apply**

```
kubectl apply -f required-resources-template.yaml
```

**Constraint**

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

### Testing

**Test Policy 1**

```
kubectl create ns cks-8
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: cks-8
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Expected**

```
Missing required label: team
Missing required label: environment
```

### Test Policy 2

```
kubectl run test --image=nginx
```

**Expected**

```
Image nginx has no explicit tag

or

Image nginx:latest uses latest tag
```

### Test Policy 3

```
apiVersion: v1
kind: Pod
metadata:
  name: no-limits
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

**Expected**

```
Container nginx missing CPU limit
Container nginx missing memory limit
```

**Save Violations**

```
cat <<EOF >/tmp/gatekeeper-violations.txt
Policy 1:
Missing required label: team
Missing required label: environment

Policy 2:
Image nginx has no explicit tag

Policy 3:
Container nginx missing CPU limit
Container nginx missing memory limit
EOF
```

### CKS Exam Tips

- A Gatekeeper policy requires both a ConstraintTemplate (defines the Rego logic) and a Constraint (enforces it). Missing either means the policy won't be applied.
- kubectl run --image=nginx uses the image nginx with no explicit tag, which implicitly resolves to :latest. Your policy should reject both images with an explicit :latest tag and images without any tag.
- Resource policies should check the container's resources.limits fields. If the task asks for requests as well, add corresponding checks for resources.requests.cpu and resources.requests.memory.
- After applying each template and constraint, verify they're active with:

```
kubectl get constrainttemplates
kubectl get constraints
```

*************************************************************************************************************************************************************

## Task 9 — Secrets Security (6%)

- Find all Secrets in namespace cks-9 that are mounted as environment variables in pods (not volume mounts) — write their names to /tmp/env-secrets.txt
- Explain why volume mounts are more secure than env vars for secrets — write to /tmp/secret-security.txt
- Verify etcd encryption is configured — check the API server flags and write result to /tmp/etcd-encryption.txt
- Create a Secret secure-secret in cks-9 with key PASSWORD=TopSecret123
- Verify it's accessible in a pod but NOT visible in plain text via etcd direct read

```
# Setup
kubectl create namespace cks-9
kubectl create secret generic exposed-secret \
  --from-literal=API_KEY=leaked_key_123 -n cks-9

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-consumer
  namespace: cks-9
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: exposed-secret
          key: API_KEY
EOF
```

**Solution**

### Step 1: Run Setup

*Run the setup commands to create namespace cks-9, the secret, and the consumer pod:*

```
kubectl create namespace cks-9

kubectl create secret generic exposed-secret \
  --from-literal=API_KEY=leaked_key_123 -n cks-9

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-consumer
  namespace: cks-9
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: exposed-secret
          key: API_KEY
EOF
```

### Step 2: Find Secrets Mounted as Environment Variables

*Extract secret names referenced in container environment variables (env[*].valueFrom.secretKeyRef.name or envFrom[*].secretRef.name) across pods in cks-9:*

```
kubectl get pods -n cks-9 -o jsonpath='{range .items[*].spec.containers[*]}{.env[*].valueFrom.secretKeyRef.name}{" "}{.envFrom[*].secretRef.name}{"\n"}{end}' | tr ' ' '\n' | grep -v '^$' | sort -u > /tmp/env-secrets.txt
```

**Verify Output:**

```
cat /tmp/env-secrets.txt
```

**Expected output: exposed-secret**

### Step 3: Explain Volume Mount Security vs Environment Variables

*Write the security explanation to /tmp/secret-security.txt:*

```
cat <<'EOF' > /tmp/secret-security.txt
Why volume mounts are more secure than environment variables for Secrets:

1. Process Visibility & Leakage: Environment variables are accessible to all child processes spawned inside the container and can easily leak through application error logs, crash dumps, system monitoring tools, or inspection of `/proc/1/environ`.
2. Dynamic Updates: Secrets mounted as volumes are updated automatically by K8s when the Secret resource changes (without restarting the container), whereas environment variables remain static after container startup.
3. Memory Storage (tmpfs): Mounted Secrets reside in `tmpfs` (RAM-backed memory file systems), ensuring secret payload data is never written to disk storage on the worker node.
EOF
```

### Step 4: Verify ETCD Encryption Configuration

*Check if kube-apiserver is configured with the --encryption-provider-config flag:*

```
if sudo grep -q "\--encryption-provider-config" /etc/kubernetes/manifests/kube-apiserver.yaml; then
  echo "ETCD Encryption Status: Configured" | sudo tee /tmp/etcd-encryption.txt
  sudo grep "\--encryption-provider-config" /etc/kubernetes/manifests/kube-apiserver.yaml | sudo tee -a /tmp/etcd-encryption.txt
else
  echo "ETCD Encryption Status: NOT Configured" | sudo tee /tmp/etcd-encryption.txt
fi
```

**Verify Output**

Check the contents of the generated status file:

```
cat /tmp/etcd-encryption.txt
```

**Expected Output (Default kubeadm install):**

```
ETCD Encryption Status: NOT Configured
```

### Step 5: Create Secret secure-secret

*Create the Secret in namespace cks-9:*

```
kubectl create secret generic secure-secret \
  --from-literal=PASSWORD=TopSecret123 \
  -n cks-9
```

### Step 6: Verify Accessibility in Pod vs Direct ETCD Read

**1. Verify Pod Access**

*Deploy a pod mounting secure-secret as a volume to confirm the workload can read it:*

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-reader
  namespace: cks-9
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: secure-secret
EOF

# Wait for pod to start and verify value
kubectl exec -n cks-9 secure-reader -- cat /etc/secrets/PASSWORD
```

**Expected Output: TopSecret123**

**2. Verify Direct ETCD Read (Plain Text Check)**

*Run etcdctl directly against the etcd datastore on the control plane node:*

```
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/cks-9/secure-secret
```

*************************************************************************************************************************************************************

## Task 10 — Cluster Hardening: API Server (7%)

Examine the kube-apiserver configuration and:

- Write all currently enabled admission plugins to /tmp/admission-plugins.txt
- Verify --anonymous-auth setting — write to /tmp/anonymous-auth.txt
- Verify --authorization-mode includes both Node and RBAC — write to /tmp/authz-mode.txt
- Enable the NodeRestriction admission plugin if not already enabled
- Disable the AlwaysAdmit plugin if present
- Write the before and after admission plugin list to /tmp/admission-changes.txt

**Solution**

### Step 1: Find the API Server manifest

```
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

or search for the relevant flags

sudo grep -E "enable-admission-plugins|disable-admission-plugins|anonymous-auth|authorization-mode" \
/etc/kubernetes/manifests/kube-apiserver.yaml
```

### Step 2: Save the enabled admission plugins

*Extract the value of --enable-admission-plugins:*

```
grep enable-admission-plugins /etc/kubernetes/manifests/kube-apiserver.yaml \
| sed 's/.*=//' \
| tr ',' '\n' \
> /tmp/admission-plugins.txt

Verify:

cat /tmp/admission-plugins.txt
```

### Step 3: Verify --anonymous-auth

```
grep anonymous-auth \
/etc/kubernetes/manifests/kube-apiserver.yaml \
> /tmp/anonymous-auth.txt

Typical secure value:

--anonymous-auth=false
```

*If the flag is missing, note that in recent Kubernetes versions the default is false, but for an exam you should report what is actually configured.*

### Step 4: Verify Authorization Mode

```
grep authorization-mode \
/etc/kubernetes/manifests/kube-apiserver.yaml \
> /tmp/authz-mode.txt

A secure configuration should include:

--authorization-mode=Node,RBAC

OR

--authorization-mode=Node,RBAC,...
```

*Both Node and RBAC should be present.*

### Step 5: Record the current admission plugin list

```
grep enable-admission-plugins \
/etc/kubernetes/manifests/kube-apiserver.yaml \
> /tmp/admission-changes.txt
```

### Step 6: Edit the API Server manifest

*Open the manifest:*

```
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

Find something similar to:

- --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,...
```

**Ensure NodeRestriction is included**

Example:

```
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,NodeRestriction,...
```

*If it's already present, don't add it again.*

**Disable AlwaysAdmit if present**

*Older Kubernetes versions supported AlwaysAdmit, but it has been removed for many releases. If you see it:*

```
--enable-admission-plugins=...,AlwaysAdmit,...

remove it. If you instead find:

--disable-admission-plugins=...
```

*ensure AlwaysAdmit is not enabled anywhere. On modern kubeadm clusters you typically won't see it at all.*

### Step 7: Save the updated plugin list

*Append the updated configuration:*

```
echo "----- AFTER -----" >> /tmp/admission-changes.txt

grep enable-admission-plugins \
/etc/kubernetes/manifests/kube-apiserver.yaml \
>> /tmp/admission-changes.txt

Verify:

cat /tmp/admission-changes.txt
```

### Step 8: Wait for the API Server to restart

**Because it's a static Pod, the kubelet will restart it automatically.**

*Watch the API server:*

```
kubectl get pods -n kube-system -w | grep kube-apiserver

or simply wait until:

kubectl get nodes
```

**Quick verification**

*Check that NodeRestriction is present:*

```
grep enable-admission-plugins \
/etc/kubernetes/manifests/kube-apiserver.yaml

Check the API server is healthy:

kubectl get componentstatuses

or, on newer Kubernetes versions where ComponentStatus is deprecated:
kubectl get --raw /readyz
Expected : OK
```

**CKS Exam Tips**

- Do not edit the running Pod. Always edit the static manifest at /etc/kubernetes/manifests/kube-apiserver.yaml.
- The kubelet automatically recreates the API server after the file changes—no systemctl restart is needed.
- NodeRestriction is a recommended admission plugin and is enabled by default in many recent kubeadm clusters.
- AlwaysAdmit is obsolete and is usually absent on current Kubernetes releases. If your cluster is recent (such as v1.33), you'll likely find nothing to remove.
- Before modifying the manifest, it's good practice to back it up:

```
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml \
/etc/kubernetes/manifests/kube-apiserver.yaml.bak
```

*************************************************************************************************************************************************************

## Task 11 — Audit Logging (7%)

Configure audit logging on the API server:

- Create an audit policy at /etc/kubernetes/audit/audit-policy.yaml that:
    - Logs all Secret access at RequestResponse level
    - Logs Pod create/delete at Request level
    - Logs everything else at Metadata level
    - Does NOT log get/list/watch on ConfigMaps (too noisy)
- Update the API server to use this policy:
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
- Verify: create and read a Secret — confirm it appears in the audit log
- Write 3 audit log entries (formatted) to /tmp/audit-entries.txt

**Solution**

### Step 1: Create the Audit Policy Directory and Policy File

*Create the directory /etc/kubernetes/audit/ and write the policy manifest to /etc/kubernetes/audit/audit-policy.yaml:*

```
sudo mkdir -p /etc/kubernetes/audit /var/log/kubernetes/audit

cat <<EOF | sudo tee /etc/kubernetes/audit/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 1. Do NOT log get/list/watch on ConfigMaps
  - level: None
    resources:
      - group: ""
        resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

  # 2. Log all Secret access at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]

  # 3. Log Pod create/delete at Request level
  - level: Request
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "delete"]

  # 4. Log everything else at Metadata level
  - level: Metadata
EOF
```

### Step 2: Update the kube-apiserver Static Pod Manifest

*To allow the static API server pod to read the policy file and write logs to the host, you must configure both the command flags and the host path volume mounts.*

**Edit /etc/kubernetes/manifests/kube-apiserver.yaml:**

```
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

**1. Add Flags Under spec.containers[0].command:**

```
- --audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
```

**2. Add Volume Mounts Under spec.containers[0].volumeMounts:**

```
- mountPath: /etc/kubernetes/audit
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes/audit
      name: audit-logs
      readOnly: false
```

**3. Add Volumes Under spec.volumes:**

```
- name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
  - name: audit-logs
    hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
```

### Step 3: Wait for API Server Restart & Verify Audit Log Generation

*Save the manifest and wait ~30-60 seconds for kubelet to restart the API server pod. Verify it comes back healthy:*

```
kubectl get pods -n kube-system -l component=kube-apiserver
```

### Step 4: Create a Secret and Verify Log Entries

*Trigger secret events and write 3 matching audit entries to /tmp/audit-entries.txt:*

```
# 1. Create a test secret
kubectl create secret generic audit-test-secret --from-literal=key=value -n default

# 2. Read the secret
kubectl get secret audit-test-secret -n default -o yaml

# 3. Extract 3 formatted audit log entries to /tmp/audit-entries.txt
sudo grep "audit-test-secret" /var/log/kubernetes/audit/audit.log | head -n 3 | sudo tee /tmp/audit-entries.txt
```

### Step 5: Validate Log File Output

*Display the contents of /tmp/audit-entries.txt:*

```
cat /tmp/audit-entries.txt
```
*************************************************************************************************************************************************************

## Task 12 — Supply Chain: Image Verification (6%)

- Install cosign on your EC2 instance
- Generate a cosign key pair — save to /tmp/cosign-keys/
- Sign the image nginx:1.25 (use a local registry or skip push — sign the digest)
- Verify the signature
- Create a Kyverno or Gatekeeper policy that would enforce image signature verification for namespace cks-12
- Write the policy YAML to /tmp/image-signing-policy.yaml

**Solution:**

**1. The 'Why' (The Problem)**

Container image tags are mutable pointers, not content identifiers. nginx:1.25 today and nginx:1.25 tomorrow can point to two completely different sets of bytes — because someone pushed over the tag, because the registry was compromised, or because a malicious actor with push access swapped it. Your cluster has zero built-in way to know "is this the exact image my CI pipeline built and scanned, or something else wearing its name?"

This is precisely the SolarWinds/codecov-style attack class: compromise something upstream of the deployment (build system, registry, or the tag itself) and every consumer trusts the result blindly. Kubernetes' imagePullPolicy and RBAC control who can deploy, but say nothing about whether the artifact is what it claims to be. Vulnerability scanning (Trivy/Grype) tells you if an image is safe, but not if it's authentic — a scanner will happily give a clean bill of health to a perfectly-crafted malicious image that was never scanned by your pipeline at all.

Signing closes that gap: it binds a cryptographic identity to a specific image digest, and lets the cluster refuse to admit anything that isn't provably vouched for by a trusted key.

**2. Deep-Dive Mechanics**

Key generation: cosign generate-key-pair creates an ECDSA P-256 keypair. The private key (cosign.key) is stored encrypted at rest using a password-derived key (scrypt KDF), wrapped in a PEM-like envelope. The public key (cosign.pub) is plain PEM — safe to distribute to every verifier (including your Kyverno controller).

Signing: cosign doesn't touch or re-tag your image. It:

Resolves the image reference to its immutable sha256 digest (this is why tag-based signing is actively discouraged — see the gotcha below).
Signs that digest with the private key, producing a signature over a small JSON payload ({"critical": {"identity": ..., "image": {"docker-manifest-digest": "sha256:..."}}}).
Pushes the signature as a separate OCI artifact to the same repository, tagged sha256-<digest>.sig. It never mutates the original manifest, so the digest you signed never changes.

**Verification:** 

the verifier pulls that .sig artifact, checks the signature against the provided public key, and — critically — checks that the digest inside the signed payload matches the digest of the image you're actually about to run. This second check is what stops a "signature replay" where a valid signature for image A gets attached to malicious image B.

**Admission-time enforcement (Kyverno):** 

verifyImages intercepts the Pod at admission, extracts every container/initContainer image reference, resolves each to its digest, does the same registry lookup + signature check cosign does, and denies the request if verification fails. It can also mutate the Pod spec to pin the tag to the verified digest (mutateDigest: true) — closing the TOCTOU window between "verified" and "actually pulled by kubelet."

**3. The Alternative Landscape**

<img width="875" height="487" alt="image" src="https://github.com/user-attachments/assets/6f0898fc-0f3a-4039-85ea-2ed7cc7997af" />

<img width="865" height="281" alt="image" src="https://github.com/user-attachments/assets/c6accd45-3abb-4842-9358-64e03b650976" />

**Why cosign + Kyverno for this task specifically:** 

Gatekeeper (raw OPA) has no native concept of image signatures at all — Rego evaluates policy against the AdmissionReview object it's handed; it can't independently go fetch a signature from a registry mid-evaluation. To do signature verification with Gatekeeper you need an external system like Ratify or Connaisseur running as a sidecar/provider that does the crypto work and hands OPA a verdict to check. Kyverno, by contrast, has verifyImages built directly into its admission controller — it does the registry round-trip itself. For a single-cluster exam task, Kyverno is strictly less moving parts, which is also the practical reason most teams reach for it first for this specific use case.

**4. Interview POV & Edge Cases**

**How it's asked:** 

"Design a way to guarantee only CI-built images run in production" or "how would you prevent someone from deploying latest with a tampered image" — they're fishing for: digest-pinning, signing, and admission-time enforcement as three separate layers, not one.

**Gotchas senior engineers get asked about:**

- **Signing a tag ≠ signing an image.** If you sign nginx:1.25 and someone later pushes a new image to that tag, your signature is now attached to (and still valid for) the old digest — the new push is simply unsigned. Always reason in digests; cosign will warn you if you hand it a tag.
- **Signature verification proves origin/integrity, not safety.** A perfectly signed image can still have a critical CVE. Signing and scanning are complementary controls, not substitutes.
- **Key compromise = silent trust break.** There's no revocation mechanism for a bare keypair the way there is with Rekor's transparency log in keyless mode — if cosign.key leaks, every image ever signed with it is suspect and you have no way to know which signatures were "before" vs "after" compromise.
- **The classic trap in exactly this exercise:** if you run your registry as a container on the EC2 host (localhost:5000) and then point Kyverno (running inside the cluster) at localhost:5000, it resolves to the Kyverno pod's own loopback, not your host. You must use the host's private IP (or a DNS name resolvable in-cluster).
- **Insecure (non-TLS) registries:** cosign needs --allow-insecure-registry; Kyverno's own registry client needs the same allowance configured on its controller (not just containerd's insecure-registry config, which only affects kubelet's pulls, not Kyverno's signature lookups).

**5. The 'Better Way' (Evolution)**

Static keypairs are the "must learn it" baseline, but production shops — especially the fintechs on your target list (Adyen, Stripe, Revolut) where SOC2/PCI-DSS auditors want to know exactly who signed what and when — increasingly use Sigstore keyless signing: your CI job authenticates to Fulcio via OIDC (its GitHub Actions/GitLab identity is the key), gets a 10-minute ephemeral cert, signs with it, and the signature + cert + timestamp gets recorded immutably in Rekor, a public transparency log. No cosign.key to leak or rotate, and every signature is independently auditable against "which CI job, which commit, which identity" rather than "whoever had the private key." Kyverno's verifyImages supports keyless verification against a Fulcio issuer/subject pattern instead of a publicKeys block — same policy shape, stronger provenance model. That's the answer worth giving if an interviewer pushes past "how do you sign an image" into "how do you do this at scale without a key-management headache."

### Practical Solution — Task 12

**Step 1 — Install cosign**

```
COSIGN_VERSION=$(curl -s https://api.github.com/repos/sigstore/cosign/releases/latest | grep '"tag_name"' | cut -d '"' -f4)
curl -LO "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
sudo install -m 0755 cosign-linux-amd64 /usr/local/bin/cosign
cosign version
```

**Step 2 — Generate the key pair into /tmp/cosign-keys/**

```
mkdir -p /tmp/cosign-keys && cd /tmp/cosign-keys
export COSIGN_PASSWORD='Ch0se-A-Strong-Passphrase'   # non-interactive; use your own
cosign generate-key-pair
ls /tmp/cosign-keys      # cosign.key (encrypted private), cosign.pub
```

**Step 3 — Stand up a local registry and push the image (needed so Kyverno can look up the signature later)**

```
docker run -d -p 5000:5000 --restart=always --name registry registry:2

docker pull nginx:1.25
docker tag nginx:1.25 localhost:5000/nginx:1.25
docker push localhost:5000/nginx:1.25

# Resolve the immutable digest — never sign the tag
DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' localhost:5000/nginx:1.25 | cut -d'@' -f2)
echo $DIGEST
```

**Skip-push alternative:** 

if you genuinely can't run a registry, use cosign sign-blob against the digest string itself instead of an image reference — it produces a detached .sig/.cert with no registry involved. Flag this to yourself as a limitation, though: a blob signature isn't something Kyverno/Gatekeeper can look up at admission time, since they query the registry for the signature artifact. It proves you can sign/verify, but it won't wire into Step 5 below.

**Step 4 — Sign the digest, then verify**

```
cosign sign --key /tmp/cosign-keys/cosign.key \
  --allow-insecure-registry \
  --yes \
  localhost:5000/nginx@${DIGEST}

cosign verify --key /tmp/cosign-keys/cosign.pub \
  --allow-insecure-registry \
  localhost:5000/nginx@${DIGEST}
```

A clean run prints the verified claims as JSON and exits 0.

**Step 5 — Write the Kyverno policy**

Get your EC2 host's private IP first — this is the address in-cluster pods need to use, not localhost:

```
HOST_IP=$(hostname -I | awk '{print $1}')
```

```
cat > /tmp/image-signing-policy.yaml << EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-signature
  annotations:
    policies.kyverno.io/title: Verify Image Signatures
    policies.kyverno.io/category: Supply Chain Security
    policies.kyverno.io/description: >-
      Requires all Pod images deployed into the cks-12 namespace to carry
      a valid cosign signature from the trusted key before admission.
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: verify-image-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - cks-12
      verifyImages:
        - imageReferences:
            - "${HOST_IP}:5000/*"
          mutateDigest: true
          verifyDigest: true
          required: true
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
$(sed 's/^/                      /' /tmp/cosign-keys/cosign.pub)
EOF
```

This scopes enforcement to cks-12 only, targets images pulled from your signed registry, and denies any Pod whose image lacks a valid signature from cosign.pub.

If your Kyverno install talks to an insecure (HTTP) registry, add the equivalent of --allowInsecureRegistry to the admission/background controller's args (Helm value admissionController.container.extraArgs on most installs) — otherwise the controller's own registry client will refuse the lookup even though your manual cosign verify succeeded.

*************************************************************************************************************************************************************

## Task 13 — Trivy Operator — Continuous Scanning (6%)

- Install the Trivy Operator on your cluster
- It automatically scans all workloads for vulnerabilities
- Check the scan results for the monitoring namespace
- Find any workload with CRITICAL vulnerabilities
- Write the vulnerability report summary to /tmp/trivy-operator-report.txt

```
# Install Trivy Operator
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm repo update
helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true

# Check scan results
kubectl get vulnerabilityreports -A
kubectl get configauditreports -A
```

**Solution**

### Step 1: Install Trivy Operator via Helm

*Run the commands to add the repository, update, and install the Trivy Operator in the trivy-system namespace:*

```
# Add Trivy Operator Helm repository
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm repo update

# Install Trivy Operator
helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true
```

### Step 2: Wait for Trivy Operator to Initialize

*Wait 1–2 minutes for the Trivy Operator pods to start and generate the initial Custom Resources (VulnerabilityReport / ConfigAuditReport):*

```
# Watch operator pod come up
kubectl get pods -n trivy-system -w
```

### Step 3: Inspect Scan Results in the monitoring Namespace

*Check the generated VulnerabilityReport CRDs in the monitoring namespace:*

```
# View vulnerability reports in the monitoring namespace
kubectl get vulnerabilityreports -n monitoring
```

### Step 4: Extract Detailed Vulnerability Summaries Correctly

*Run this command to inspect the severity counts for all reports in monitoring and write the summary directly:*

```
cat <<EOF > /tmp/trivy-operator-report.txt
=== Trivy Operator Vulnerability Report (monitoring namespace) ===
EOF

kubectl get vulnerabilityreports -n monitoring -o custom-columns=\
NAME:.metadata.name,\
REPOSITORY:.report.artifact.repository,\
TAG:.report.artifact.tag,\
CRITICAL:.report.summary.criticalCount,\
HIGH:.report.summary.highCount,\
MEDIUM:.report.summary.mediumCount,\
LOW:.report.summary.lowCount >> /tmp/trivy-operator-report.txt
```

### Step 5: Extract Only Workloads With CRITICAL > 0

*If you specifically need to capture only the workloads containing CRITICAL vulnerabilities (where criticalCount > 0):*

```
# Clear file
cat <<EOF > /tmp/trivy-operator-report.txt
=== Workloads with CRITICAL Vulnerabilities (monitoring) ===
EOF

# Filter for reports with CRITICAL count > 0
kubectl get vulnerabilityreports -n monitoring -o jsonpath='{range .items[?(@.report.summary.criticalCount > 0)]}{.metadata.name}{"\t"}CRITICAL: {.report.summary.criticalCount}{"\t"}HIGH: {.report.summary.highCount}{"\n"}{end}' >> /tmp/trivy-operator-report.txt

# If no workload has critical vulnerabilities, log all report summaries
if [ $(wc -l < /tmp/trivy-operator-report.txt) -eq 1 ]; then
  echo "No CRITICAL vulnerabilities found in monitoring namespace." >> /tmp/trivy-operator-report.txt
  echo -e "\n=== Full Namespace Report ===" >> /tmp/trivy-operator-report.txt
  kubectl get vulnerabilityreports -n monitoring >> /tmp/trivy-operator-report.txt
fi
```

### Step 6: Verify the Generated File

*Check the file contents:*

```
cat /tmp/trivy-operator-report.txt
```

*************************************************************************************************************************************************************

## Task 14 — Runtime Security: Restrict a Compromised Pod (8%)

*Scenario: You detect that pod compromised-app in namespace cks-14 is exhibiting suspicious behavior (making unexpected network connections).*

```
# Setup the compromised pod
kubectl create namespace cks-14
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compromised-app
  namespace: cks-14
  labels:
    app: compromised
spec:
  containers:
  - name: app
    image: nginx:1.25
EOF
```

Your tasks — respond to the incident:

- Isolate the pod immediately without deleting it — apply a NetworkPolicy that blocks ALL ingress and egress for pods with label app: compromised
- Investigate — exec into the pod and check:
    - Running processes: ps aux
    - Network connections: ss -tulpn or netstat -tulpn
    - Write findings to /tmp/investigation.txt
- Harden — patch the pod's Deployment (create one if needed) to add:
    - readOnlyRootFilesystem: true
    - allowPrivilegeEscalation: false
    - Drop ALL capabilities
    - Write the hardened spec to /tmp/hardened-compromised.yaml
- Document — write an incident response summary to /tmp/incident-summary.txt:
    - What you observed
    - What you isolated
    - What you hardened
    - What you would do next (Falco rule, image scan, replace image)

**Solution**

### Step 1: Create the namespace and compromised Pod

```
kubectl create namespace cks-14

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compromised-app
  namespace: cks-14
  labels:
    app: compromised
spec:
  containers:
  - name: app
    image: nginx:1.25
EOF
```

```
kubectl get pods -n cks-14
```

### Step 2: Isolate the Pod

*Create a deny-all NetworkPolicy.*

```
cat <<EOF > deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised
  namespace: cks-14
spec:
  podSelector:
    matchLabels:
      app: compromised
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
EOF
```

**Apply it.**

```
kubectl apply -f deny-all.yaml

Verify:

kubectl get networkpolicy -n cks-14
```

### Step 3: Investigate the Pod

Since nginx:1.25 has no diagnostic utilities, launch a debug container.

```
kubectl debug \
  -it compromised-app \
  -n cks-14 \
  --image=nicolaka/netshoot \
  --target=app
```

You should now see a shell prompt similar to:

```
root@compromised-app:/#

Run:
ps aux
ss -tulpn
netstat -tulpn
ip addr
ip route
```

### Step 4: Save Investigation

Still inside the debug container:

```
{
echo "===== Running Processes ====="
ps aux

echo
echo "===== Network Connections ====="
ss -tulpn

} > /tmp/investigation.txt
```

**Display it.**

```
cat /tmp/investigation.txt

# If you want the file on the node:

exit
kubectl cp \
cks-14/compromised-app:/tmp/investigation.txt \
/tmp/investigation.txt
```

### Step 5: Harden the Workload

The running object is a Pod. Pods are immutable for these security settings. Therefore create a Deployment manifest.

```
cat <<EOF >/tmp/hardened-compromised.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compromised-app
  namespace: cks-14
spec:
  replicas: 1
  selector:
    matchLabels:
      app: compromised
  template:
    metadata:
      labels:
        app: compromised
    spec:
      containers:
      - name: app
        image: nginx:1.25
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
EOF
```

Verify:

```
cat /tmp/hardened-compromised.yaml

If required:

kubectl delete pod compromised-app -n cks-14
kubectl apply -f /tmp/hardened-compromised.yaml
```

### Step 6: Incident Summary

```
cat <<EOF >/tmp/incident-summary.txt
Incident Response Summary

Observed
--------
Investigated the running container using an ephemeral debug container.
Reviewed running processes and active network connections.

Isolated
---------
Applied a deny-all NetworkPolicy.
Blocked all ingress traffic.
Blocked all egress traffic.

Hardened
--------
Created a Deployment manifest with:
- readOnlyRootFilesystem: true
- allowPrivilegeEscalation: false
- Drop ALL Linux capabilities

Next Steps
----------
1. Deploy Falco runtime detection.
2. Scan image using Trivy.
3. Replace compromised image.
4. Review Kubernetes Audit Logs.
5. Review container logs.
6. Rotate Secrets and credentials if compromise is confirmed.
EOF
```

Verify:

```
cat /tmp/incident-summary.txt
```

**Final Files**

```
/tmp/investigation.txt
/tmp/hardened-compromised.yaml
/tmp/incident-summary.txt
```

*************************************************************************************************************************************************************

## Task 15 — CKS Synthesis: Secure a Full Namespace (7%)

*Secure namespace cks-15 end-to-end:*

```
kubectl create namespace cks-15
```

Apply ALL of the following in a single namespace:

- PSA: enforce restricted, warn restricted
- Default-deny NetworkPolicy (all ingress + egress)
- ResourceQuota: max 5 pods, requests.cpu=500m, requests.memory=512Mi
- LimitRange: default cpu limit 200m, memory limit 128Mi
- A Role ns-developer with get/list/watch on pods, deployments, services
- A ServiceAccount dev-sa bound to ns-developer
- Deploy a fully hardened Pod secure-workload (passes restricted PSA, has NetworkPolicy allowing only port 80 ingress, has resource limits, non-root, read-only filesystem)

*Verify everything is applied:*

```
# These should all return meaningful output
kubectl get psa -n cks-15 2>/dev/null || kubectl describe ns cks-15 | grep -i security
kubectl get networkpolicy -n cks-15
kubectl get resourcequota -n cks-15
kubectl get limitrange -n cks-15
kubectl get role -n cks-15
kubectl get rolebinding -n cks-15
kubectl get pods -n cks-15
kubectl auth can-i list pods --as=system:serviceaccount:cks-15:dev-sa -n cks-15
kubectl auth can-i delete pods --as=system:serviceaccount:cks-15:dev-sa -n cks-15
```

Write verification output to /tmp/cks15-verify.txt

**Solution**

### Step 1: Create Namespace

```
kubectl create namespace cks-15
```

### Step 2: Enable Pod Security Admission (PSA)

```
kubectl label namespace cks-15 \
pod-security.kubernetes.io/enforce=restricted \
pod-security.kubernetes.io/warn=restricted \
--overwrite
```

Verify:

```
kubectl describe namespace cks-15
```

### Step 3: Default Deny NetworkPolicy

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: cks-15
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### Step 4: ResourceQuota

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq
  namespace: cks-15
spec:
  hard:
    pods: "5"
    requests.cpu: "500m"
    requests.memory: "512Mi"
EOF
```

### Step 5: LimitRange

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: cks-15
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "128Mi"
EOF
```

### Step 6: RBAC

**Create the Role**

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ns-developer
  namespace: cks-15
rules:
- apiGroups: [""]
  resources:
  - pods
  - services
  verbs:
  - get
  - list
  - watch

- apiGroups: ["apps"]
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
EOF
```

**Create ServiceAccount**

```
kubectl create serviceaccount dev-sa -n cks-15
```

**Bind Role**

```
kubectl create rolebinding dev-binding \
--role=ns-developer \
--serviceaccount=cks-15:dev-sa \
-n cks-15
```

### Step 7: Allow Port 80 NetworkPolicy

*Because the namespace has a default-deny policy, the workload needs an allow policy.*

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
  namespace: cks-15
spec:
  podSelector:
    matchLabels:
      app: secure-workload

  policyTypes:
  - Ingress

  ingress:
  - ports:
    - protocol: TCP
      port: 80
EOF
```

### Step 8: Deploy Hardened Pod

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-workload
  namespace: cks-15
  labels:
    app: secure-workload
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:stable-alpine

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"

    volumeMounts:
    - name: tmp
      mountPath: /tmp

    - name: cache
      mountPath: /var/cache/nginx

    - name: run
      mountPath: /var/run

  volumes:
  - name: tmp
    emptyDir: {}

  - name: cache
    emptyDir: {}

  - name: run
    emptyDir: {}
EOF
```

### Step 9: Verify

```
kubectl get pods -n cks-15
```

### Step 10: Verify Everything

```
{
echo "===== Namespace ====="
kubectl get ns cks-15 --show-labels

echo
echo "===== PSA ====="
kubectl get psa -n cks-15 2>/dev/null || \
kubectl describe ns cks-15 | grep pod-security

echo
echo "===== Network Policies ====="
kubectl get networkpolicy -n cks-15

echo
echo "===== ResourceQuota ====="
kubectl get resourcequota -n cks-15

echo
echo "===== LimitRange ====="
kubectl get limitrange -n cks-15

echo
echo "===== Roles ====="
kubectl get role -n cks-15

echo
echo "===== RoleBindings ====="
kubectl get rolebinding -n cks-15

echo
echo "===== Pods ====="
kubectl get pods -n cks-15

echo
echo "===== RBAC Tests ====="
kubectl auth can-i list pods \
--as=system:serviceaccount:cks-15:dev-sa \
-n cks-15

kubectl auth can-i delete pods \
--as=system:serviceaccount:cks-15:dev-sa \
-n cks-15

} > /tmp/cks15-verify.txt
```

**Verify File**

```
cat /tmp/cks15-verify.txt
```

**Expected Results**

```
NetworkPolicies
---------------
default-deny
allow-http

ResourceQuota
-------------
pods: 5
requests.cpu: 500m
requests.memory: 512Mi

LimitRange
----------
default cpu: 200m
default memory: 128Mi

Role
----
ns-developer

RoleBinding
-----------
dev-binding

Pod
---
secure-workload Running

RBAC
----
list pods   -> yes
delete pods -> no
```

****************************************************************************************************************************************

<img width="906" height="586" alt="image" src="https://github.com/user-attachments/assets/bc4ff900-4f63-4d2a-8bff-321f53e4ebe1" />

**Pass: 67%**

**CKS Speed Reference**

```
# PSA namespace labeling
kubectl label ns <name> \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Check API server flags
docker exec k8s-mastery-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml \
  | grep -E "admission|auth|audit"

# Falco alerts — live stream
kubectl logs -n falco \
  $(kubectl get pod -n falco -l app.kubernetes.io/name=falco \
    -o jsonpath='{.items[0].metadata.name}') -f

# Trivy scan — fast, high/critical only
trivy image --severity HIGH,CRITICAL --no-progress <image>

# Check seccomp on a pod
kubectl get pod <name> \
  -o jsonpath='{.spec.securityContext.seccompProfile}'

# Verify RBAC
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<ns>:<sa> -n <ns>

# NetworkPolicy — test connectivity
kubectl exec <pod> -n <ns> -- \
  wget -qO- --timeout=3 http://<target-svc>

# Find all CRBs for a SA
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] |
    select(
      .subjects[]? |
      select(.kind=="ServiceAccount" and .name=="<sa-name>")
    ) | .metadata.name'

# Audit log — find secret access
docker exec k8s-mastery-control-plane \
  grep '"resource":"secrets"' \
  /var/log/kubernetes/audit/audit.log \
  | jq -r '"\(.user.username) \(.verb) \(.objectRef.name)"' \
  | head -20

# Gatekeeper — check violations
kubectl get constraints -o json \
  | jq -r '.items[] |
    "\(.metadata.name): \(.status.totalViolations) violations"'
```


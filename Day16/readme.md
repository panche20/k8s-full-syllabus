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

*************************************************************************************************************************************************************

## Task 10 — Cluster Hardening: API Server (7%)

Examine the kube-apiserver configuration and:

- Write all currently enabled admission plugins to /tmp/admission-plugins.txt
- Verify --anonymous-auth setting — write to /tmp/anonymous-auth.txt
- Verify --authorization-mode includes both Node and RBAC — write to /tmp/authz-mode.txt
- Enable the NodeRestriction admission plugin if not already enabled
- Disable the AlwaysAdmit plugin if present
- Write the before and after admission plugin list to /tmp/admission-changes.txt

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

*************************************************************************************************************************************************************

## Task 12 — Supply Chain: Image Verification (6%)

- Install cosign on your EC2 instance
- Generate a cosign key pair — save to /tmp/cosign-keys/
- Sign the image nginx:1.25 (use a local registry or skip push — sign the digest)
- Verify the signature
- Create a Kyverno or Gatekeeper policy that would enforce image signature verification for namespace cks-12
- Write the policy YAML to /tmp/image-signing-policy.yaml

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


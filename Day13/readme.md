# Day 13 — CKS Week Begins: Pod Security + OPA Gatekeeper

*The difficulty jumps significantly. CKS assumes you already passed CKA — it tests you on securing a cluster that's already running, not just operating it. 
Today covers the two biggest CKS domains: Pod Security and Policy Enforcement.*

## 🧠 Part 1: The CKS Mindset

**CKA = "keep the cluster running"
CKS = "keep the cluster from being compromised"**

*The threat model you're defending against:*

```
Supply chain attack    — malicious image, compromised dependency
Container escape       — process breaks out of container to host
Lateral movement       — compromised pod reaches other pods/services
Privilege escalation   — pod gains more permissions than intended
Data exfiltration      — secrets read and sent outside cluster
```

*Every CKS topic maps to one of these five threats. Keep that frame in mind all week.*

## 🛡️ Part 2: Pod Security — The Evolution

*Three generations you must know:*

```
PodSecurityPolicy (PSP)     — deprecated K8s 1.21, removed 1.25
                               complex, hard to use, always-on admission
Pod Security Admission (PSA) — built-in since 1.23, GA in 1.25
                               namespace-level labels, three standards
OPA Gatekeeper               — external webhook, fully custom policies
                               most flexible, production standard
```

*PSA is what CKS tests. Gatekeeper is what production teams run. You need both.*

## 🔒 Part 3: Pod Security Admission (PSA)

*PSA enforces three security standards at the namespace level via labels. No CRDs, no webhooks — built directly into the API server.*

**The three Pod Security Standards**

- Privileged — completely unrestricted. Equivalent to no policy.
- Baseline — prevents known privilege escalations. Allows most legitimate workloads.
- Restricted — heavily restricted. Follows current pod hardening best practices.

*What Restricted blocks that Baseline allows:*

<img width="887" height="446" alt="image" src="https://github.com/user-attachments/assets/2aef7ec4-9ee2-4013-b272-11044a698b1f" />

**Three enforcement modes**

<img width="861" height="220" alt="image" src="https://github.com/user-attachments/assets/71559fb5-ebcc-45f3-8e32-cbf9b6bfc200" />

*You can mix modes — common pattern: audit + warn first, then flip to enforce once you've fixed all violations.*

**Applying PSA via namespace labels**

```
# Label format:
# pod-security.kubernetes.io/<MODE>: <STANDARD>
# pod-security.kubernetes.io/<MODE>-version: <VERSION>

# Enforce restricted on production namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest

# Check labels
kubectl get namespace production --show-labels

# Test: try to create a pod that violates restricted
kubectl run bad-pod --image=nginx:1.25 -n production
# Warning: would violate PodSecurity "restricted:latest"
# Error from server (Forbidden): pods "bad-pod" is forbidden:
# violates PodSecurity "restricted:latest":
# allowPrivilegeEscalation != false, ...
```

**Writing a pod that passes Restricted**

```
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true              # must not run as root
    runAsUser: 1000                 # specific non-root UID
    runAsGroup: 3000
    fsGroup: 2000                   # volume ownership group
    seccompProfile:
      type: RuntimeDefault          # use container runtime's default seccomp

  containers:
  - name: app
    image: nginx:1.25
    securityContext:
      allowPrivilegeEscalation: false   # critical — must be false
      readOnlyRootFilesystem: true      # best practice
      capabilities:
        drop:
        - ALL                           # drop all Linux capabilities
        add:
        - NET_BIND_SERVICE              # only add what's needed
    resources:
      requests: {memory: "128Mi", cpu: "250m"}
      limits:   {memory: "256Mi", cpu: "500m"}
    volumeMounts:
    - name: tmp                         # app needs /tmp writable
      mountPath: /tmp
    - name: cache                       # nginx cache dir
      mountPath: /var/cache/nginx

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**Exemptions — some pods need special treatment**

```
# In kube-apiserver static pod manifest — add to admission config
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline             # global default for unlabeled namespaces
      enforce-version: latest
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:                   # these namespaces are exempt
      - kube-system                 # system components need privileged
      - monitoring                  # node-exporter needs hostPath
      - kube-public
```

## 🔐 Part 4: SecurityContext Deep Dive

*SecurityContext is what PSA enforces. Understand every field.*

**Pod-level vs container-level**

```
spec:
  securityContext:           # pod-level — applies to all containers
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000            # only at pod level — sets volume ownership
    seccompProfile:          # only at pod level in older K8s versions
      type: RuntimeDefault
    sysctls:                 # kernel parameters
    - name: net.core.somaxconn
      value: "1024"

  containers:
  - name: app
    securityContext:         # container-level — overrides pod-level per container
      runAsUser: 2000        # this container runs as 2000, not 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false      # never true in production
      capabilities:
        drop: [ALL]
        add: [NET_BIND_SERVICE]
```

**Linux capabilities — the CKS exam loves this**

*Capabilities split root's omnipotence into discrete units. Dropping all and adding only what's needed is the principle of least privilege.*

```
CAP_NET_BIND_SERVICE  — bind to ports < 1024 (web servers)
CAP_NET_ADMIN         — network configuration (dangerous)
CAP_SYS_ADMIN         — huge attack surface (avoid)
CAP_SYS_PTRACE        — debug other processes (avoid)
CAP_CHOWN             — change file ownership
CAP_DAC_OVERRIDE      — bypass file permission checks (dangerous)
```

```
# See what capabilities a running container has
kubectl exec secure-pod -- cat /proc/1/status | grep Cap
# CapInh: 0000000000000000
# CapPrm: 0000000000000400
# CapEff: 0000000000000400
# CapBnd: 0000000000000400

# Decode the hex (on your machine)
capsh --decode=0000000000000400
# = cap_net_bind_service  ← only this capability remains
```

## 🚧 Part 5: OPA Gatekeeper

*Gatekeeper is a ValidatingAdmissionWebhook that enforces custom policies written in Rego. It gives you fine-grained control that PSA can't — you can enforce any rule on any field of any resource.*

```
Policy (ConstraintTemplate)  — defines the Rego logic
Constraint                   — applies the policy with parameters
Audit                        — periodically scans existing resources
```

**Install Gatekeeper**

```
kubectl apply -f \
  https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Watch it come up
kubectl get pods -n gatekeeper-system -w

# What deployed:
# gatekeeper-controller-manager  — webhook server (ValidatingWebhook)
# gatekeeper-audit               — scans existing resources for violations
```

**ConstraintTemplate — define the policy in Rego**

```
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels        # CRD name this creates
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels  # Constraint kind name
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:              # parameters the Constraint can pass in
              type: array
              items:
                type: string

  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        # For every required label...
        required := input.parameters.labels[_]
        # ...if it's missing from the resource...
        not input.review.object.metadata.labels[required]
        # ...report a violation
        msg := sprintf(
          "Resource '%v' missing required label: '%v'",
          [input.review.object.metadata.name, required]
        )
      }
```

**Constraint — apply the policy to specific resources**

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels           # must match ConstraintTemplate CRD name
metadata:
  name: require-team-label
spec:
  enforcementAction: deny         # deny | warn | dryrun
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]       # applies to Deployments
    namespaces:                   # only in these namespaces
    - production
    - staging
  parameters:
    labels:                       # passed to Rego as input.parameters
    - "team"
    - "environment"
```

```
# Apply template + constraint
kubectl apply -f constraint-template.yaml
kubectl apply -f constraint.yaml

# Test it
kubectl create deployment test --image=nginx:1.25 -n production
# Error from server (Forbidden):
# Resource 'test' missing required label: 'team'

# Compliant deployment
kubectl create deployment test \
  --image=nginx:1.25 \
  -n production \
  --dry-run=client -o yaml \
  | kubectl label -f - --local team=platform environment=production -o yaml \
  | kubectl apply -f -

# See all violations (audit mode shows existing violations)
kubectl get constraint require-team-label -o yaml | grep -A 20 violations
```

**More real-world Gatekeeper policies**

```
# Policy 1: Block latest image tag
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sbannedimagetags
spec:
  crd:
    spec:
      names:
        kind: K8sBannedImageTags
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sbannedimagetags

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        endswith(container.image, ":latest")
        msg := sprintf(
          "Container '%v' uses ':latest' tag — pin to a specific version",
          [container.name]
        )
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not contains(container.image, ":")
        msg := sprintf(
          "Container '%v' has no image tag — pin to a specific version",
          [container.name]
        )
      }
```

```
# Policy 2: Require resource limits on all containers
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
        not container.resources.limits.memory
        msg := sprintf(
          "Container '%v' missing memory limit",
          [container.name]
        )
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.cpu
        msg := sprintf(
          "Container '%v' missing CPU limit",
          [container.name]
        )
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.requests.memory
        msg := sprintf(
          "Container '%v' missing memory request",
          [container.name]
        )
      }
```

```
# Policy 3: Block privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegedcontainers
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainers
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snoprivilegedcontainers

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged == true
        msg := sprintf(
          "Container '%v' must not run as privileged",
          [container.name]
        )
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.initContainers[_]
        container.securityContext.privileged == true
        msg := sprintf(
          "Init container '%v' must not run as privileged",
          [container.name]
        )
      }
```

## 🔍 Part 6: Gatekeeper Audit Mode

*Audit scans existing resources against constraints and reports violations — even resources that existed before the policy was created.*

```
# Set a constraint to audit mode (not deny) first
kubectl patch constraint require-team-label \
  -p '{"spec":{"enforcementAction":"warn"}}' \
  --type=merge

# Wait for audit cycle (runs every 60s by default)
sleep 65

# Check audit results
kubectl get constraint require-team-label \
  -o jsonpath='{.status.violations}' | jq .
# [
#   {
#     "kind": "Deployment",
#     "name": "old-deployment",
#     "namespace": "production",
#     "message": "Resource 'old-deployment' missing required label: 'team'"
#   }
# ]

# See total violation count
kubectl get constraints
# NAME                    ENFORCEMENT-ACTION  TOTAL-VIOLATIONS
# require-team-label      warn                7
```

## 🖥️ Part 7: Hands-On Exercises

**Exercise 1: PSA namespace labeling progression**

```
# Create test namespace — start with warn mode
kubectl create namespace psa-test
kubectl label namespace psa-test \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# Try non-compliant pod — see the warning
kubectl run bad --image=nginx:1.25 -n psa-test
# Warning: would violate PodSecurity "restricted:latest": ...
# Pod is created but warned
```

### Fix the pod spec to be compliant

*For this lab, create an NGINX config:*

```
cat <<'EOF' > nginx.conf
events {}

http {
    server {
        listen 8080;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
EOF
```

**Create a ConfigMap:**

```
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n psa-test
```

**Then use:**

```
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: psa-test

spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: nginx:1.25

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

    ports:
    - containerPort: 8080

    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
      readOnly: true

    - name: tmp
      mountPath: /tmp

    - name: cache
      mountPath: /var/cache/nginx

    - name: run
      mountPath: /var/run

  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config

  - name: tmp
    emptyDir: {}

  - name: cache
    emptyDir: {}

  - name: run
    emptyDir: {}
```

```
kubectl delete pod good-pod -n psa-test

kubectl apply -f good-pod.yaml

kubectl get pods -n psa-test
```

```
# No warning — compliant
kubectl get pod good-pod -n psa-test

# Now enforce it
kubectl label namespace psa-test \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  --overwrite

# Try bad pod again — now rejected
kubectl run bad2 --image=nginx:1.25 -n psa-test
# Error from server (Forbidden): ...
```

**Exercise 2: Deploy Gatekeeper and enforce three policies**

```
# Install Gatekeeper
kubectl apply -f \
  https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

kubectl get pods -n gatekeeper-system -w

# Deploy the banned image tags template + constraint
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sbannedimagetags
spec:
  crd:
    spec:
      names:
        kind: K8sBannedImageTags
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sbannedimagetags
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        endswith(container.image, ":latest")
        msg := sprintf("Container '%v' uses :latest tag", [container.name])
      }
EOF

# Verify the CRD is Registered
kubectl get crd | grep k8sbannedimagetags

# Apply the Constraint

cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBannedImageTags
metadata:
  name: ban-latest-tag
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
EOF

# Test it
kubectl run latest-test --image=nginx:latest
# Error: Container 'latest-test' uses :latest tag

kubectl run pinned-test --image=nginx:1.25
# Works fine

# Check constraints
kubectl get constraints
```

**Exercise 3: Security context audit**

```
# Find all running pods that are NOT running as non-root
kubectl get pods -A -o json \
  | jq -r '.items[] |
    select(
      .spec.securityContext.runAsNonRoot != true and
      (.spec.containers[].securityContext.runAsNonRoot != true)
    ) |
    "\(.metadata.namespace)/\(.metadata.name)"'

# Find pods with privileged containers
kubectl get pods -A -o json \
  | jq -r '.items[] |
    select(
      .spec.containers[].securityContext.privileged == true
    ) |
    "\(.metadata.namespace)/\(.metadata.name)"'

# Find pods mounting hostPath
kubectl get pods -A -o json \
  | jq -r '.items[] |
    select(
      .spec.volumes[]?.hostPath != null
    ) |
    "\(.metadata.namespace)/\(.metadata.name): \(.spec.volumes[]? | select(.hostPath) | .hostPath.path)"'
```

**Exercise 4: Tighten a real namespace end-to-end**














## 🎯 Part 8: Interview Questions — Day 13

**Q1: PSA vs OPA Gatekeeper — when do you use each?**

PSA for broad namespace-level enforcement of the three security standards — quick to apply, zero dependencies, built into K8s. Gatekeeper for custom policies that PSA can't express: require specific labels, ban latest tags, enforce naming conventions, restrict allowed registries, check any arbitrary field. In production you run both — PSA as the baseline floor (every namespace gets at least Baseline), Gatekeeper for organization-specific rules on top.

**Q2: A developer says their pod worked last week but now gets rejected. You check and a new PSA label was added to the namespace. What do you do?**

Run kubectl run test --image=<their-image> --dry-run=server -n <namespace> to see the exact PSA violations. Fix them one by one: add allowPrivilegeEscalation: false, add seccompProfile: RuntimeDefault, drop capabilities, add runAsNonRoot: true, fix volumes. If the workload legitimately needs something restricted blocks (like node-exporter needing hostPath), either move it to a less restricted namespace or add a PSA exemption for that specific namespace.

**Q3: What is the difference between enforce, audit, and warn modes in PSA?**

Enforce hard-blocks violating pods — the API server rejects them. Audit allows them through but records a violation in the audit log — useful for discovering what would break before enforcing. Warn allows them through and returns a warning message to the user — visible in kubectl output. Best practice: start with audit+warn to identify violations across the cluster, fix them all, then flip to enforce with confidence.

**Q4: How does Gatekeeper know about resources that were created before a constraint was applied?**

The audit controller. Gatekeeper runs a separate gatekeeper-audit deployment that periodically (default 60s) scans all existing resources against all constraints and records violations in the constraint's status.violations field. This means you can apply a policy today and immediately see how many existing resources violate it — without blocking anything. This is the safe way to introduce new policies in production.

**Q5: What Linux capabilities does a container have by default, and why is dropping ALL important?**

By default Docker/containerd grants a set of ~14 capabilities including CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_NET_BIND_SERVICE, CAP_SETUID, CAP_SETGID, and others. These give more access than most apps need. CAP_DAC_OVERRIDE alone can bypass file permission checks. Dropping ALL and adding back only what's needed (usually just NET_BIND_SERVICE for web servers) means a compromised container has the minimum footprint for lateral movement or privilege escalation.

**Q6: A Gatekeeper constraint is in deny mode but violations are still getting through. Why?**

Several causes: Gatekeeper pods might be down — check kubectl get pods -n gatekeeper-system. The ValidatingWebhookConfiguration might have failurePolicy: Ignore — if the webhook is unreachable, requests go through. The constraint's match block might not cover the resource being created (wrong apiGroups, kinds, or namespaces). The ConstraintTemplate might have a Rego bug that never produces violations. Check kubectl describe constraint <name> for errors in status.





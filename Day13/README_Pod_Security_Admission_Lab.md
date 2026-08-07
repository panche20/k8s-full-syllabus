# Kubernetes Pod Security Admission (PSA) --- Production Lab & Troubleshooting Guide

## Overview

This README documents the complete hands-on Pod Security Admission (PSA)
lab and troubleshooting journey performed on a Kubernetes cluster
running the `kube-prometheus-stack`.

The goal was not merely to apply Pod Security labels, but to understand:

-   Why Pod Security Admission exists.
-   How `Privileged`, `Baseline`, and `Restricted` differ.
-   How `warn`, `audit`, and `enforce` behave.
-   Why infrastructure workloads such as Prometheus Node Exporter
    conflict with stricter Pod Security Standards.
-   How PSA evaluates workloads during Kubernetes API admission.
-   Why an admission-compliant Pod can still fail at runtime.
-   How Linux security controls such as non-root execution,
    capabilities, seccomp, privilege escalation, UID/GID, and filesystem
    permissions relate to Kubernetes security.
-   How to introduce security policies safely into an existing
    production cluster.

------------------------------------------------------------------------

# 1. The Problem PSA Solves

Containers provide isolation, but a Kubernetes Pod can deliberately
request access that weakens that isolation.

Examples include:

``` yaml
hostNetwork: true
hostPID: true

volumes:
- name: host-root
  hostPath:
    path: /
```

A workload may also run as root, retain unnecessary Linux capabilities,
allow privilege escalation, or expose a broad system-call surface.

If arbitrary workloads can request these privileges, a compromised
container can have a much larger blast radius.

Typical threats include:

-   Container escape.
-   Privilege escalation.
-   Host compromise.
-   Lateral movement.
-   Unauthorized access to node resources.
-   Data exfiltration.

Pod Security Admission provides a built-in admission-time mechanism for
enforcing predefined Kubernetes Pod Security Standards.

------------------------------------------------------------------------

# 2. Pod Security Admission

Pod Security Admission is a built-in Kubernetes admission controller.

It evaluates Pod security settings against one of three Pod Security
Standards:

1.  `privileged`
2.  `baseline`
3.  `restricted`

Policies are normally configured using namespace labels.

Example:

``` bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

Conceptually:

``` text
kubectl / controller
        |
        | API request
        v
+----------------------+
| kube-apiserver       |
+----------+-----------+
           |
           v
+----------------------+
| Admission chain      |
| Pod Security         |
+----------+-----------+
           |
     policy evaluation
           |
       +---+---+
       |       |
     allow   reject
```

PSA is an **admission control mechanism**. It does not continuously
inspect running processes inside containers.

------------------------------------------------------------------------

# 3. Pod Security Standards

## 3.1 Privileged

`privileged` imposes essentially no Pod Security Standard restrictions.

It is appropriate only where highly privileged workloads are
intentionally required and separately controlled.

Examples may include certain:

-   CNI components.
-   CSI node plugins.
-   Node-level agents.
-   Security agents.
-   Infrastructure DaemonSets.

It should not be the default for normal application namespaces.

------------------------------------------------------------------------

## 3.2 Baseline

Baseline prevents common privilege-escalation techniques while remaining
compatible with many ordinary workloads.

A normal NGINX Pod was accepted:

``` bash
kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test
```

Result:

``` text
pod/normal-pod created
```

However, when the Pod requested the host network:

``` bash
kubectl run hostnetwork-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test \
  --overrides='{"spec":{"hostNetwork":true}}'
```

Kubernetes rejected it:

``` text
Error from server (Forbidden):
pods "hostnetwork-pod" is forbidden:
violates PodSecurity "baseline:latest":
host namespaces (hostNetwork=true)
```

This demonstrated that Baseline still protects important host/container
isolation boundaries.

------------------------------------------------------------------------

## 3.3 Restricted

Restricted represents a significantly stronger Pod hardening profile.

A default NGINX Pod was rejected:

``` bash
kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-restricted-test
```

Kubernetes reported:

``` text
allowPrivilegeEscalation != false
unrestricted capabilities
runAsNonRoot != true
seccompProfile
```

This is useful because the admission error itself tells an engineer
which controls must be fixed.

------------------------------------------------------------------------

# 4. PSA Modes: Warn, Audit, and Enforce

The standard and the enforcement mode are separate concepts.

A namespace can use modes such as:

``` text
warn=restricted
audit=restricted
enforce=baseline
```

## Warn

`warn` allows the API request but returns a warning to the client.

Example:

``` text
Warning: would violate PodSecurity "restricted:latest": ...
daemonset.apps/monitoring-prometheus-node-exporter restarted
```

The second line is important: the request was allowed.

## Audit

`audit` allows the request but records the policy violation in
Kubernetes audit information when audit logging is configured
appropriately.

Audit is useful for identifying policy violations without immediately
disrupting workloads.

## Enforce

`enforce` rejects non-compliant admission requests.

Example:

``` text
Error from server (Forbidden):
pods "hostnetwork-pod" is forbidden:
violates PodSecurity "baseline:latest":
host namespaces (hostNetwork=true)
```

The Pod never becomes an accepted Kubernetes object.

------------------------------------------------------------------------

# 5. Safe Production Rollout Pattern

Do not immediately add strict enforcement to an existing namespace.

A safer progression is:

``` text
Discover
   |
   v
Audit
   |
   v
Warn
   |
   v
Identify violations
   |
   v
Remediate / design exceptions
   |
   v
Test
   |
   v
Enforce
```

This avoids discovering incompatibilities during an outage, node
replacement, rollout, or application deployment.

------------------------------------------------------------------------

# 6. Monitoring Namespace Investigation

The monitoring namespace contained a `kube-prometheus-stack`
installation.

Pods included:

-   Alertmanager.
-   Grafana.
-   Prometheus Operator.
-   kube-state-metrics.
-   Prometheus Node Exporter.
-   Prometheus.

The Node Exporter was deployed as a DaemonSet:

``` bash
kubectl get daemonset -n monitoring
```

It had one Pod per node.

------------------------------------------------------------------------

# 7. Why Node Exporter Is Special

Prometheus Node Exporter collects node-level operating-system metrics.

Inspection of the DaemonSet showed:

``` yaml
hostNetwork: true
hostPID: true

securityContext:
  fsGroup: 65534
  runAsGroup: 65534
  runAsNonRoot: true
  runAsUser: 65534
```

It also mounted host paths:

``` text
proc => /proc
sys  => /sys
root => /
```

Equivalent configuration:

``` yaml
volumes:
- name: proc
  hostPath:
    path: /proc

- name: sys
  hostPath:
    path: /sys

- name: root
  hostPath:
    path: /
```

Node Exporter therefore deliberately crosses portions of the normal
container isolation boundary so that it can observe the host.

Conceptually:

``` text
Kubernetes Node
|
+-- /proc -----------+
+-- /sys ------------+----> node-exporter
+-- / ---------------+
|
+-- Host Network -----> hostNetwork: true
|
+-- Host PIDs --------> hostPID: true
```

This is a legitimate infrastructure requirement, but it must be treated
as an explicit security exception rather than assuming that every
monitoring workload can satisfy Restricted.

------------------------------------------------------------------------

# 8. Testing Restricted Against Node Exporter

The monitoring namespace was first configured with:

``` bash
kubectl label namespace monitoring \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest

kubectl label namespace monitoring \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

Verification:

``` bash
kubectl get namespace monitoring --show-labels
```

Then the DaemonSet was restarted:

``` bash
kubectl rollout restart daemonset \
  monitoring-prometheus-node-exporter \
  -n monitoring
```

Kubernetes returned:

``` text
Warning: would violate PodSecurity "restricted:latest":
host namespaces (hostNetwork=true, hostPID=true),
allowPrivilegeEscalation != false,
unrestricted capabilities,
restricted volume types (... hostPath),
seccompProfile (...)
```

The DaemonSet still restarted successfully because the namespace was
using `warn` and `audit`, not `enforce`.

------------------------------------------------------------------------

# 9. Decoding the Restricted Violations

## 9.1 Host Network

``` yaml
hostNetwork: true
```

Normally, Pods receive their own network namespace.

With host networking enabled, the Pod uses the node's network namespace.

Conceptually:

``` text
Normal Pod

Node network namespace
       |
       +-- Pod network namespace


hostNetwork Pod

Node network namespace
       ^
       |
       +-- Pod
```

This reduces network isolation.

------------------------------------------------------------------------

## 9.2 Host PID Namespace

``` yaml
hostPID: true
```

Normally, containers operate inside isolated PID namespaces.

With `hostPID`, the Pod shares the node's process namespace, increasing
visibility into host processes.

Restricted rejects this.

------------------------------------------------------------------------

## 9.3 HostPath

Example:

``` yaml
hostPath:
  path: /
```

`hostPath` mounts part of the node filesystem into the Pod.

This is powerful and potentially dangerous.

A compromised workload with an overly permissive host mount may be able
to inspect or modify host data.

Restricted does not permit arbitrary `hostPath` volumes.

Baseline also rejected the Node Exporter's hostPath usage in this lab.

------------------------------------------------------------------------

# 10. Testing Baseline Against Node Exporter

The warning policy was changed from Restricted to Baseline:

``` bash
kubectl label namespace monitoring \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest \
  --overwrite
```

The DaemonSet was restarted again:

``` bash
kubectl rollout restart daemonset \
  monitoring-prometheus-node-exporter \
  -n monitoring
```

Result:

``` text
Warning: would violate PodSecurity "baseline:latest":
host namespaces (hostNetwork=true, hostPID=true),
hostPath volumes (volumes "proc", "sys", "root")
```

This was a critical finding.

The Node Exporter configuration was **not Baseline compliant**.

Therefore, simply applying:

``` text
enforce=baseline
```

to the monitoring namespace could prevent future Node Exporter Pods from
being admitted.

------------------------------------------------------------------------

# 11. Restricted vs Baseline --- Observed Difference

  -----------------------------------------------------------------------------------
  Security Control                    Baseline Result         Restricted Result
  ----------------------------------- ----------------------- -----------------------
  `hostNetwork: true`                 Rejected                Rejected

  `hostPID: true`                     Rejected                Rejected

  `hostPath`                          Rejected                Rejected

  `allowPrivilegeEscalation: false`   Not reported in this    Required
                                      test                    

  Drop all capabilities               Not reported in this    Required
                                      test                    

  `runAsNonRoot: true`                Not reported for        Required
                                      default NGINX           

  Seccomp profile                     Not reported for        Required
                                      default NGINX           
  -----------------------------------------------------------------------------------

The key lesson is:

> Baseline is not simply Restricted with one or two fields removed. Each
> Pod Security Standard defines a specific set of controls.

------------------------------------------------------------------------

# 12. Why Existing Pods Do Not Immediately Die

PSA is primarily admission-time enforcement.

If a namespace receives a stricter label, existing Pods are not simply
terminated because they violate the new policy.

The important moment is when a new admission request occurs.

For example:

``` text
Existing node-exporter
        |
        | still running
        |
Node fails / rollout happens
        |
        v
DaemonSet creates replacement Pod
        |
        v
kube-apiserver
        |
        v
PSA enforce=baseline
        |
        v
hostNetwork / hostPID / hostPath
        |
        v
REJECT
```

This creates an operational trap: everything can appear healthy
immediately after enabling enforcement, but future rescheduling or
rollouts can fail.

------------------------------------------------------------------------

# 13. Controlled Baseline Enforcement Test

Instead of risking the monitoring namespace, a disposable namespace was
created:

``` bash
kubectl create namespace psa-baseline-test
```

Baseline enforcement was enabled:

``` bash
kubectl label namespace psa-baseline-test \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest
```

A normal NGINX Pod was accepted:

``` bash
kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test
```

A host-network Pod was rejected:

``` bash
kubectl run hostnetwork-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test \
  --overrides='{"spec":{"hostNetwork":true}}'
```

Result:

``` text
Error from server (Forbidden):
pods "hostnetwork-pod" is forbidden:
violates PodSecurity "baseline:latest":
host namespaces (hostNetwork=true)
```

This directly demonstrated `enforce`.

------------------------------------------------------------------------

# 14. Controlled Restricted Test

A second disposable namespace was configured for Restricted enforcement.

Example:

``` bash
kubectl create namespace psa-restricted-test

kubectl label namespace psa-restricted-test \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

A default NGINX Pod was then attempted:

``` bash
kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-restricted-test
```

It was rejected for four major reasons:

``` text
allowPrivilegeEscalation != false
unrestricted capabilities
runAsNonRoot != true
seccompProfile
```

------------------------------------------------------------------------

# 15. SecurityContext

`securityContext` controls security-related runtime settings for Pods
and containers.

Some settings are Pod-level:

``` yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
```

Other controls are commonly container-level:

``` yaml
containers:
- name: app
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
      - ALL
```

Container-level settings can provide per-container control inside a
multi-container Pod.

------------------------------------------------------------------------

# 16. `runAsNonRoot`

Example:

``` yaml
runAsNonRoot: true
```

This declares that the container must not execute as UID 0.

Restricted required this during the lab.

A stronger explicit configuration can include:

``` yaml
runAsNonRoot: true
runAsUser: 1000
runAsGroup: 1000
```

The difference is useful:

-   `runAsNonRoot: true` expresses the security requirement.
-   `runAsUser: 1000` explicitly selects the UID.
-   `runAsGroup: 1000` explicitly selects the primary GID.

------------------------------------------------------------------------

# 17. `allowPrivilegeEscalation`

Restricted required:

``` yaml
allowPrivilegeEscalation: false
```

At a Linux level, this is related to preventing a process from acquiring
additional privileges.

Conceptually:

``` text
Application process
      |
      | execute privileged mechanism
      v
Potential privilege gain
```

With privilege escalation disabled, Kubernetes/container runtime
configures the process so that it cannot gain additional privileges in
this manner.

This is an important defense after an attacker has already obtained code
execution inside a container.

------------------------------------------------------------------------

# 18. Linux Capabilities

Traditional Unix root has extremely broad power.

Linux capabilities divide portions of that power into smaller units.

Examples include:

``` text
CAP_CHOWN
CAP_DAC_OVERRIDE
CAP_NET_BIND_SERVICE
CAP_NET_ADMIN
CAP_SYS_ADMIN
CAP_SYS_PTRACE
```

A hardened workload should normally start with:

``` yaml
capabilities:
  drop:
  - ALL
```

Then add back only capabilities that are truly required and permitted by
the applicable policy.

Conceptually:

``` text
Broad privileges
      |
      v
DROP ALL
      |
      v
Add minimum required privilege
```

This implements the principle of least privilege.

------------------------------------------------------------------------

# 19. Seccomp

Restricted required a valid seccomp profile.

Example:

``` yaml
seccompProfile:
  type: RuntimeDefault
```

Seccomp restricts the Linux system calls available to a process.

Conceptually:

``` text
Application
    |
    | syscall
    v
+-----------+
| seccomp   |
+-----+-----+
      |
 +----+----+
 |         |
allow    block
 |         X
 v
Linux kernel
```

Because containers share the host kernel, reducing unnecessary syscall
exposure helps reduce kernel attack surface.

------------------------------------------------------------------------

# 20. First Restricted-Compliant NGINX Attempt

A hardened NGINX Pod was created with settings similar to:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-nginx
  namespace: psa-restricted-test

spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: nginx
    image: nginx:1.25

    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

PSA accepted the security configuration.

However, the application failed at runtime.

Logs:

``` text
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed
(13: Permission denied)
```

This teaches an extremely important distinction:

``` text
PSA compliant
     !=
Application operational
```

Admission security and application runtime compatibility are separate
layers.

------------------------------------------------------------------------

# 21. Why NGINX Failed

The container was forced to run as:

``` yaml
runAsUser: 1000
```

NGINX attempted to write under:

``` text
/var/cache/nginx
```

The non-root process did not have the required filesystem permissions.

The flow was:

``` text
PSA Restricted
      |
      v
PASS
      |
      v
Pod accepted
      |
      v
Container starts as UID 1000
      |
      v
NGINX mkdir(/var/cache/nginx/client_temp)
      |
      v
Permission denied
      |
      v
NGINX exits
```

This is a runtime filesystem problem, not a PSA rejection.

------------------------------------------------------------------------

# 22. `fsGroup`

A Pod can define:

``` yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
```

`fsGroup` is relevant to supported volume ownership and permissions.

It is particularly useful when a non-root application needs writable
mounted storage.

Conceptually:

``` text
Pod
|
+-- UID 1000
+-- primary GID 1000
+-- fsGroup 1000
|
+-- mounted volume
      |
      +-- writable to workload group
```

Do not interpret `fsGroup` as a universal fix for every filesystem path
in an image. Its behavior is associated with supported mounted volume
handling, not arbitrary ownership rewriting of the entire container
image filesystem.

------------------------------------------------------------------------

# 23. `emptyDir`

An `emptyDir` is temporary Pod-scoped storage.

Example:

``` yaml
volumes:
- name: cache
  emptyDir: {}
```

Mounted with:

``` yaml
volumeMounts:
- name: cache
  mountPath: /var/cache/nginx
```

Its lifetime is tied to the Pod rather than an individual container
restart.

For the hardened NGINX workload, writable `emptyDir` mounts can be
provided for paths that legitimately need runtime writes while the rest
of the container filesystem remains read-only.

Examples:

``` text
/tmp
/var/cache/nginx
/var/run
```

This creates a strong pattern:

``` text
Container root filesystem
        |
        +-- read-only
        |
        +-- explicit writable volume -> /tmp
        +-- explicit writable volume -> /var/cache/nginx
        +-- explicit writable volume -> /var/run
```

This is substantially better than making the entire root filesystem
writable merely because the application needs a few writable
directories.

------------------------------------------------------------------------

# 24. `readOnlyRootFilesystem`

A hardened container can use:

``` yaml
readOnlyRootFilesystem: true
```

This prevents arbitrary writes to the container's root filesystem.

Applications frequently need certain writable locations, so a production
design often combines:

``` text
readOnlyRootFilesystem=true
          +
explicit writable volumes
```

This reduces the writable attack surface available to a compromised
process.

------------------------------------------------------------------------

# 25. Running NGINX on Port 8080

The lab used an NGINX configuration similar to:

``` nginx
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
```

Why use `8080`?

A non-root process should avoid depending on privileged low-port
behavior unless there is a specific requirement and capability design.

Instead of granting additional privilege simply to bind a conventional
container port, running the process on an unprivileged port such as
`8080` is often simpler.

A Kubernetes Service can still expose port 80 externally while
forwarding to container port 8080.

For example:

``` yaml
ports:
- port: 80
  targetPort: 8080
```

Users can therefore access port 80 while the application process remains
non-root.

------------------------------------------------------------------------

# 26. Hardened NGINX Configuration

Create the NGINX configuration:

``` bash
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

Create the ConfigMap:

``` bash
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n psa-restricted-test
```

A hardened Pod can then use:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: psa-restricted-test

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

Apply:

``` bash
kubectl apply -f good-pod.yaml
```

Check:

``` bash
kubectl get pod good-pod -n psa-restricted-test
kubectl logs good-pod -n psa-restricted-test
```

------------------------------------------------------------------------

# 27. Why `kubectl get pods -w` Does Not Show PSA Admission Warnings

This was an important correction to the original practical workflow.

PSA warnings belong to the API response sent to the client making the
admission request.

For example:

``` text
kubectl rollout restart
        |
        | PATCH request
        v
kube-apiserver
        |
        v
Pod Security Admission
        |
        v
warning returned to kubectl
```

By contrast:

``` bash
kubectl get pods -w
```

watches resource events:

``` text
API server
   |
   +-- ADDED
   +-- MODIFIED
   +-- DELETED
         |
         v
kubectl watch
```

The watch client is not necessarily the client that submitted the
violating CREATE/PATCH request.

Therefore, expecting PSA admission warnings to appear as runtime Pod
events in `kubectl get pods -w` is conceptually incorrect.

------------------------------------------------------------------------

# 28. Why Randomly Deleting a Pod Is a Weak Test

A command such as:

``` bash
kubectl get pods -n monitoring -o name \
  | head -1 \
  | xargs kubectl delete -n monitoring
```

is not an ideal production troubleshooting method.

It arbitrarily selects the first Pod and may affect Prometheus,
Alertmanager, Grafana, or another component.

A controlled operation is preferable.

For Node Exporter:

``` bash
kubectl rollout restart daemonset \
  monitoring-prometheus-node-exporter \
  -n monitoring
```

This clearly identifies the workload being tested and makes the
experiment reproducible.

------------------------------------------------------------------------

# 29. PSA and Controllers

Kubernetes controllers complicate the mental model slightly.

A Deployment, StatefulSet, or DaemonSet owns a Pod template.

For example:

``` text
DaemonSet
   |
   | PodTemplate
   v
DaemonSet Controller
   |
   +--> Pod node-1
   +--> Pod node-2
   +--> Pod node-3
```

Security policy can affect workload changes and Pod admission.

A rollout can therefore reveal policy incompatibilities even when the
existing Pods were created before stricter policy was introduced.

This is why policy rollout must consider:

-   Deployments.
-   StatefulSets.
-   DaemonSets.
-   Jobs/CronJobs.
-   Operators.
-   Helm-managed workloads.
-   GitOps controllers.

------------------------------------------------------------------------

# 30. Production Namespace Strategy

A mature cluster should not assume one Pod Security level fits every
namespace.

A conceptual architecture is:

``` text
Cluster
|
+-- application namespaces
|      |
|      +-- Restricted where compatible
|
+-- ordinary infrastructure
|      |
|      +-- Baseline / Restricted where compatible
|
+-- privileged infrastructure
       |
       +-- CNI agents
       +-- CSI node plugins
       +-- node-level monitoring
       +-- security agents
              |
              +-- explicitly controlled exceptions
```

The objective is not to disable security because an infrastructure
component is trusted.

The objective is to make privileged access:

-   Necessary.
-   Explicit.
-   Minimal.
-   Isolated.
-   Auditable.
-   Difficult for ordinary workloads to obtain.

------------------------------------------------------------------------

# 31. PSA Exemptions

Kubernetes supports Pod Security Admission configuration with
exemptions.

Conceptually:

``` yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration

    defaults:
      enforce: baseline
      enforce-version: latest

    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system
```

Exemptions are powerful and must be used carefully.

Exempting an entire namespace means workloads admitted through that
exemption are outside the relevant PSA checks.

Therefore, access to create workloads in exempt namespaces becomes
security-sensitive.

------------------------------------------------------------------------

# 32. PSA vs OPA Gatekeeper

PSA provides predefined Pod Security Standards.

It is excellent for controls such as:

-   Baseline Pod hardening.
-   Restricted Pod hardening.
-   Namespace-level security posture.

But it is not a general arbitrary policy language.

OPA Gatekeeper can enforce organization-specific policies such as:

-   Require labels.
-   Reject `:latest`.
-   Restrict approved image registries.
-   Require resource requests/limits.
-   Enforce naming conventions.
-   Apply custom field-level rules.

Conceptually:

``` text
PSA
|
+-- predefined Kubernetes Pod Security Standards


Gatekeeper
|
+-- custom organization policy
    |
    +-- ConstraintTemplate
    +-- Constraint
    +-- admission webhook
    +-- audit
```

They are complementary rather than direct replacements in many
environments.

------------------------------------------------------------------------

# 33. Production Defense in Depth

PSA should not be treated as the entire Kubernetes security strategy.

A production security architecture can include:

``` text
Supply chain
   |
   +-- trusted registries
   +-- image scanning
   +-- signatures / provenance

Admission
   |
   +-- PSA
   +-- Gatekeeper / other policy engines

Runtime identity
   |
   +-- non-root
   +-- capabilities
   +-- seccomp
   +-- read-only filesystem

Kubernetes authorization
   |
   +-- RBAC
   +-- ServiceAccounts

Network
   |
   +-- NetworkPolicy
   +-- ingress/egress controls

Secrets
   |
   +-- least-privilege access
   +-- external secret management where appropriate

Detection
   |
   +-- audit logs
   +-- runtime security
   +-- monitoring
```

Defense in depth matters because no single layer can prevent every
attack.

------------------------------------------------------------------------

# 34. Troubleshooting Decision Tree

When a Pod fails, first identify **which layer failed**.

``` text
kubectl apply
     |
     +--> Forbidden?
     |       |
     |       +--> Admission policy
     |             PSA / Gatekeeper / webhook / RBAC
     |
     +--> Created?
             |
             +--> Pending?
             |      |
             |      +--> scheduling / PVC / resources / taints
             |
             +--> CrashLoopBackOff?
             |      |
             |      +--> application/runtime/config
             |
             +--> Permission denied?
                    |
                    +--> UID/GID
                    +--> filesystem permissions
                    +--> volume ownership
                    +--> capabilities
                    +--> read-only filesystem
```

For this lab:

``` text
Default NGINX + Restricted
        |
        v
Forbidden
        |
        +--> PSA problem


Hardened NGINX accepted
        |
        v
NGINX exits
        |
        +--> runtime filesystem permission problem
```

Knowing the failure layer prevents wasted troubleshooting.

------------------------------------------------------------------------

# 35. Useful Inspection Commands

## Namespace policy

``` bash
kubectl get namespace monitoring --show-labels
```

## DaemonSets

``` bash
kubectl get daemonset -n monitoring
```

## Deployments

``` bash
kubectl get deployment -n monitoring
```

## Inspect security-sensitive DaemonSet fields

``` bash
kubectl get ds monitoring-prometheus-node-exporter \
  -n monitoring \
  -o yaml | grep -A5 -B3 -E \
  'hostPath|hostNetwork|hostPID|privileged|runAsUser|allowPrivilegeEscalation'
```

## Check host namespaces

``` bash
kubectl get pod <node-exporter-pod> \
  -n monitoring \
  -o jsonpath='{.spec.hostNetwork}{"\n"}{.spec.hostPID}{"\n"}'
```

## List hostPath mounts

``` bash
kubectl get pod <node-exporter-pod> \
  -n monitoring \
  -o jsonpath='{range .spec.volumes[*]}{.name}{" => "}{.hostPath.path}{"\n"}{end}'
```

## Logs

``` bash
kubectl logs <pod> -n <namespace>
```

## Describe

``` bash
kubectl describe pod <pod> -n <namespace>
```

------------------------------------------------------------------------

# 36. Common Gotchas

## Gotcha 1 --- Non-root does not mean Restricted-compliant

Node Exporter already had:

``` yaml
runAsNonRoot: true
runAsUser: 65534
```

but still violated Restricted because it used:

-   `hostNetwork`.
-   `hostPID`.
-   `hostPath`.
-   Other missing hardening controls.

Security is the combination of controls, not one boolean.

## Gotcha 2 --- Existing Pods can hide enforcement problems

Existing workloads may remain healthy after a policy change.

The problem can appear later during:

-   Deployment rollout.
-   Node maintenance.
-   Node failure.
-   Autoscaling.
-   Pod eviction.
-   Helm upgrade.
-   GitOps reconciliation.

## Gotcha 3 --- Admission success does not imply application health

A Pod can satisfy PSA and then crash because the application cannot run
as non-root.

## Gotcha 4 --- Infrastructure namespaces need deliberate design

Do not blindly label `monitoring`, `kube-system`, or another
infrastructure namespace as Restricted.

Inventory its workloads first.

## Gotcha 5 --- `warn` is not `enforce`

Seeing:

``` text
Warning: would violate ...
```

does not mean Kubernetes rejected the resource.

Look at whether the API operation succeeded.

## Gotcha 6 --- `latest` policy version changes over time

Using:

``` text
*-version=latest
```

tracks the policy version associated with the running Kubernetes
release.

For controlled environments, version pinning may be preferable during
carefully managed upgrades so that security behavior does not
unexpectedly shift with a Kubernetes version change.

------------------------------------------------------------------------

# 37. Interview Questions

## Q1. What is Pod Security Admission?

Pod Security Admission is Kubernetes' built-in admission controller for
enforcing the predefined Pod Security Standards at namespace level.

The standards are Privileged, Baseline, and Restricted.

------------------------------------------------------------------------

## Q2. What is the difference between warn, audit, and enforce?

`warn` allows the request and returns a warning to the API client.

`audit` allows the request and records a policy violation in audit data
when audit logging is configured.

`enforce` rejects a violating admission request.

------------------------------------------------------------------------

## Q3. Why shouldn't you immediately enable Restricted enforcement cluster-wide?

Infrastructure components may legitimately require host access or other
settings that Restricted prohibits.

A safer migration is:

``` text
audit/warn -> inventory -> remediation/exceptions -> test -> enforce
```

------------------------------------------------------------------------

## Q4. Why did Node Exporter violate Baseline?

The observed DaemonSet used:

``` yaml
hostNetwork: true
hostPID: true
```

and hostPath volumes for:

``` text
/proc
/sys
/
```

The live Baseline admission warning rejected these host namespace and
hostPath requirements.

------------------------------------------------------------------------

## Q5. Why did a Restricted-compliant NGINX Pod still crash?

PSA only established that its Pod security settings satisfied the
admission policy.

NGINX then ran as UID 1000 and attempted to create:

``` text
/var/cache/nginx/client_temp
```

The process lacked filesystem permission.

This was an application/runtime permission failure, not an admission
failure.

------------------------------------------------------------------------

## Q6. Why use `emptyDir` with a read-only root filesystem?

Applications often need a small number of writable runtime directories.

`readOnlyRootFilesystem: true` protects the general container filesystem
while explicit writable volumes can be mounted only where necessary.

This follows least privilege.

------------------------------------------------------------------------

## Q7. What is the difference between `runAsNonRoot` and `runAsUser`?

`runAsNonRoot: true` expresses the requirement that the workload must
not run as UID 0.

`runAsUser: 1000` explicitly selects UID 1000.

They can be used together.

------------------------------------------------------------------------

## Q8. Why drop Linux capabilities?

Capabilities represent individual privileged kernel operations.

Dropping all unnecessary capabilities reduces what a compromised process
can do.

A strong default is:

``` yaml
capabilities:
  drop:
  - ALL
```

and then explicitly add only what is required and permitted.

------------------------------------------------------------------------

## Q9. What does seccomp protect?

Seccomp filters Linux system calls.

It reduces the kernel API exposed to a process, decreasing the attack
surface available after application compromise.

------------------------------------------------------------------------

## Q10. PSA vs Gatekeeper?

PSA provides standardized Pod security profiles.

Gatekeeper provides customizable policy-as-code for
organization-specific admission rules.

They can be used together.

------------------------------------------------------------------------

# 38. Senior-Level Scenario

**Scenario:** A platform team enables `enforce=baseline` on the
monitoring namespace. Everything initially looks healthy. Several days
later a worker node is replaced and Node Exporter disappears from the
new node.

A strong troubleshooting approach is:

1.  Check the DaemonSet desired/current/ready counts.
2.  Inspect DaemonSet and Replica/Pod-related events.
3.  Check whether Pod creation was rejected.
4.  Inspect namespace PSA labels.
5.  Compare the Pod template against Baseline.
6.  Identify `hostNetwork`, `hostPID`, and `hostPath`.
7.  Recognize that old Pods surviving the policy change did not prove
    the workload was compliant.
8.  Decide whether the workload can be hardened or requires an
    explicitly governed exception.
9.  Validate the chosen design using warn/audit before enforcement.

This tests the engineer's understanding of both Kubernetes controllers
and admission security.

------------------------------------------------------------------------

# 39. Recommended Lab Cleanup

After completing the experiments:

``` bash
kubectl delete namespace psa-baseline-test
kubectl delete namespace psa-restricted-test
```

Review the monitoring namespace labels before leaving the lab:

``` bash
kubectl get namespace monitoring --show-labels
```

If the monitoring namespace is being returned to its pre-lab state,
remove only the labels intentionally added during the exercise:

``` bash
kubectl label namespace monitoring \
  pod-security.kubernetes.io/audit- \
  pod-security.kubernetes.io/audit-version- \
  pod-security.kubernetes.io/warn- \
  pod-security.kubernetes.io/warn-version-
```

Always verify afterward:

``` bash
kubectl get namespace monitoring --show-labels
```

Do not remove security policy from a real production namespace merely as
generic cleanup; the final state should follow the platform's approved
security design.

------------------------------------------------------------------------

# 40. Final Mental Model

The entire exercise can be summarized as:

``` text
                   Kubernetes Security
                          |
                          v
                 Pod Security Admission
                          |
          +---------------+---------------+
          |               |               |
     Privileged        Baseline        Restricted
          |               |               |
       minimal        moderate          strong
    restrictions     hardening        hardening

                          |
                          v
               Enforcement behavior
                          |
             +------------+------------+
             |            |            |
            WARN         AUDIT       ENFORCE
             |            |            |
           allow        allow         reject
             |            |
          tell user    record

                          |
                          v
                 Application runtime
                          |
              +-----------+-----------+
              |                       |
         Pod accepted              Pod rejected
              |
              v
       Application starts
              |
        +-----+------+
        |            |
      works        crashes
                     |
                     v
            troubleshoot runtime
```

The most important production lesson is:

> **A secure Kubernetes platform is not created by blindly applying the
> strictest policy. It is created by understanding workload
> requirements, reducing privilege to the minimum necessary, testing
> policy effects before enforcement, isolating unavoidable exceptions,
> and applying defense in depth.**

------------------------------------------------------------------------

# 41. Practical Command Summary

``` bash
# Inspect monitoring namespace
kubectl get pods -n monitoring -o wide
kubectl get namespace monitoring --show-labels
kubectl get daemonset -n monitoring
kubectl get deployment -n monitoring

# Restricted audit/warn
kubectl label namespace monitoring \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest

kubectl label namespace monitoring \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# Controlled workload evaluation
kubectl rollout restart daemonset \
  monitoring-prometheus-node-exporter \
  -n monitoring

# Baseline warning experiment
kubectl label namespace monitoring \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest \
  --overwrite

kubectl rollout restart daemonset \
  monitoring-prometheus-node-exporter \
  -n monitoring

# Baseline enforcement lab
kubectl create namespace psa-baseline-test

kubectl label namespace psa-baseline-test \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test

kubectl run hostnetwork-pod \
  --image=nginx:1.25 \
  -n psa-baseline-test \
  --overrides='{"spec":{"hostNetwork":true}}'

# Restricted enforcement lab
kubectl create namespace psa-restricted-test

kubectl label namespace psa-restricted-test \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

kubectl run normal-pod \
  --image=nginx:1.25 \
  -n psa-restricted-test
```

------------------------------------------------------------------------

## Conclusion

This lab started as a Pod Security namespace-labeling exercise but
exposed several deeper production concepts:

-   Admission control happens before workload creation.
-   Baseline and Restricted solve different levels of Pod hardening.
-   Node-level agents often require deliberate security exceptions.
-   Warn and audit are valuable migration mechanisms.
-   Existing healthy Pods do not prove compatibility with newly
    configured enforcement.
-   SecurityContext maps Kubernetes configuration to Linux process
    security.
-   Non-root execution can expose filesystem assumptions in
    legacy/container images.
-   Read-only root filesystems should be paired with explicit writable
    storage.
-   Admission compliance and application health are separate
    troubleshooting domains.
-   Production security policy should be introduced progressively and
    validated against real workloads.

The correct engineering objective is not simply **"make PSA Restricted
pass."**

The objective is:

``` text
Understand the workload
        |
        v
Identify required privileges
        |
        v
Remove unnecessary privileges
        |
        v
Isolate unavoidable exceptions
        |
        v
Validate with warn/audit
        |
        v
Enforce safely
        |
        v
Continuously verify
```

That is the mindset expected from Kubernetes platform, SRE, DevOps, and
security engineers.

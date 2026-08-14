# Day 27 — Capstone Complete, Job Strategy & The Road Forward

*The final day. You've spent 30 days going from Kubernetes basics to building production-grade AI-powered infrastructure. Today we close the loop — complete the capstone, build your job strategy for European tech companies, run a final interview simulation, and map the road forward.*

## 🏆 Part 1: What You've Built — The Full Picture

```
Week 1 (Days 1-7)    CKAD Foundation
├── Cluster architecture + kubectl mastery
├── Pods: lifecycle, init containers, multi-container patterns
├── Workload controllers: Deployment, StatefulSet, DaemonSet, HPA
├── Networking: CNI, Services, Ingress + Gateway API
├── Config + Secrets + Storage: PV/PVC/StorageClass
├── Helm v3 + Kustomize
└── CKAD mock exam

Week 2 (Days 8-15)   CKA Administration
├── kubeadm bootstrap + cluster upgrade
├── RBAC + ServiceAccounts + cert auth + OIDC/IRSA
├── etcd backup/restore + certificate management
├── Troubleshooting: 10 break/fix scenarios
├── Observability: Prometheus + Grafana + Loki + Jaeger
├── GitOps: ArgoCD + Argo Rollouts
├── CKA mock exam
└── Advanced troubleshooting (DNS, iptables, control plane)

Week 3 (Days 16-20)  CKS Security
├── Pod Security Admission + OPA Gatekeeper
├── Falco runtime security + Trivy image scanning
├── Supply chain: Cosign + SBOM + image signing
├── Secrets at scale: Sealed Secrets + ESO + Vault
├── CKS mock exam
└── Advanced scheduling: affinity, topology, priority, VPA

Week 4 (Days 21-26)  Senior Engineer Depth
├── Custom controllers + Operators + CRDs
├── Multi-cluster: Cluster API + ArgoCD ApplicationSet
├── Istio service mesh: traffic management + mTLS + Gateway API
├── Multi-tenancy + admission webhooks + API machinery
├── K8s internals: API server lifecycle, etcd, scheduler
├── 50 interview questions + system design
└── GPU scheduling + ML workloads

Week 5 (Days 27-30)  AI/K8s Integration (Capstone)
├── KubeFlow Pipelines + distributed training
├── LLM inference: vLLM + KServe + Triton + LiteLLM
├── AI-powered cluster analyzer (Day 28)
├── Production operator: Slack + Prometheus + deduplication
└── Job strategy + final interview simulation (today)
```

## 🚀 Part 2: Capstone Final Feature — GitHub Actions CI/CD

*Add automated testing and deployment to your analyzer. This makes the capstone genuinely deployable.*

```
# .github/workflows/ci-cd.yaml
name: K8s Analyzer CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/k8s-analyzer

jobs:
  # ── Test ──────────────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: "3.11"
        cache: pip

    - name: Install dependencies
      run: pip install -r requirements.txt pytest pytest-asyncio

    - name: Run unit tests
      run: pytest tests/ -v --tb=short
      env:
        ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

  # ── Security scan ─────────────────────────────────────────────
  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - uses: actions/checkout@v4

    - name: Build image for scanning
      run: docker build -t ${{ env.IMAGE_NAME }}:scan .

    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.IMAGE_NAME }}:scan
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: 1

    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: trivy-results.sarif

    - name: Scan K8s manifests
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: config
        scan-ref: k8s/
        severity: HIGH,CRITICAL
        exit-code: 0             # warn only on manifest issues

  # ── Build and push ────────────────────────────────────────────
  build:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
    - uses: actions/checkout@v4

    - name: Log in to GHCR
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix={{branch}}-
          type=ref,event=branch
          type=semver,pattern={{version}}
          latest

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64

    - name: Sign image with Cosign
      env:
        COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      run: |
        cosign sign --key env://COSIGN_PRIVATE_KEY \
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  # ── Deploy to cluster ─────────────────────────────────────────
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/setup-kubectl@v3

    - name: Set kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig.yaml
        export KUBECONFIG=kubeconfig.yaml

    - name: Update image tag in manifests
      run: |
        IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main-${{ github.sha }}"
        sed -i "s|image:.*k8s-analyzer.*|image: ${IMAGE}|" \
          k8s/operator-deployment.yaml

    - name: Apply manifests
      run: |
        export KUBECONFIG=kubeconfig.yaml
        kubectl apply -f k8s/operator-deployment.yaml

    - name: Verify rollout
      run: |
        export KUBECONFIG=kubeconfig.yaml
        kubectl rollout status deployment/k8s-analyzer \
          -n k8s-analyzer \
          --timeout=120s

    - name: Commit updated manifests to GitOps repo
      run: |
        git config user.email "ci@yourcompany.com"
        git config user.name "CI Bot"
        git add k8s/operator-deployment.yaml
        git commit -m "deploy: k8s-analyzer@${{ github.sha }}"
        git push
```

**Unit tests for the analyzer**

```
# tests/test_detector.py
import pytest
from unittest.mock import MagicMock, patch
from datetime import datetime, timezone
from collector.pods import PodIssue
from detector.issues import detect_issues, Severity

def make_pod_issue(**kwargs):
    defaults = dict(
        namespace="test-ns",
        name="test-pod",
        status="Error",
        reason="CrashLoopBackOff",
        message="back-off restarting",
        restart_count=6,
        age_minutes=30,
        node="node-1",
        logs="Error: connection refused\n",
        events=["[Warning] BackOff: restarting failed container"]
    )
    defaults.update(kwargs)
    return PodIssue(**defaults)

class TestIssueDetection:
    def test_crashloop_detected_as_critical(self):
        pod = make_pod_issue(
            reason="CrashLoopBackOff",
            restart_count=10
        )
        issues = detect_issues([pod], [])
        assert len(issues) == 1
        assert issues[0].severity == Severity.CRITICAL

    def test_crashloop_low_restarts_is_warning(self):
        pod = make_pod_issue(
            reason="CrashLoopBackOff",
            restart_count=2
        )
        issues = detect_issues([pod], [])
        assert issues[0].severity == Severity.WARNING

    def test_image_pull_backoff_is_critical(self):
        pod = make_pod_issue(reason="ImagePullBackOff")
        issues = detect_issues([pod], [])
        assert issues[0].severity == Severity.CRITICAL
        assert "ImagePullBackOff" in issues[0].title

    def test_pending_old_pod_is_critical(self):
        pod = make_pod_issue(
            status="Pending",
            reason="PodNotScheduled",
            age_minutes=15
        )
        issues = detect_issues([pod], [])
        assert issues[0].severity == Severity.CRITICAL

    def test_no_issues_returns_empty(self):
        issues = detect_issues([], [])
        assert issues == []

    def test_issues_sorted_by_severity(self):
        pods = [
            make_pod_issue(
                name="pod-1",
                reason="HighRestartCount",
                restart_count=6
            ),
            make_pod_issue(
                name="pod-2",
                reason="ImagePullBackOff",
                restart_count=0
            ),
        ]
        issues = detect_issues(pods, [])
        assert issues[0].severity == Severity.CRITICAL   # ImagePullBackOff
        assert issues[1].severity == Severity.WARNING    # HighRestartCount

class TestAlertSuppressor:
    def test_new_issue_should_alert(self):
        from alerts.suppressor import AlertSuppressor
        from detector.issues import ClusterIssue, Severity
        suppressor = AlertSuppressor(cooldown_seconds=1800)
        issue = MagicMock(spec=ClusterIssue)
        issue.severity = Severity.CRITICAL
        issue.namespace = "test"
        issue.affected_resource = "pod/test-pod"
        issue.category = "Pod"
        should, reason = suppressor.should_alert(issue)
        assert should is True
        assert reason == "new_issue"

    def test_same_issue_suppressed_in_cooldown(self):
        from alerts.suppressor import AlertSuppressor
        from detector.issues import ClusterIssue, Severity
        suppressor = AlertSuppressor(cooldown_seconds=1800)
        issue = MagicMock(spec=ClusterIssue)
        issue.severity = Severity.CRITICAL
        issue.namespace = "test"
        issue.affected_resource = "pod/test-pod"
        issue.category = "Pod"
        suppressor.should_alert(issue)
        suppressor.mark_alerted(issue)
        should, reason = suppressor.should_alert(issue)
        assert should is False
        assert "suppressed" in reason

class TestAnalysisCache:
    def test_cache_hit(self):
        from analyzer.cache import AnalysisCache
        from detector.issues import ClusterIssue, Severity
        cache = AnalysisCache(ttl_seconds=3600)
        issue = MagicMock(spec=ClusterIssue)
        issue.namespace = "test"
        issue.affected_resource = "pod/test"
        issue.category = "Pod"
        issue.title = "CrashLoopBackOff: test-pod"
        issue.raw_data = {"logs": "error log content"}
        cache.set(issue, "cached analysis")
        result = cache.get(issue)
        assert result == "cached analysis"

    def test_cache_miss(self):
        from analyzer.cache import AnalysisCache
        from detector.issues import ClusterIssue, Severity
        cache = AnalysisCache(ttl_seconds=3600)
        issue = MagicMock(spec=ClusterIssue)
        issue.namespace = "test"
        issue.affected_resource = "pod/unknown"
        issue.category = "Pod"
        issue.title = "Unknown"
        issue.raw_data = {}
        result = cache.get(issue)
        assert result is None
```

## 🌍 Part 3: European Job Market Strategy

You've built the skills. Now build the strategy.

**Target companies by country**

**Germany**

```
SAP           — Berlin, Munich, Walldorf (K8s, cloud native)
Zalando       — Berlin (K8s at massive scale, open source)
HelloFresh    — Berlin (K8s, data engineering)
N26           — Berlin (fintech, K8s, strict security)
Delivery Hero — Berlin (platform engineering)
Siemens       — Munich (industrial K8s, edge)
BMW/Mercedes  — Munich (automotive K8s, AI)
```

**Netherlands**

```
Booking.com   — Amsterdam (massive scale, platform eng)
Adyen         — Amsterdam (fintech, SRE, K8s)
ASML          — Eindhoven (semiconductor, edge K8s)
Elastic       — Amsterdam (remote-first)
TomTom        — Amsterdam (maps, ML infra)
ING Bank      — Amsterdam (banking K8s)
```

**Ireland**

```
Google        — Dublin (GKE, cloud infra)
Meta          — Dublin (infra at scale)
Stripe        — Dublin (payments, K8s, reliability)
HubSpot       — Dublin (platform engineering)
Workday       — Dublin (HR tech, SRE)
LinkedIn      — Dublin (platform infra)
```

**Portugal (Lisbon/Porto)**

```
Farfetch      — Porto (fashion tech, K8s)
Revolut       — Lisbon (fintech, fast growth)
Feedzai       — Porto (ML, fraud detection)
Talkdesk      — Lisbon (AI, cloud native)
OutSystems    — Lisbon (low-code platform)
```

**Sweden**

```
Spotify       — Stockholm (massive K8s, data infra)
King          — Stockholm (gaming, platform eng)
Klarna        — Stockholm (fintech, microservices)
Ericsson      — Stockholm (telco cloud, K8s at edge)
Volvo Cars    — Gothenburg (automotive AI, K8s)
```

**Job titles to target**

```
Platform Engineer           ← your primary target
Site Reliability Engineer (SRE)
DevOps Engineer
Cloud Native Engineer
Infrastructure Engineer
Kubernetes Engineer
MLOps Engineer              ← with your AI/K8s week background
Staff Engineer (Infrastructure)
```

**Salary benchmarks (2024-2025)**

```
Germany:
  Senior DevOps/Platform:    €70,000 - €110,000
  Staff Engineer:            €110,000 - €150,000

Netherlands:
  Senior DevOps/Platform:    €75,000 - €115,000
  Staff Engineer:            €115,000 - €160,000

Ireland:
  Senior DevOps/Platform:    €80,000 - €120,000
  Senior @ Google/Meta:      €120,000 - €180,000+

Portugal:
  Senior DevOps/Platform:    €45,000 - €75,000
  Senior @ Revolut/Farfetch: €65,000 - €95,000

Sweden:
  Senior DevOps/Platform:    SEK 700,000 - 1,000,000 (~€60-90k)
  Senior @ Spotify/Klarna:   SEK 900,000 - 1,200,000
```

## 📋 Part 4: Your Application Strategy

**Resume — K8s-specific framing**

```
WRONG: "Worked with Kubernetes and Docker"
RIGHT: "Designed and operated production Kubernetes clusters 
        serving 50M daily requests across 3 AWS regions, 
        including etcd disaster recovery, 
        custom admission webhooks, and GitOps with ArgoCD"

WRONG: "Used Helm for deployments"
RIGHT: "Built Helm chart library used by 8 engineering teams, 
        reducing deployment time from 2 hours to 8 minutes 
        and eliminating environment drift across 
        dev/staging/production"

WRONG: "Implemented monitoring"
RIGHT: "Deployed kube-prometheus-stack with 150+ custom 
        recording rules and alerting policies, reducing 
        MTTR from 45 minutes to 8 minutes across 
        12 production services"
```

**Portfolio projects to highlight**

```
1. k8s-analyzer (Days 28-29)
   — AI-powered cluster analyzer with Slack alerts + Prometheus
   — Shows: K8s API, operator pattern, AI integration, prod engineering

2. URL shortener on K8s (DevOps curriculum)
   — FastAPI + Redis + Helm + ArgoCD + Terraform
   — Shows: full stack, GitOps, infrastructure as code

3. CKA/CKAD/CKS certifications
   — Scheduled and passed (register NOW while fresh)
   — Shows: validated, standardized knowledge

4. Contributions to open source K8s tooling
   — File a bug report or PR against a CNCF project
   — Shows: community participation, real-world engagement
```

**Where to apply**

```
Primary channels:
  LinkedIn        — set "Open to work" + Europe location
  relocate.me     — specifically for relocation-sponsored roles
  arbeitnow.com   — German tech jobs, many with visa sponsorship
  Glassdoor       — research salaries before interviews
  wellfound.com   — startups, often more open to remote/relocation

Company career pages directly:
  jobs.zalando.com
  careers.booking.com
  stripe.com/jobs
  greylock.com/job-board (backed startups)
  
Visa sponsorship research:
  ind.nl/en         (Netherlands — check public register)
  make-it-in-germany.com
  euworkpermit.com
```

**Visa pathways**

```
Germany:
  EU Blue Card     ← your primary target
  Requires: job offer + relevant degree + €58,400 minimum salary
  Timeline: 4-8 weeks after offer

Netherlands:
  Highly Skilled Migrant (HSM / Kennismigrant)
  Requires: job offer from IND-recognized sponsor + €5,008/month (2024)
  Timeline: 2-4 weeks (fastest in Europe)
  Check: ind.nl/en/public-register-recognised-sponsors

Ireland:
  Critical Skills Employment Permit
  Requires: job offer + €32,000+ salary (IT roles: €32k+)
  Timeline: 6-8 weeks
  Renews to permanent residency in 2 years

Portugal:
  Tech Visa (D3) or Job Seeker Visa (D10)
  Timeline: 2-4 months, more bureaucracy
  
Sweden:
  Work permit via employer sponsorship
  Timeline: 1-4 months
```

## 💼 Part 5: Final Interview Simulation

This is a complete 45-minute senior K8s interview. Answer each section before reading the expected answer.

**Round 1: System Design (15 minutes)**

**Question**: Design the Kubernetes infrastructure for a fintech company launching in Europe. Requirements:

- 99.99% uptime SLA for payment processing API
- Must comply with GDPR (EU data stays in EU)
- 50 microservices, 10 ML models serving real-time fraud detection
- 100,000 requests/second peak load
- Zero-downtime deployments required
- Regulatory requirement: full audit trail of all cluster operations

**Expected answer structure:**

```
Multi-cluster architecture:
  - 2 active clusters: eu-west-1 (Ireland) + eu-central-1 (Frankfurt)
  - Active-active with Route53 latency routing
  - Data residency: all PVs, databases, secrets in EU regions only
  - Separate cluster for ML workloads (GPU nodes, different scaling patterns)

Control plane:
  - EKS managed control plane (AWS handles etcd, API server HA)
  - Or self-managed: 5 etcd nodes across 3 AZs, 3 API server replicas
  - etcd backups to S3 (same region) every 15 minutes
  - Cert rotation automated via cert-manager

Workload isolation:
  - Namespace per team with ResourceQuota + LimitRange
  - PSA: restricted on payment namespaces, baseline on others
  - OPA Gatekeeper: enforce image signing, required labels, no :latest
  - Separate node pools: payment (Guaranteed QoS, dedicated nodes),
    ML (GPU nodes, spot for training), general (mixed)

Networking:
  - Cilium CNI (eBPF, NetworkPolicy enforcement, mTLS-ready)
  - Istio service mesh: mTLS between all payment services
  - NetworkPolicy: default-deny, explicit allow per service
  - AWS Load Balancer Controller: NLB for payment API (UDP support)
  - Gateway API for ingress (Ingress deprecated)

Zero-downtime deployments:
  - Argo Rollouts: canary with analysis (error rate < 0.1%)
  - PodDisruptionBudget: minAvailable=N-1 on all critical services
  - maxSurge:1 maxUnavailable:0 on all Deployments
  - Readiness gates for Istio sidecar

ML serving:
  - KServe for model serving with canary
  - KEDA: scale to zero at night, up on demand
  - GPU node pool: A100 on EKS, MIG for multi-model serving
  - Triton: multiple models per GPU

Audit trail:
  - API server audit logging: RequestResponse level for all writes
  - Audit logs shipped to CloudTrail + SIEM
  - Falco: runtime security events
  - All audit logs immutable in S3 with Object Lock

GitOps:
  - ArgoCD: one instance per cluster, ApplicationSet for multi-cluster
  - All changes via Git PR + approval, no direct kubectl in prod
  - Sealed Secrets or ESO (AWS Secrets Manager) — no secrets in Git
```

**Round 2: Troubleshooting (10 minutes)**

**Scenario**: Production payment API suddenly shows 30% error rate at 2am. Your PagerDuty fires. Walk me through your investigation in the first 10 minutes.

**Expected answer (timed, in order):**

```
0:00 — Check the alert: what metric triggered? error rate? latency? 
       Which service? Which namespace? Which cluster?

0:30 — kubectl get pods -n payments
       Look for: CrashLoopBackOff, Pending, high restart counts
       kubectl get events -n payments --sort-by=lastTimestamp | tail -20

1:00 — Check if it's a deployment issue
       kubectl rollout history deployment/payment-api -n payments
       kubectl rollout status deployment/payment-api -n payments
       Was there a deploy at 2am? 'kubectl rollout undo' if yes.

2:00 — Check Grafana: Istio golden signals
       - Request rate: is traffic up or same?
       - Error rate breakdown by status code (503? 500? 429?)
       - P99 latency: spiking or same?
       - Which upstream service is returning errors?

3:00 — Check Istio: is this a mesh issue?
       kubectl exec payment-api-pod -c istio-proxy -- \
         pilot-agent request GET stats | grep error
       istioctl proxy-config endpoints payment-api-pod | grep EJECTED

4:00 — Check dependencies: Redis, Postgres, downstream services
       kubectl get pods -n databases
       kubectl exec payment-api-pod -- nc -zv redis-svc 6379

5:00 — Check node health:
       kubectl get nodes
       kubectl top nodes (is any node at 100% CPU/memory?)
       kubectl describe node <problem-node> | grep Conditions -A 10

6:00 — Check Kubernetes events cluster-wide:
       kubectl get events -A --field-selector=type=Warning \
         --sort-by=lastTimestamp | tail -30

8:00 — If still unclear: check logs
       kubectl logs -l app=payment-api -n payments \
         --tail=100 --since=10m

10:00 — Communicate: post to #incidents channel with what you know,
        what you've ruled out, and next steps
        Never go silent during an incident
```

**Round 3: Deep Technical (10 minutes)**

Q: You have 500 microservices. Every time a developer pushes code, all 500 services run their integration tests against a shared staging cluster. The cluster is overwhelmed. How do you fix this?

**Expected answer:**

```
Root problem: shared staging cluster becomes a bottleneck with 500 teams.

Solution 1 — Ephemeral preview environments:
  Each PR spins up its own namespace (preview-pr-1234) with just 
  the changed service + mocked or lightweight dependencies.
  Namespace is destroyed when PR merges.
  Tools: Argo CD ApplicationSet + PR-based generators,
         Terraform workspace per PR, or vCluster.
  Result: 500 PRs = 500 isolated namespaces, no contention.

Solution 2 — vCluster (virtual clusters):
  Run lightweight virtual K8s clusters inside the staging cluster.
  Each team/PR gets its own vCluster — own API server, own scheduler,
  shares host cluster's nodes and network.
  Isolation without the cost of real clusters.
  50ms cluster creation time vs 15 minutes for a real EKS cluster.

Solution 3 — Smart scheduling with priority:
  Integration tests run as K8s Jobs with low priority class.
  CI system queues them and submits in batches.
  KEDA scales the test node pool based on pending job count.
  Preemption ensures critical tests (main branch) run before PR tests.

Solution 4 — Optimize the tests:
  Contract testing (Pact) instead of full integration tests for 
  most services — only the changed service runs real integration.
  Test duration SLO: any test > 10 minutes is flagged for optimization.
  Parallel test execution within each test suite.

Recommendation: vCluster + ephemeral namespaces + 
  dedicated low-priority test node pool with KEDA autoscaling.
```

**Round 4: Behavioral (10 minutes)**

*Q1: Tell me about a time you had to make a hard infrastructure decision that affected other teams.*

**Structure: Situation → Decision needed → Stakeholders affected → How you decided → Outcome → What you'd do differently**

Use your 35-day curriculum as material: Helm chart standardization across teams, RBAC model changes, observability stack adoption, NetworkPolicy enforcement that broke things.

*Q2: How do you stay current with Kubernetes?*

**Strong answer covers:**

```
Active learning:
  - Building things (this 30-day curriculum)
  - Reading K8s changelogs for each minor release
  - Following specific SIGs (sig-network, sig-security, sig-storage)
  - KubeCon talks on YouTube (filter by year)

Community:
  - K8s Slack workspace
  - CNCF landscape tracking
  - Twitter/X K8s community (search #kubernetes)

Certifications:
  - CKA/CKAD/CKS renewal every 2 years forces review
  - Maintaining skills > achieving certification

Practical:
  - Contributing to open source (filing issues, small PRs)
  - Building tools that solve real problems (like your analyzer)
```

**Round 5: Questions to Ask Them (5 minutes)**

These questions show senior thinking and help you evaluate if this is a good team:

```
On engineering culture:
"How does your team handle post-mortems — 
 are they blameless and what comes out of them?"

"What does your deployment pipeline look like — 
 how long from merge to production?"

On technical depth:
"What's the most interesting Kubernetes problem 
 you've solved in the last 6 months?"

"How do you handle the upgrade lifecycle — 
 what version are you on and how far behind are you?"

On team:
"What does career growth look like for platform engineers here — 
 what does a Staff Engineer do differently than a Senior?"

On the role:
"What would success look like in the first 90 days?"

"What are the biggest pain points the team is trying to solve 
 that this role would own?"
```

## 🗓️ Part 6: Your 90-Day Post-Curriculum Plan

```
Days 1-30 (done ✅): Build the knowledge

Days 31-60: Certify + Apply
  Week 5-6:  Schedule CKA exam (book NOW — do it while fresh)
  Week 6-7:  Schedule CKAD exam
  Week 7-8:  Schedule CKS exam
  Week 5-8:  Polish resume with K8s-specific framing
  Week 6-8:  Upload k8s-analyzer to GitHub with good README
  Week 7-8:  Start applications: 20 companies, 3 countries
  Week 8:    LinkedIn: "Open to work", add K8s skills section

Days 61-90: Interview + Negotiate
  Week 9-10: First-round interviews (phone screens)
  Week 10-11: Technical interviews (use this curriculum)
  Week 11-12: System design rounds
  Week 12:   Offers + negotiation
  
Certification exam order:
  1. CKAD first (broadest, most job-relevant)
  2. CKA second (deeper, shows admin knowledge)
  3. CKS third (differentiator, few people have it)
  
Exam booking:
  cncf.io/certification → use code KUBERNETES30 
  (check current discount codes — CNCF often runs promotions)
  
GitHub portfolio checklist:
  ✅ k8s-analyzer (Days 28-29) — with README, CI badge, Helm chart
  ✅ url-shortener-k8s (DevOps curriculum) — with architecture diagram
  ✅ gitops-demo (ArgoCD) — shows GitOps workflow
  □  Blog post: "Building an AI-powered K8s cluster analyzer"
     → Post on dev.to, Hashnode, Medium
     → Share in K8s Slack, LinkedIn
     → This gets recruiter attention
```

## 🧭 Part 7: The Road Beyond

**What's next after certification**

```
Level 1 (you are here): CKA + CKAD + CKS certified, Senior Engineer
  Focus: get the job, prove yourself in production

Level 2 (6-12 months): Operate at scale, build internal platform
  Focus: multi-cluster, cost optimization, platform team leadership

Level 3 (1-2 years): Staff/Principal Engineer or Architecture
  Focus: technical strategy, cross-team influence, system design

Level 4 (2-3 years): Distinguished Engineer or Engineering Manager
  Focus: org-level impact, hiring, technical vision
```

**Technologies to add next**

```
Immediate value-add:
  Terraform + AWS (if not done) — IaC for the infra K8s runs on
  Python/Go — write your own operators (Go is the K8s lingua franca)
  eBPF — Cilium, Falco internals, the future of K8s networking

6-12 months:
  Platform engineering: Backstage (IDP), Crossplane (K8s for everything)
  Cost optimization: Kubecost, OpenCost, spot instance management
  WASM on K8s: Wasmtime, SpinKube — the next frontier after containers

1-2 years:
  AI infrastructure: Ray, Kubeflow, NVIDIA NIM, LLM fine-tuning pipelines
  Multi-cloud: Cluster API, K8s federation, cloud-agnostic networking
  FinOps: K8s cost allocation, chargeback, rightsizing automation
```

**K8s community to join**

```
CNCF Slack: cloud-native.slack.com
  #kubernetes-novice, #kubernetes-dev, #sig-network, #sig-security

Kubernetes SIGs (Special Interest Groups):
  Attend SIG meetings (public, on YouTube)
  Read SIG meeting notes (github.com/kubernetes/community)
  File issues or review PRs — even reading is educational

KubeCon (your target: attend one in Europe)
  KubeCon EU: Amsterdam, Paris, London rotation
  CloudNativeCon: co-located, broader CNCF ecosystem
  Schedule: March/April each year
  CFP (Call for Papers): submit a talk about your analyzer!
```

**🏁 Final Assessment**

Rate yourself honestly on each area. Use this for interview preparation.

```
Core K8s (CKAD):
  □ Pods, controllers, services           1 2 3 4 5
  □ Networking: CNI, NetworkPolicy        1 2 3 4 5
  □ Storage: PV/PVC/StorageClass          1 2 3 4 5
  □ Config: ConfigMap, Secrets            1 2 3 4 5
  □ Helm + Kustomize                      1 2 3 4 5

Administration (CKA):
  □ Cluster setup + upgrade               1 2 3 4 5
  □ RBAC + certificate management         1 2 3 4 5
  □ etcd backup + restore                 1 2 3 4 5
  □ Troubleshooting methodology           1 2 3 4 5
  □ Observability stack                   1 2 3 4 5

Security (CKS):
  □ PSA + OPA Gatekeeper + Kyverno        1 2 3 4 5
  □ Falco + runtime security              1 2 3 4 5
  □ Image scanning + supply chain         1 2 3 4 5
  □ Secrets management (ESO + Vault)      1 2 3 4 5

Senior depth:
  □ Operators + CRDs                      1 2 3 4 5
  □ Service mesh (Istio)                  1 2 3 4 5
  □ Multi-cluster + GitOps                1 2 3 4 5
  □ K8s internals (API server, etcd)      1 2 3 4 5
  □ Admission webhooks                    1 2 3 4 5

AI/ML:
  □ GPU scheduling                        1 2 3 4 5
  □ LLM inference (vLLM, KServe)          1 2 3 4 5
  □ AI-powered tooling                    1 2 3 4 5

Score: anything below 4 = spend another week there before interviews
```




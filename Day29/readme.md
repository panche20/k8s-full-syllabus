# Day Roadmap: K8s Observability Masterclass (1-Day Intensive)

Your cluster has one gap that most tutorials ignore: HAProxy sits outside the cluster, so it needs its own exporter, and your etcd is stacked on the 3 masters (not external), so etcd health directly = control-plane health. We'll build monitoring for both, not just the pods inside K8s.

<img width="883" height="585" alt="image" src="https://github.com/user-attachments/assets/6bcd1109-854d-4be4-89ae-cf0a2c75a07e" />

*Each module below follows your framework in full before we touch the cluster. Let's start with M0.*

## Module 0: The Three Pillars — Why Observability ≠ Monitoring

**The 'Why' (The Problem)**

Traditional monitoring (Nagios-era) answered one question: is this host up? It worked when infrastructure was static — a few dozen servers, long-lived, predictable failure modes. It falls apart in your world because:

- Cardinality explosion: with 3 masters + 2 workers + N pods that get rescheduled constantly, "is host X up" is meaningless — pods don't have stable identities.
- Unknown-unknowns: monitoring answers questions you thought to ask in advance (predefined checks). Observability answers questions you didn't know you'd need to ask — e.g., "why did p99 latency spike only for requests routed through master-2's apiserver during the etcd leader election at 03:14?" You can't pre-build a check for that; you need raw, high-cardinality data you can slice after the fact.
- Distributed failure is non-linear: in your topology, HAProxy could be healthy, 2 of 3 masters could be healthy, but if etcd loses quorum, the whole control plane goes read-only — no single host check catches this. You need to correlate metrics across components.

**Deep-Dive Mechanics**

Observability rests on three pillars, each answering a different question:

- Metrics (Prometheus) — numeric time series, cheap to store, great for trends/alerting/aggregation. Answers "what" and "how much."
- Logs (Loki) — discrete events with full context. Answers "what exactly happened."
- Traces (OTel/Tempo) — causal chains across service boundaries. Answers "where in the request path did it break."

The mechanical distinction that matters for interviews: metrics are pre-aggregated at write time (a counter increments, you lose the individual event), logs are stored raw (every event, expensive), traces are sampled and correlated by ID (a trace_id ties spans across services). This is why you use all three — each trades detail for cost differently.

**The Alternative Landscape**

<img width="901" height="451" alt="image" src="https://github.com/user-attachments/assets/051b35c7-28df-451e-979f-975c079fbf05" />

<img width="872" height="377" alt="image" src="https://github.com/user-attachments/assets/1b6c9b4b-5cb6-41b2-9d99-382114390a48" />

You're choosing Prometheus/Grafana/Loki because it's what Booking.com, Adyen, and most European scale-ups actually run internally (Datadog is common but expensive at their scale, so OSS + in-house is the senior-engineer differentiator).

**Interview POV & Edge Cases**

- Classic system design question: "Design a monitoring system for a 3-master, N-worker Kubernetes cluster." Interviewers want you to say: pull vs push model, why Prometheus scrapes rather than receives, how service discovery finds targets dynamically, and how you'd federate/scale it beyond one Prometheus instance.
- Gotcha senior engineers are expected to know: Prometheus's local storage is not built for HA or long-term retention — a single Prometheus instance is a single point of failure for your monitoring itself. If asked "what happens if your Prometheus pod dies," the wrong answer is "we lose nothing." The right answer involves remote_write to long-term storage or a Thanos sidecar.
- Another gotcha specific to your setup: kubeadm binds many control-plane metrics endpoints to 127.0.0.1 by default (scheduler, controller-manager, and depending on version, etcd). If you don't know this, your Module 1 install will show "targets down" and you'll waste an hour debugging network policy when it's actually a bind-address flag in a static pod manifest.


**The 'Better Way' (Evolution)**

The modern trajectory is OpenTelemetry (OTel) unifying all three pillars under one collector/SDK/protocol, and eBPF-based tools (Pixie, Cilium Hubble) that get metrics/traces without any code instrumentation — reading straight from the kernel. We'll touch this in Module 7, but the fundamentals you're about to learn (pull model, label cardinality, PromQL) transfer directly even if the collection layer evolves.

## Module 1: Prometheus Deep-Dive + Install

**The 'Why' (The Problem)**

Before Prometheus (2012, born from Google's internal Borgmon), the monitoring landscape was:

- Graphite/StatsD: push-based, metrics named as flat dot-separated strings (app.host1.useast.requests). No labels — to slice by region AND host AND status code, you baked all of it into the metric name, causing name explosion and making ad-hoc queries nearly impossible.
- Nagios/SNMP-style checks: binary up/down checks, no real time-series query language, no way to ask "what's the 99th percentile latency over the last 5 minutes, grouped by pod."
- Push-based agents: every host had to know where to push data, and the monitoring server couldn't tell the difference between "app is down" and "app forgot to push."

The specific pain that forced Prometheus into existence: container orchestration made hosts ephemeral. In your cluster, pods get rescheduled, IPs change, and worker nodes can be replaced. A push-based system with a static list of destinations breaks immediately. You need a system that discovers what to monitor dynamically and pulls from it — which is exactly the architectural bet Prometheus made.

**Deep-Dive Mechanics**

**Pull model over HTTP.** Prometheus scrapes /metrics endpoints on a timer (scrape_interval, typically 15–30s). Every scrape target exposes plain-text metrics in the exposition format: metric_name{label="value"} 42. The pull model means Prometheus itself knows immediately if a target is unreachable (up == 0) — that absence is the signal, no separate heartbeat needed.

**Service discovery + relabeling.** For Kubernetes, kubernetes_sd_configs watches the API server for pods/services/endpoints/nodes and auto-generates scrape targets as they appear/disappear — this is what solves the ephemeral-IP problem from above. relabel_configs then filter/rewrite these targets before scraping (e.g., "only scrape pods with annotation prometheus.io/scrape: true").

**The four metric types** — this is the single most interview-tested Prometheus concept:

- Counter: monotonically increasing (e.g., total HTTP requests). You never read it raw — you apply rate() over a range vector to get per-second velocity.
- Gauge: goes up/down (e.g., current memory usage).
- Histogram: buckets observations into cumulative le (less-than-or-equal) buckets server-side. Aggregatable across instances (sum the buckets, then compute quantiles).
- Summary: computes quantiles client-side per-instance. Cannot be aggregated across instances — a p99 average across 5 pods is mathematically meaningless. This is why histograms are almost always preferred in distributed systems.

**TSDB internals.** Recent samples live in an in-memory HEAD block (~2hr window) backed by a Write-Ahead Log for crash recovery. Older data compacts into immutable on-disk blocks using Gorilla-style delta+XOR compression (this is why Prometheus storage is so space-efficient compared to naive float storage). Retention is time-based deletion of old blocks — there's no update/delete of individual samples.

**Counter resets.** When a pod restarts, its counter resets to 0. rate() is specifically designed to detect and correct for this (it looks for decreases and treats them as resets) — irate() doesn't smooth as aggressively and is more sensitive to spikes. Knowing when to use which is a real gotcha (see below).

**The Alternative Landscape**

<img width="877" height="525" alt="image" src="https://github.com/user-attachments/assets/45c6750d-154d-4444-9621-5c360b8cdbcf" />

<img width="877" height="351" alt="image" src="https://github.com/user-attachments/assets/a327c0fd-833e-49ed-8394-8e5f3d6774f6" />

Since you've already built CloudWatch + composite alarms, the mental model transfers, but the key difference interviewers probe for: CloudWatch is push-based and per-metric-priced (cost scales with cardinality directly), while Prometheus is pull-based and compute-priced (cost scales with your own infra, not per-metric). That's a real architectural trade-off, not just a tooling preference.

**Interview POV & Edge Cases**

**Common system-design framing:** "How does Prometheus discover targets in a dynamic environment, and what happens when a target disappears mid-scrape?" — expects: SD mechanism + up metric + staleness markers (Prometheus marks a series stale after ~5 min of no new samples, so dashboards don't show flatlined-but-actually-dead data as if it's still live).

**Gotchas senior engineers are expected to know:**

- **Cardinality explosion:** never put unbounded values (user IDs, request paths, IPs) as label values — each unique label combination is a new time series, and this is the #1 cause of Prometheus OOMs in production.
- **rate() vs irate():** rate() for alerting/dashboards (smooths noise), irate() only for high-resolution graphing of fast-changing counters — using irate() in an alert rule causes flapping.
- **Single point of failure:** one Prometheus server scraping everything means monitoring itself has no HA — this is a classic "what's wrong with this design" interview trap.
- **Your specific cluster gotcha:** kube-scheduler and kube-controller-manager bind their metrics port to 127.0.0.1 by default (hardened since ~v1.22 for security). You'll need to edit the static pod manifests on all 3 masters (/etc/kubernetes/manifests/kube-scheduler.yaml and kube-controller-manager.yaml) to change --bind-address=127.0.0.1 → 0.0.0.0, or Prometheus targets will show as down and it looks like a networking bug.
- **etcd metrics port:** since etcd is stacked on your masters, metrics are exposed on port 2381 (metrics-only, typically no auth) — don't confuse this with 2379 (client API port, requires mTLS certs). Scraping the wrong port is a very common time-sink.

**The 'Better Way' (Evolution)**

- **Managed Prometheus** (Amazon Managed Service for Prometheus, Grafana Cloud) offloads storage/HA — good for interviews where cost-of-ops is discussed.
- **Thanos / Mimir / VictoriaMetrics:** add global query view, long-term object-storage-backed retention, and HA (multiple Prometheus replicas deduplicated) — this is what "Prometheus at scale" actually means in production, and is a strong answer to the SPOF gotcha above.
- **eBPF-based (Cilium Hubble, Pixie):** derives golden-signal metrics straight from the kernel/network layer with zero app instrumentation — the bleeding edge, but not a full replacement yet.

**Hands-On Lab: Installing Prometheus on Your Cluster**

**Step 1 — Fix control-plane bind addresses (run on all 3 masters):**

```
sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
  /etc/kubernetes/manifests/kube-scheduler.yaml \
  /etc/kubernetes/manifests/kube-controller-manager.yaml
# kubelet watches this directory — pods restart automatically within ~20s
```

**Step 2 — Install kube-prometheus-stack via Helm:**

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

**Step 3 — Add HAProxy as an external scrape target**. On the HAProxy EC2 instance, install haproxy_exporter (port 9101 by default), enable HAProxy's stats socket, then add this to your Helm values and upgrade:

```
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'haproxy-external'
        static_configs:
          - targets: ['<haproxy-ec2-private-ip>:9101']
      - job_name: 'etcd'
        scheme: https
        tls_config:
          ca_file: /etc/prometheus/secrets/etcd-certs/ca.crt
          cert_file: /etc/prometheus/secrets/etcd-certs/healthcheck-client.crt
          key_file: /etc/prometheus/secrets/etcd-certs/healthcheck-client.key
        static_configs:
          - targets: ['<master1-ip>:2381','<master2-ip>:2381','<master3-ip>:2381']
```

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f values.yaml
```

**Step 4 — Verify:**

```
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090
# open localhost:9090/targets — confirm etcd, scheduler, controller-manager, haproxy-external all show "UP"
```

*Quick correction first: my etcd scrape config in the previous message was overcomplicated. The metrics port (2381) is plain HTTP by design — separate from the client port (2379) specifically so you don't need client certs to scrape it. It just needs the same bind-address fix as Step 1. I'll correct that below alongside the HAProxy setup.*

**Part A: Fix etcd metrics exposure (all 3 masters)**

```
# Check current bind address
sudo grep listen-metrics-urls /etc/kubernetes/manifests/etcd.yaml
# Typically: --listen-metrics-urls=http://127.0.0.1:2381

# Fix it — kubelet auto-restarts the static pod on manifest change
sudo sed -i 's#--listen-metrics-urls=http://127.0.0.1:2381#--listen-metrics-urls=http://0.0.0.0:2381#' \
  /etc/kubernetes/manifests/etcd.yaml
```

Corrected scrape config (no TLS block needed):

```
- job_name: 'etcd'
  metrics_path: /metrics
  static_configs:
    - targets: ['<master1-ip>:2381','<master2-ip>:2381','<master3-ip>:2381']
```

**Part B: Expose HAProxy metrics**

The standalone haproxy_exporter binary is now deprecated — upstream Prometheus explicitly says to use HAProxy's built-in exporter (PROMEX) instead. First check if your install has it:

```
haproxy -vv | grep -A3 "Available services"
```

**Path A — if it says prometheus-exporter** (most HAProxy 2.4+ builds): just edit /etc/haproxy/haproxy.cfg:

```
frontend stats
    bind *:8404
    http-request use-service prometheus-exporter if { path /metrics }
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:<strong-password>
```

```
sudo systemctl reload haproxy
curl http://localhost:8404/metrics | head    # sanity check
```

**Path B — if it says Available services** : none (common on the default Ubuntu apt package): use Ubuntu's packaged exporter instead of hand-installing a GitHub binary:

```
sudo apt update && sudo apt install -y prometheus-haproxy-exporter
```

Enable the stats page in haproxy.cfg (same block as above, minus the use-service line), then point the exporter at it:

```
echo 'ARGS="--haproxy.scrape-uri=http://admin:<strong-password>@localhost:8404/stats;csv"' | \
  sudo tee -a /etc/default/prometheus-haproxy-exporter
sudo systemctl restart prometheus-haproxy-exporter
curl http://localhost:9101/metrics | head
```

**Part C: Networking**

On the HAProxy EC2 instance's security group, add an inbound rule for the port you used (8404 for Path A, 9101 for Path B), source = your worker nodes' security group (not 0.0.0.0/0 — Prometheus runs from inside the VPC).

**Part D: Update Prometheus scrape config**

```
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'haproxy-external'
        static_configs:
          - targets: ['<haproxy-ec2-private-ip>:8404']   # or :9101 if Path B
      - job_name: 'etcd'
        metrics_path: /metrics
        static_configs:
          - targets: ['<master1-ip>:2381','<master2-ip>:2381','<master3-ip>:2381']
```

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090
# localhost:9090/targets → confirm etcd (3/3 up) and haproxy-external (up)
```

## Module 2: Grafana + Dashboards-as-Code

**The 'Why' (The Problem)**

Prometheus's built-in expression browser can plot a single query, but it has no persistence, no templating across hosts, no way to overlay multiple data sources on one screen, and no sharing/permissions model. Before tools like Grafana existed, ops teams relied on single-backend tools (Cacti, Graphite-web) — tied to one data source, with weak query flexibility and no way to correlate signals.

The specific pain in your setup: you now have Prometheus scraping etcd, HAProxy, node-exporter, and kube-state-metrics — four different signal sources. During an incident ("why did requests fail at 14:32?"), you need to overlay HAProxy backend status, node CPU, and (once we add it) nginx error logs on one screen, not tab between four separate query consoles. That correlation-in-one-pane-of-glass is the actual product Grafana sells — the graphing is almost secondary.

**Deep-Dive Mechanics**

**Architecture**: Grafana is a stateless web app + a small backend store (SQLite by default, or Postgres/MySQL for HA) that holds dashboard JSON, users, orgs, and alert rule definitions. Critically, Grafana stores none of your metrics — every panel render is a live query proxied to the underlying data source (Prometheus, Loki, etc.) at view time. This means Grafana going down doesn't lose historical data; it just blinds you to it temporarily.

**Dashboards are JSON documents**. Every dashboard — panels, queries ("targets"), layout, templating variables — is one JSON blob. This is the entire basis of "dashboards as code": since it's just JSON, it can live in Git, get generated programmatically, and be code-reviewed like any other infra artifact.

**Provisioning (the part that matters for you)**. Instead of clicking through the UI, Grafana reads YAML config files at /etc/grafana/provisioning/{datasources,dashboards}/ on startup. A datasources provisioning file declares "here's a Prometheus data source at this URL"; a dashboards provisioning file points Grafana at a directory of JSON files to auto-load. In your Helm-based install, this is already wired: kube-prometheus-stack provisions the Prometheus data source for you automatically, and any dashboard JSON dropped into a labeled ConfigMap gets auto-discovered by a sidecar container that watches the K8s API.

**Templating variables**. A dashboard built with a $node or $instance variable (populated via label_values(up, instance)) becomes reusable across all 3 masters or both workers instead of needing 3 separate dashboards — this is how you avoid dashboard sprawl as your cluster grows.

**The Alternative Landscape**

<img width="912" height="532" alt="image" src="https://github.com/user-attachments/assets/aa577236-c2a4-4a63-aa57-3c487100c569" />

You're on Grafana OSS for the same reason as Module 1 — it's what's actually run in-house at your target companies, and self-hosting it is the differentiator a Datadog-only candidate can't speak to in an interview.

**Interview POV & Edge Cases**

- **Common question:** "How do you prevent dashboard drift across environments (staging vs prod)?" — the expected answer is provisioning-as-code (JSON in Git, loaded via ConfigMap/sidecar), not manual UI edits per environment.
- **Gotcha #1 (the big one):** Grafana ships its own alerting engine (unified alerting, Grafana 8+), separate from Prometheus's rule evaluation + Alertmanager. Engineers who click "New Alert" in the Grafana UI instead of writing a PrometheusRule CRD end up with two parallel, conflicting alerting systems — and worse, if Grafana itself is down during an outage, any alert defined only in Grafana never fires. Convention in K8s-native stacks: alerting logic lives in Prometheus/Alertmanager; Grafana is visualization-only. This is a very commonly probed distinction.
- **Gotcha #2:** a dashboard showing a data gap during an incident doesn't necessarily mean "nothing happened" — check whether it's a real scrape failure (up == 0 for that window) versus Prometheus's staleness marker, versus the Prometheus pod itself having restarted and replayed its WAL.
- **Gotcha #3, specific to your cluster:** kube-prometheus-stack ships pre-built dashboards for K8s objects and node-exporter out of the box, but nothing for HAProxy or etcd — those custom targets need dashboards imported or hand-built, which is exactly what we do below.

**The 'Better Way' (Evolution)**

Hand-editing dashboard JSON doesn't scale past a handful of dashboards. The current direction: Grafana's Foundation SDK (Go/TypeScript/Python) generates dashboard JSON programmatically instead of raw JSON editing, and Grafonnet (Jsonnet-based) does the same for large multi-service dashboard templating. Separately, Perses (CNCF sandbox) is emerging as a vendor-neutral, fully declarative dashboard-as-code alternative that isn't tied to Grafana's proprietary schema — worth knowing the name exists even if Grafana remains dominant today.

**Hands-On Lab**

**Step 1 — Access Grafana and get credentials (already deployed by your Module 1 Helm install):**

```
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo

kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# browse to localhost:3000, user: admin
```

**Step 2 — Import the standard dashboards (Dashboards → Import, by ID):**

- 1860 — Node Exporter Full (covers your masters + 2 workers)
- 315 — Kubernetes cluster monitoring (pod/deployment level)

**Step 3 — Build the HAProxy panel set manually (no bundled dashboard exists). New Dashboard → Add panels using these confirmed metric names:**

<img width="903" height="472" alt="image" src="https://github.com/user-attachments/assets/508f03ba-6bab-42cc-9234-d45f7632e9fa" />

<img width="895" height="351" alt="image" src="https://github.com/user-attachments/assets/662f8169-a925-4acd-be37-063a899fb745" />

**Step 4 — Add an etcd panel** using metrics now exposed from Module 1:

```
etcd_server_has_leader          # 1/0 — catches split-brain immediately
etcd_server_leader_changes_seen_total   # spikes here = instability
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))  # disk latency, the #1 etcd perf killer
```

**Step 5 — Save as code, not just in the UI**. Export the dashboard JSON (Dashboard settings → JSON Model), save it to a file, then provision it declaratively instead of relying on the UI copy:

```
kubectl create configmap haproxy-etcd-dashboard \
  -n monitoring \
  --from-file=haproxy-etcd-dashboard.json \
  --dry-run=client -o yaml | \
  kubectl label --local -f - grafana_dashboard=1 -o yaml | \
  kubectl apply -f -
```

*The grafana_dashboard=1 label is what the sidecar watches for — this ConfigMap is now your dashboard's source of truth, live in Git-manageable YAML instead of trapped in Grafana's UI state.*

**QUESTIONS:**

```
Do I need to port forward even if I am using Kubeadm cluster on AWS EC2 instances?
Also, the observability stack do I need to install on all the instances?
```

**Port-forwarding**

**Not required for the stack to work** — Prometheus/Grafana run fine regardless. Port-forward is just a temporary tunnel through the API server so you can view the UI from your laptop. It's fine for this masterclass, but it's single-session, ties up a terminal, and isn't how you'd access it day-to-day.

Since your kubeadm cluster has no cloud-controller-manager, a type: LoadBalancer Service will just sit in <pending> forever — that's a real gotcha (interviewers ask "how do you expose services on a self-managed cluster" precisely because this trips people up coming from EKS/GKE). Two practical options instead:

**Option A — NodePort (simplest):**

```
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
kubectl get svc monitoring-grafana -n monitoring   # note the NodePort, e.g. 32000
```

Open that port on both workers' security groups, then hit http://<worker-public-ip>:32000.

**Option B — route it through your existing HAProxy box** (better — you already have it, and it's the more production-realistic pattern: one stable URL regardless of which node the pod lands on). Add a new frontend on the HAProxy EC2 instance:

```
frontend grafana_fe
    bind *:3000
    default_backend grafana_be

backend grafana_be
    balance roundrobin
    server worker1 <worker1-ip>:32000 check
    server worker2 <worker2-ip>:32000 check
```

Reload HAProxy → access via http://<haproxy-ip>:3000. NodePort exposes the service on every node regardless of where the pod actually runs, so this keeps working through pod reschedules.

If you want type: LoadBalancer to genuinely work later (no manual HAProxy wiring), MetalLB is the standard answer for bare-metal/self-managed clusters — worth name-dropping in an interview even if you don't set it up today.

**Installing on all instances**

No — you run helm install once. Kubernetes' own scheduler places Prometheus, Grafana, Alertmanager, and kube-state-metrics onto whichever worker has capacity — you never touch those nodes by hand.

The one exception inside the chart: node-exporter is a DaemonSet, so Kubernetes automatically runs one copy per node — including your masters, since the chart adds a toleration for the control-plane taint. Still one helm install, not something you repeat per host.

What genuinely does need manual, per-host work — because it's outside what Helm/kubectl can reach:

<img width="841" height="210" alt="image" src="https://github.com/user-attachments/assets/e76a97b7-66ce-4884-bc48-cfcd8b161095" />

*So: 1 Helm install, 3 manual master edits, 1 manual LB-box edit. Everything else is automatic scheduling.*

**CONTEXT:**

*Nope my setup is using haproxy and 3 master nodes and 2 worker nodes. Can we start the setup from scratch wrt to this architecture? And then move on with below:
That's the visualization layer done. Next up per the roadmap is Module 3: Loki + Promtail for logs — want to move there, or spend more time on this dashboard first? yes we should do the dashboard in details*

**Part 1: Complete Setup From Scratch — 3 Masters + 2 Workers + HAProxy**

Consolidating everything into one clean run, in order, for your exact topology.

**Step 1 — Fix bind-addresses on all 3 masters (stacked etcd + control-plane components)**

Run on **each of the 3 masters individually** (these are static pods, not chart-managed — no way to do this cluster-wide):

```
# etcd metrics port (2381) — separate from client port 2379, no TLS needed
sudo sed -i 's#--listen-metrics-urls=http://127.0.0.1:2381#--listen-metrics-urls=http://0.0.0.0:2381#' \
  /etc/kubernetes/manifests/etcd.yaml

# scheduler + controller-manager — bound to localhost by default since ~v1.22
sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
  /etc/kubernetes/manifests/kube-scheduler.yaml \
  /etc/kubernetes/manifests/kube-controller-manager.yaml

# kubelet watches this directory — pods restart automatically within ~20s
# verify:
sudo grep -E "bind-address|listen-metrics" /etc/kubernetes/manifests/*.yaml
```
**Step 2 — Install kube-prometheus-stack**

Run once, from anywhere with kubectl access (this schedules pods onto your 2 workers automatically — you never touch master/worker filesystems for this part):

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

**Step 3 — Expose HAProxy metrics (on the standalone HAProxy EC2 box only)**

```
haproxy -vv | grep -A3 "Available services"
```

If prometheus-exporter is listed, add to /etc/haproxy/haproxy.cfg:

```
frontend stats
    bind *:8404
    http-request use-service prometheus-exporter if { path /metrics }
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:<strong-password>
```

```
sudo systemctl reload haproxy
curl http://localhost:8404/metrics | head
```

If it says none, fall back to sudo apt install -y prometheus-haproxy-exporter (covered in detail earlier).

**Step 4 — Security groups**

On the HAProxy EC2 instance's SG: inbound 8404 (or 9101 if you used the apt exporter fallback) from your workers' SG — not 0.0.0.0/0.

**Step 5 — Wire the scrape configs together**

```
# values.yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'etcd'
        metrics_path: /metrics
        static_configs:
          - targets: ['<master1-ip>:2381','<master2-ip>:2381','<master3-ip>:2381']
      - job_name: 'haproxy-external'
        static_configs:
          - targets: ['<haproxy-ip>:8404']
```

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f values.yaml
```

**Step 6 — Expose Grafana through HAProxy (not port-forward — production pattern for your architecture)**

```
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
kubectl get svc monitoring-grafana -n monitoring   # note the NodePort
```

On the HAProxy box, add:

```
frontend grafana_fe
    bind *:3000
    default_backend grafana_be

backend grafana_be
    balance roundrobin
    server worker1 <worker1-ip>:<nodeport> check
    server worker2 <worker2-ip>:<nodeport> check
```

```
sudo systemctl reload haproxy
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Browse to http://<haproxy-ip>:3000.

**Step 7 — Verification checklist**

- Prometheus /targets (also route through HAProxy the same way, or temporary port-forward): etcd job shows 3/3 up, haproxy-external shows up
- Grafana loads at http://<haproxy-ip>:3000 with the credentials above
- up{job="haproxy-external"} == 1 and etcd_server_has_leader == 1 both return data in Grafana's Explore view

Setup's done end-to-end. Now the dashboard, in detail.

**Part 2: The HAProxy + etcd Dashboard, In Detail**

Rather than clicking through the UI panel-by-panel, here's the full dashboard as importable JSON — this is the dashboards-as-code practice from Module 2's mechanics section. Save it, import it, and it becomes your Git-tracked source of truth instead of UI state.

```
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "Prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "title": "HAProxy + etcd — LB & Control Plane Health",
  "uid": "haproxy-etcd-health",
  "tags": ["haproxy", "etcd", "control-plane"],
  "timezone": "browser",
  "schemaVersion": 39,
  "version": 1,
  "editable": true,
  "refresh": "30s",
  "time": { "from": "now-6h", "to": "now" },
  "panels": [
    {
      "id": 1, "title": "HAProxy Backend Up",
      "type": "stat", "gridPos": { "x": 0, "y": 0, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "haproxy_backend_up", "legendFormat": "{{backend}}" }],
      "fieldConfig": { "defaults": { "mappings": [
        { "type": "value", "options": { "0": { "text": "DOWN", "color": "red" }, "1": { "text": "UP", "color": "green" } } }
      ] } }
    },
    {
      "id": 2, "title": "etcd Leader Status",
      "type": "stat", "gridPos": { "x": 6, "y": 0, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "etcd_server_has_leader", "legendFormat": "{{instance}}" }],
      "fieldConfig": { "defaults": { "mappings": [
        { "type": "value", "options": { "0": { "text": "NO LEADER", "color": "red" }, "1": { "text": "OK", "color": "green" } } }
      ] } }
    },
    {
      "id": 3, "title": "etcd Leader Changes (15m)",
      "type": "stat", "gridPos": { "x": 12, "y": 0, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "increase(etcd_server_leader_changes_seen_total[15m])", "legendFormat": "{{instance}}" }],
      "fieldConfig": { "defaults": { "thresholds": { "steps": [
        { "value": 0, "color": "green" }, { "value": 1, "color": "red" }
      ] } } }
    },
    {
      "id": 4, "title": "HAProxy Server Downtime Total",
      "type": "stat", "gridPos": { "x": 18, "y": 0, "w": 6, "h": 4 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "haproxy_server_downtime_seconds_total", "legendFormat": "{{server}}" }]
    },
    {
      "id": 5, "title": "HAProxy Active Sessions",
      "type": "timeseries", "gridPos": { "x": 0, "y": 4, "w": 12, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "haproxy_backend_sessions_total", "legendFormat": "{{backend}}" }]
    },
    {
      "id": 6, "title": "HAProxy 5xx Error Rate",
      "type": "timeseries", "gridPos": { "x": 12, "y": 4, "w": 12, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{
        "expr": "sum(rate(haproxy_frontend_http_requests_total{code=\"5xx\"}[5m])) / sum(rate(haproxy_frontend_http_requests_total[5m]))",
        "legendFormat": "error %"
      }],
      "fieldConfig": { "defaults": { "unit": "percentunit" } }
    },
    {
      "id": 7, "title": "HAProxy Server Check Failures",
      "type": "timeseries", "gridPos": { "x": 0, "y": 12, "w": 12, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{ "expr": "rate(haproxy_server_check_failures_total[5m])", "legendFormat": "{{server}}" }]
    },
    {
      "id": 8, "title": "etcd Disk fsync p99 Latency",
      "type": "timeseries", "gridPos": { "x": 12, "y": 12, "w": 12, "h": 8 },
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "targets": [{
        "expr": "histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))",
        "legendFormat": "{{instance}}"
      }],
      "fieldConfig": { "defaults": { "unit": "s", "thresholds": { "steps": [
        { "value": 0, "color": "green" }, { "value": 0.01, "color": "yellow" }, { "value": 0.025, "color": "red" }
      ] } } }
    }
  ]
}
```

**Import path A (quick, UI)**: Grafana → Dashboards → New → Import → paste the JSON → when prompted for the DS_PROMETHEUS input, select your Prometheus data source → Import.

**Import path B (dashboards-as-code, the correct long-term pattern):**

```
kubectl create configmap haproxy-etcd-dashboard \
  -n monitoring --from-file=haproxy-etcd-dashboard.json \
  --dry-run=client -o yaml | \
  kubectl label --local -f - grafana_dashboard=1 -o yaml | \
  kubectl apply -f -
```

The chart's sidecar watches for the grafana_dashboard=1 label and auto-loads it — no UI interaction, no manual re-import after a Grafana restart.

**Why the fsync-latency thresholds matter (interview-relevant):** etcd's own docs flag disk fsync p99 above ~10ms as a warning sign and above ~25ms as likely to trigger leader elections — that's exactly why panel 8 uses those thresholds instead of arbitrary round numbers. This is the metric senior engineers check first when etcd looks "flaky" — it's almost always underlying EBS volume latency, not etcd itself.

## Current-state note before we start

Since your last read on this topic may predate it: Promtail reached full end-of-life on March 2, 2026 — no more updates, security patches, or support. Grafana Labs has folded its functionality into Grafana Alloy (their OpenTelemetry-based unified collector), which is now the only actively maintained path. So we're building this module around Alloy, not Promtail — which is actually the right call for an interview answer today too, since naming Promtail as your current logging agent would read as slightly stale.

Separately: as of March 16, 2026, the Loki OSS Helm chart moved from grafana/helm-charts to grafana-community/helm-charts (community-maintained now; the old repo is GEL/enterprise-only going forward). Commands below use the correct current repo.

## Module 3: Loki + Grafana Alloy (Logs)

**The 'Why' (The Problem)**

Before Loki, K8s log aggregation meant ELK/EFK: Elasticsearch indexes every word of every log line in an inverted index, which means JVM heap sizing headaches, shard rebalancing, and storage costs that scale with log volume, not with how much you actually query. The deeper pain for you specifically: pods are ephemeral — when a pod gets rescheduled or a node is replaced, its logs vanish with it unless something centralizes them with the K8s metadata (namespace/pod/container) attached automatically. And querying logs with a completely different language (Kibana DSL) than your metrics (PromQL) makes correlation — "the HAProxy dashboard shows a spike at 14:32, what do the logs say happened" — a manual, slow, tab-switching exercise.

**Deep-Dive Mechanics**

**The core design bet:** Loki indexes only labels (namespace, app, container — the same small set Prometheus uses), not log content. The actual log text is gzip-compressed into chunks and stored in object storage (or plain disk for our lab). This is why Loki is dramatically cheaper than Elasticsearch at scale — you're not building a searchable index over every word, ever.

**Architecture (monolithic mode, right-sized for a learning cluster):** distributor receives incoming log streams → ingester buffers and periodically flushes compressed chunks to storage → querier reads chunks back out. Production splits these into separate scalable components ("Simple Scalable" or "Microservices" mode); for your cluster, running all of them in one pod (deploymentMode: Monolithic) is correct and is explicitly Grafana's own recommendation for small/meta-monitoring setups.

**Grafana Alloy's component model** replaces Promtail's static YAML with a dataflow graph — components you wire together explicitly:

- discovery.kubernetes — finds pod targets via the K8s API (same SD mechanism as Prometheus)
- discovery.relabel — promotes K8s metadata (namespace, pod, container) into Loki labels
- loki.source.kubernetes — tails the discovered pods' logs via the K8s API
- loki.source.journal — separately reads systemd journal (for kubelet itself, containerd — not the same thing as container logs)
- loki.write — ships the labeled stream to Loki

**LogQL** mirrors PromQL deliberately: {namespace="default", app="nginx"} |= "error" is a stream selector + line filter. You can also derive metrics from log lines with unwrap — e.g., turning nginx's logged response time into a real histogram without needing a separate exporter, which is a genuinely useful "logs-to-metrics" pattern worth knowing.

The cardinality rule carries over exactly from Module 1: label only stable, low-cardinality fields (namespace, app, container). Never make pod_name, request_id, or user_id a label — put them in the log line body and filter with LogQL instead. This is the single most common Loki misconfiguration, and it's the same underlying mechanism (unbounded label values = unbounded streams) as the Prometheus cardinality gotcha you already know.

**The Alternative Landscape**

<img width="912" height="572" alt="image" src="https://github.com/user-attachments/assets/9bdfd4a0-df0f-4e40-88b3-6fdf103c0e9d" />

**Interview POV & Edge Cases**

- **Classic question:** "Why Loki over ELK for Kubernetes logging?" — expects: index-light design → cost, plus the same label model as Prometheus meaning a metric spike and its logs can be correlated in one Grafana pane without leaving the tool.
- **Gotcha #1:** Loki is genuinely bad at ad-hoc full-text search across a high-cardinality field over a long time range (e.g., "find every log line mentioning this arbitrary session ID across 30 days") — that requires brute-force scanning chunks, unlike Elasticsearch's inverted index. Knowing when not to reach for Loki is the senior-level answer.
- **Gotcha #2, specific to your cluster:** loki.source.kubernetes reads via the K8s API and doesn't strictly require a DaemonSet — but for masters you also want loki.source.journal to capture kubelet's own systemd-unit logs (distinct from container logs), which does require running on each node with /var/log/journal mounted. If kubelet itself is crash-looping, container logs won't show you why — journal will.
- **Gotcha #3:** the control-plane taint. Alloy's DaemonSet needs an explicit toleration or it silently skips all 3 masters — meaning you'd lose etcd, apiserver, and scheduler logs precisely when you need them most (during a quorum-loss incident).
- **Gotcha #4:** HAProxy lives outside the cluster entirely — same theme as Module 1. It needs its own Alloy instance (or a lightweight forwarder), not a DaemonSet.

**The 'Better Way' (Evolution)**

Alloy already is "the better way" replacing Promtail — it's Grafana Labs' distribution of the OpenTelemetry Collector. The next step past Alloy is going fully vendor-neutral: run the OpenTelemetry Collector directly and have Loki ingest via its native OTLP endpoint, converging your logs pipeline into the exact same protocol as traces and metrics — no Loki-specific write component at all. Worth naming in an interview as the direction the industry is heading.

**Hands-On Lab**

**Step 1 — Install Loki (monolithic, filesystem storage — right-sized for a lab):**

```
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
```

```
# loki-values.yaml
deploymentMode: Monolithic
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
singleBinary:
  replicas: 1
  persistence:
    enabled: true
    size: 20Gi
# required — chart validation rejects mixed deployment modes otherwise
backend: { replicas: 0 }
read: { replicas: 0 }
write: { replicas: 0 }
```

```
helm install loki grafana-community/loki -n monitoring -f loki-values.yaml
```

**Step 2 — Install Grafana Alloy as a DaemonSet, with the control-plane toleration:**

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```
# alloy-values.yaml
alloy:
  configMap:
    content: |
      discovery.kubernetes "pod" {
        role = "pod"
        selectors {
          role  = "pod"
          field = "spec.nodeName=" + coalesce(sys.env("HOSTNAME"), constants.hostname)
        }
      }

      discovery.relabel "pod_logs" {
        targets = discovery.kubernetes.pod.targets
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_label_app"]
          target_label  = "app"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pod_logs.output
        forward_to = [loki.write.default.receiver]
      }

      loki.source.journal "node_journal" {
        max_age = "12h"
        path    = "/var/log/journal"
        labels  = { job = "node/journal" }
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
        }
      }
tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
extraVolumes:
  - name: journal
    hostPath: { path: /var/log/journal }
extraVolumeMounts:
  - name: journal
    mountPath: /var/log/journal
    readOnly: true
```

```
helm install alloy grafana/alloy -n monitoring -f alloy-values.yaml
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy -o wide
# confirm one Alloy pod per node, including all 3 masters
```

**Step 3 — Wire Loki into Grafana** (add to the values.yaml you already have from Module 1/2):

```
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local
      access: proxy
```

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

**Step 4 — Ship HAProxy's logs from the external EC2 box.** HAProxy isn't a cluster node, so it needs its own local Alloy. Enable HAProxy logging to a file (add to haproxy.cfg: log /dev/log local0), install Alloy as a standalone Linux binary (apt install alloy from Grafana's apt repo, or download the release binary), and give it a minimal local config:

```
local.file_match "haproxy_log" {
  path_targets = [{ __path__ = "/var/log/haproxy.log" }]
}
loki.source.file "haproxy" {
  targets    = local.file_match.haproxy_log.targets
  forward_to = [loki.write.default.receiver]
}
loki.write "default" {
  endpoint { url = "http://<haproxy-ip-mapped-frontend>/loki/api/v1/push" }
}
```

(Route this through the same HAProxy → NodePort pattern from Module 2, on a new frontend pointing at Loki's NodePort, so the external box can reach the in-cluster Loki gateway.)

**Step 5 — Deploy nginx and generate traffic:**

```
kubectl create deployment nginx-demo --image=nginx --replicas=2
kubectl expose deployment nginx-demo --port=80 --type=NodePort
# generate some load
for i in {1..50}; do curl -s http://<worker-ip>:<nodeport>/ > /dev/null; done
curl http://<worker-ip>:<nodeport>/does-not-exist   # generates a 404 for log variety
```

**Step 6 — Query in Grafana → Explore → Loki:**

```
{app="nginx-demo"}
{app="nginx-demo"} |= "404"
sum(rate({app="nginx-demo"}[5m])) by (namespace)
{job="node/journal", unit="kubelet.service"} |= "error"
```

## Module 4: Alertmanager + Alert Design

**The 'Why' (The Problem)**

Prometheus itself can evaluate a rule and know a condition is true — but it has no concept of who should be told, how often, or how to avoid saying the same thing five different ways. Before Alertmanager, ops teams either hardcoded a destination into every individual check (Nagios-style — change your escalation policy and you're editing hundreds of scripts), or had no grouping at all, so a single root cause (one node dying) fanned out into a dozen near-identical pages for every service that happened to be running on it. That's the alert-storm problem, and it's the single biggest driver of alert fatigue and ignored pages in real incidents.

The pain specific to your topology: one underlying event — say, a kubelet crash on master-2 — can legitimately trigger apiserver-down, scheduler-election, etcd-member-down, node-NotReady, and HAProxy-backend-down alerts simultaneously, all describing the same root cause. Without grouping and inhibition, that's five pages at 3 a.m. for one incident. Alertmanager exists specifically to collapse that into one coherent notification.

**Deep-Dive Mechanics**

**Where alerts originate.** Alert rules live in PrometheusRule CRDs (with the Prometheus Operator, which kube-prometheus-stack uses) and are evaluated by Prometheus itself on a timer, completely independent of Alertmanager. Each rule moves through a state machine: Inactive → Pending → Firing. The for: duration is the mechanism that prevents a single missed scrape or transient blip from paging someone — the condition must stay true for the whole duration before it fires. Too short a for:, and you page on noise; too long, and you're slow to notice a real incident.

**Alertmanager's pipeline**, once a firing alert arrives:

- Grouping — bucket alerts sharing labels (group_by: ['alertname','cluster']) into a single notification instead of one-per-alert.
- Inhibition — suppress a lower-priority alert if a related higher-priority one is already firing (e.g., don't page individually for every pod on a node if NodeDown is already firing for that node).
- Silencing — a manual, time-bound mute for planned maintenance.
- Routing tree — a tree of label matchers deciding which receiver(s) get the notification; continue: true lets one alert match multiple branches (e.g., page and post to Slack).
- Notification, with repeat_interval controlling how often a still-firing alert re-notifies.

**Deduplication and HA.** Alertmanager itself runs as a gossip-based cluster so that if you scale Prometheus to multiple replicas (solving the Module 1 SPOF gotcha), you don't get duplicate pages for the same alert — dedup is based on a fingerprint hash of the alert's labels.

**The Alternative Landscape**

<img width="875" height="578" alt="image" src="https://github.com/user-attachments/assets/e3f528b6-7df1-4b94-9fdc-7b288495238e" />

**Important interview clarification:** Alertmanager and PagerDuty are not competing choices — Alertmanager is the routing/dedup/grouping brain; PagerDuty is typically the terminal on-call-scheduling/escalation layer that one of Alertmanager's receivers points at via webhook. Conflating the two is a common junior mistake.

**Interview POV & Edge Cases**

- **Classic question:** "How would you design alerting for a distributed system to avoid alert fatigue?" — expects: symptom-based alerting (alert on user-visible failure, not every internal cause), grouping + inhibition to prevent storms, deliberate for: durations, and severity-based routing.
- **Gotcha #1 — the big one for your setup:** a naive alert on "any etcd member down" fires on every routine blip, but that's wrong. etcd with 3 members tolerates exactly 1 failure without losing quorum (floor((N-1)/2) = 1). The alert that actually matters checks whether the number of down members threatens quorum, not whether any single member is down. This distinction is exactly what separates a senior engineer's alert design from a junior one — and it's what we build below.
- **Gotcha #2:** Alertmanager itself needs HA — one replica dying means Prometheus is still correctly evaluating rules, but nothing routes or notifies. Same SPOF pattern as Module 1's Prometheus warning.
- **Gotcha #3, specific to your architecture:** Prometheus scrapes etcd/scheduler/apiserver directly on each master's IP — not through HAProxy. That means if HAProxy itself is down or misrouting, but all 3 masters are individually healthy, none of your existing alerts catch it — yet every real client (kubectl, in-cluster workloads) is experiencing a full outage. You need a synthetic check that actually goes through the HAProxy VIP, not around it — this is the kind of blind spot interviewers love to probe for.
- **Gotcha #4:** PrometheusRule CRDs need a label matching the Operator's ruleSelector (in kube-prometheus-stack, typically release: <helm-release-name>) — omit it and your custom rules are silently ignored. Extremely common real-world footgun.

**The 'Better Way' (Evolution)**

Static thresholds (> 80% CPU) don't adapt to normal daily/weekly traffic patterns and cause both false positives and missed anomalies — the modern direction is dynamic/anomaly-based alerting (Grafana's ML-based forecasting, or hand-rolled seasonal baselines in PromQL). Separately, on the notification/on-call layer: Grafana OnCall OSS was archived on March 24, 2026 — Grafana Labs stopped active development and pushed users toward the paid Grafana Cloud IRM. If you want a self-hosted on-call/escalation layer today, look at community alternatives (e.g., Keep) or just pair Alertmanager's webhook receiver with PagerDuty/Opsgenie directly, which was always the more common production pattern anyway. Module 6 (SLOs) will also show you multi-window burn-rate alerting — a fundamentally better approach than static thresholds for anything customer-facing.

**Hands-On Lab**

**Step 1 — Install blackbox_exporter (to close the HAProxy-blind-spot gotcha above):**

```
helm install blackbox prometheus-community/prometheus-blackbox-exporter -n monitoring
```

Add a scrape config probing the apiserver **through the HAProxy VIP**, not around it:

```
# add to additionalScrapeConfigs alongside etcd/haproxy-external from earlier modules
- job_name: 'blackbox-apiserver-via-haproxy'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets: ['https://<haproxy-ip>:6443/healthz']
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox.monitoring.svc.cluster.local:9115
```

**Step 2 — Custom alert rules, correctly modeling etcd quorum math:**

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-infra-alerts
  namespace: monitoring
  labels:
    release: monitoring   # MUST match the Operator's ruleSelector or this is silently ignored
spec:
  groups:
    - name: etcd-quorum
      rules:
        - alert: etcdNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          labels: { severity: warning }
          annotations:
            summary: "etcd member {{ $labels.instance }} reports no leader (may be a brief election)"
        - alert: EtcdQuorumAtRisk
          expr: count(up{job="etcd"} == 0) >= (count(up{job="etcd"}) / 2)
          for: 3m
          labels: { severity: critical }
          annotations:
            summary: "etcd quorum at risk — {{ $value }} of 3 members down, cluster may go read-only"
    - name: haproxy
      rules:
        - alert: HAProxyBackendDown
          expr: haproxy_backend_up == 0
          for: 1m
          labels: { severity: critical }
          annotations:
            summary: "HAProxy backend {{ $labels.backend }} is DOWN"
    - name: apiserver-lb
      rules:
        - alert: APIServerUnreachableViaLB
          expr: probe_success{job="blackbox-apiserver-via-haproxy"} == 0
          for: 2m
          labels: { severity: critical }
          annotations:
            summary: "kube-apiserver unreachable via the HAProxy VIP — check LB config even if masters look healthy individually"
```

```
kubectl apply -f custom-infra-alerts.yaml
```

Notice the distinction between etcdNoLeader (per-instance, severity: warning — a brief election is normal) and EtcdQuorumAtRisk (cluster-wide math, severity: critical — this is the one that should actually page you). That's the senior-vs-junior distinction from the gotcha above, encoded directly into the rule design.

**Step 3 — Alertmanager routing, grouping, and inhibition** (add to your accumulating values.yaml):

```
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'default-webhook'
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - match: { severity: critical }
          receiver: 'critical-pager'
        - match: { severity: warning }
          receiver: 'warning-slack'
    inhibit_rules:
      - source_match: { alertname: 'EtcdQuorumAtRisk' }
        target_match: { alertname: 'etcdNoLeader' }
        equal: ['instance']
      - source_match: { severity: 'critical' }
        target_match: { severity: 'warning' }
        equal: ['alertname', 'instance']
    receivers:
      - name: 'default-webhook'
        webhook_configs: [{ url: 'http://example-webhook:8080/' }]
      - name: 'critical-pager'
        webhook_configs: [{ url: '<your PagerDuty/Opsgenie integration URL>' }]
      - name: 'warning-slack'
        slack_configs: [{ api_url: '<slack-webhook-url>', channel: '#alerts-warning' }]
```

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

**Step 4 — Verify:**

```
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093
# localhost:9093 — confirm rules loaded under /#/status, then simulate:
```

*Test the quorum alert for real: stop kubelet on one master (sudo systemctl stop kubelet) — you should see etcdNoLeader or a brief blip but not a critical page (1 down is tolerated). Stop a second master and you should see EtcdQuorumAtRisk fire and page — this is the actual quorum-loss boundary, demonstrated on real infrastructure instead of just described.*

## Module 5: nginx RED-Method Instrumentation

**The 'Why' (The Problem)**

Node-exporter and kube-state-metrics tell you about infrastructure health — CPU, memory, pod status. They tell you nothing about whether a user is having a bad time. The RED method (Rate, Errors, Duration) — Tom Wilkie's framework, the request-driven counterpart to the resource-focused USE method — exists specifically to answer "is the service actually working for the people using it," which is a completely different question than "is the box healthy."

The pain specific to this module: nginx's basic stub_status module — what nginx-prometheus-exporter reads — only exposes active/accepted/handled connection counts. It has no per-status-code breakdown and no request duration at all. A lot of tutorials wire up nginx-prometheus-exporter and call it done for RED metrics — it isn't. You get the "R," but not "E" or "D." Getting genuine RED coverage means combining what we already built: Prometheus for the basic rate, and Loki's unwrap for duration and error rate straight from logs — which is exactly why Module 3 wasn't a detour.

**Deep-Dive Mechanics**

**RED vs USE:** RED (Rate, Errors, Duration) is for request-driven services — apply it to nginx, HAProxy, any API. USE (Utilization, Saturation, Errors) is for resources — apply it to CPU, disk, memory (what node-exporter already gives you). Interviewers use this distinction to check you're not just cargo-culting "the four golden signals" without understanding which method fits which layer.

**Why stub_status falls short:** it exposes Active connections, accepts, handled, requests — connection-level counters, no status codes, no timing. To get real error rate and duration, you need either NGINX Plus (paid API), the third-party VTS module (extra compile-time module), or — the path we take here — a custom log format with $request_time and $status, shipped to Loki via the exact same container-log pipeline from Module 3, then queried with unwrap to turn a log field into a numeric time series. No new agent, no redeploy of your monitoring stack — just a richer log line.

**The Alternative Landscape**

<img width="903" height="526" alt="image" src="https://github.com/user-attachments/assets/8ae6f299-5955-47b3-95f2-402b570a2fe6" />

**Interview POV & Edge Cases**

- **Classic question:** "How do you get golden-signal metrics for a service you don't control the source of?" — the strong answer is exactly this pattern: reverse-proxy or log-based metrics extraction, not "add an SDK to the app."
- **Gotcha:** assuming nginx-prometheus-exporter gives you error rate — it doesn't, and candidates who don't know this design a dashboard with a permanently-flat "error rate" panel that never fires.
- **Gotcha:** averaging latency instead of using percentiles — avg(request_time) hides the p99 tail where real user pain lives. This is exactly why we use quantile_over_time below, not an average.

**The 'Better Way' (Evolution)**

OpenTelemetry auto-instrumentation and eBPF-based tools (Pixie, Cilium Hubble) derive RED metrics with zero config on the proxy at all — reading straight off the network layer. Worth naming as the direction the industry's heading, even though the log-based approach here is the pragmatic, zero-new-infrastructure answer for a proxy you don't want to recompile.

**Hands-On Lab**

**Step 1 — nginx with a structured log format + stub_status, via ConfigMap:**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-red-conf
data:
  nginx.conf: |
    events {}
    http {
      log_format json_combined escape=json '{ "time":"$time_iso8601", "status":"$status", "request_time":$request_time, "request":"$request" }';
      access_log /dev/stdout json_combined;
      server {
        listen 80;
        location /stub_status { stub_status; allow 127.0.0.1; deny all; }
        location / { return 200 "ok\n"; }
        location /error { return 500 "synthetic error\n"; }
      }
    }
```

Note request_time is unquoted — that keeps it numeric in the JSON so Loki's unwrap can treat it as a sample value, not a string.

**Step 2 — Deploy with the exporter as a sidecar:**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 2
  selector: { matchLabels: { app: nginx-demo } }
  template:
    metadata: { labels: { app: nginx-demo } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          volumeMounts: [{ name: conf, mountPath: /etc/nginx/nginx.conf, subPath: nginx.conf }]
        - name: exporter
          image: nginx/nginx-prometheus-exporter:1.5.1
          args: ["--nginx.scrape-uri=http://127.0.0.1:80/stub_status"]
          ports: [{ containerPort: 9113 }]
      volumes: [{ name: conf, configMap: { name: nginx-red-conf } }]
---
apiVersion: v1
kind: Service
metadata: { name: nginx-demo, labels: { app: nginx-demo } }
spec:
  selector: { app: nginx-demo }
  ports: [{ name: http, port: 80 }, { name: metrics, port: 9113 }]
  type: NodePort
```

**Step 3 — Register the scrape target the Operator-native way** (same release label footgun as Module 4 applies here too — the ServiceMonitor needs it or it's silently ignored):

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-demo
  namespace: monitoring
  labels: { release: monitoring }
spec:
  selector: { matchLabels: { app: nginx-demo } }
  namespaceSelector: { matchNames: ["default"] }
  endpoints: [{ port: metrics, interval: 15s }]
```

**Step 4 — Generate traffic, including errors:**

```
kubectl get svc nginx-demo   # note the NodePort
for i in {1..100}; do curl -s http://<worker-ip>:<nodeport>/ > /dev/null; done
for i in {1..10}; do curl -s http://<worker-ip>:<nodeport>/error > /dev/null; done
```

**Step 5 — Full RED via LogQL (Grafana → Explore → Loki):**

```
# Rate
sum(rate({app="nginx-demo"} | json [5m]))

# Errors (%)
sum(rate({app="nginx-demo"} | json | status >= 500 [5m]))
/
sum(rate({app="nginx-demo"} | json [5m]))

# Duration (p95, per pod)
quantile_over_time(0.95, {app="nginx-demo"} | json | unwrap request_time [5m]) by (pod)
```

## Module 6: SLOs, Error Budgets, Multi-Window Burn-Rate Alerts

**The 'Why' (The Problem)**

Module 4's alerts are threshold-based: "page if error rate > X%." That number is a guess — set it too sensitive and you page on noise; too loose and real degradation goes unnoticed. Google's SRE Workbook formalized a better question: instead of "did a number cross a line," ask "how fast are we breaking a promise we made to users, and does that pace justify waking someone up right now." That reframing is what an SLO gives you — alerting tied to actual user impact and a real deadline, not an arbitrary threshold.

**Deep-Dive Mechanics**

- **SLI (indicator):** the measured ratio — e.g., proportion of non-5xx requests, built directly on Module 5's RED metrics.
- **SLO (objective):** a target over a rolling window — e.g., 99.9% success over 30 days.
- **Error budget:** 1 − SLO = 0.1% allowed failures — a concrete, spendable allowance, not "try to have fewer errors."
- **Burn rate:** how fast you're consuming that budget relative to a sustainable pace. Burn rate = 1 means you'll exhaust the budget exactly at the end of the window if the current rate holds; burn rate = 14.4 means you'd exhaust a 30-day budget in about 2 days.
- **Multi-window, multi-burn-rate alerting** — the actual SRE Workbook recipe, combining a short window (fast detection) AND a long window (confirms it's not a blip) at each severity:

<img width="886" height="251" alt="image" src="https://github.com/user-attachments/assets/a60ae931-6d90-4911-8afd-f55b8cdfef62" />

**Formula**: burn_rate = (1 − SLI_over_window) / (1 − SLO). Both windows must exceed the threshold simultaneously — a spike that only shows in the short window is noise; requiring the long window too is what keeps this alert both fast and accurate, avoiding Module 4's flapping problem entirely.

**The Alternative Landscape**

<img width="880" height="466" alt="image" src="https://github.com/user-attachments/assets/4cd16417-e748-42e7-8adb-458a15b8feb7" />

**Interview POV & Edge Cases**

- **Classic question:** "Design an alerting strategy for a payment API based on SLOs." — expects multi-window multi-burn-rate reasoning by name, not raw thresholds.
- **Gotcha #1:** single-window burn-rate alerts cause false pages — a 2-minute total outage makes a 1-hour window look catastrophic even though the actual budget impact is tiny. The AND-with-a-long-window is specifically the fix, not an optional refinement.
- **Gotcha #2:** choosing the wrong SLI. Averaging latency across all requests (Module 5's mistake) hides p99 pain for a real subset of users — the SLI has to reflect what users experience, which is why we built percentiles via unwrap, not averages.
- **Gotcha #3, the most senior-level insight here:** a 100% SLO is not just unrealistic — it's actively wrong to target. Zero error budget means zero room to deploy, run chaos experiments, or take any engineering risk. Error budgets exist to give teams permission to ship, as long as budget remains. This reframes SLOs as a velocity tool, not just a monitoring one — and it's exactly the kind of answer that distinguishes a senior candidate.
- **Gotcha #4, specific to your architecture:** which layer's SLO are you even measuring — HAProxy accepted the connection, or nginx returned <400? In a real multi-tier system, an inner layer's failures can be masked by retries at an outer layer. Being explicit about which boundary your SLO measures is a real interview differentiator.

**The 'Better Way' (Evolution)**

Sloth and Pyrra (both actively maintained) generate the entire recording-rule + multi-window-alert matrix from a short YAML spec instead of hand-writing it — the hand-rolled version below is for understanding the mechanics; in production, generate it.

**Hands-On Lab**

We'll define the SLO on the HAProxy frontend layer — it already gives us per-status-code metrics natively (haproxy_frontend_http_requests_total{code=...}, confirmed in Module 2), so no Loki dependency needed for this one.

**Step 1 — Multi-window recording rules + burn-rate alerts (target: 99.9% success):**

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: nginx-slo-burnrate
  namespace: monitoring
  labels: { release: monitoring }
spec:
  groups:
    - name: slo-recording-rules
      rules:
        - record: slo:haproxy_requests:ratio_rate5m
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[5m])) / sum(rate(haproxy_frontend_http_requests_total[5m]))
        - record: slo:haproxy_requests:ratio_rate30m
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[30m])) / sum(rate(haproxy_frontend_http_requests_total[30m]))
        - record: slo:haproxy_requests:ratio_rate1h
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[1h])) / sum(rate(haproxy_frontend_http_requests_total[1h]))
        - record: slo:haproxy_requests:ratio_rate2h
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[2h])) / sum(rate(haproxy_frontend_http_requests_total[2h]))
        - record: slo:haproxy_requests:ratio_rate6h
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[6h])) / sum(rate(haproxy_frontend_http_requests_total[6h]))
        - record: slo:haproxy_requests:ratio_rate1d
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[1d])) / sum(rate(haproxy_frontend_http_requests_total[1d]))
        - record: slo:haproxy_requests:ratio_rate3d
          expr: sum(rate(haproxy_frontend_http_requests_total{code!~"5.."}[3d])) / sum(rate(haproxy_frontend_http_requests_total[3d]))
    - name: slo-burnrate-alerts
      rules:
        - alert: HAProxySLOBurnRateCritical
          expr: |
            ( (1 - slo:haproxy_requests:ratio_rate1h) / (1 - 0.999) > 14.4
              and (1 - slo:haproxy_requests:ratio_rate5m) / (1 - 0.999) > 14.4 )
            or
            ( (1 - slo:haproxy_requests:ratio_rate6h) / (1 - 0.999) > 6
              and (1 - slo:haproxy_requests:ratio_rate30m) / (1 - 0.999) > 6 )
          labels: { severity: critical }
          annotations: { summary: "SLO burn rate critical — budget exhausts in days if sustained" }
        - alert: HAProxySLOBurnRateWarning
          expr: |
            ( (1 - slo:haproxy_requests:ratio_rate1d) / (1 - 0.999) > 3
              and (1 - slo:haproxy_requests:ratio_rate2h) / (1 - 0.999) > 3 )
            or
            ( (1 - slo:haproxy_requests:ratio_rate3d) / (1 - 0.999) > 1
              and (1 - slo:haproxy_requests:ratio_rate6h) / (1 - 0.999) > 1 )
          labels: { severity: warning }
          annotations: { summary: "SLO burn rate elevated — ticket, not page" }
```

```
kubectl apply -f nginx-slo-burnrate.yaml
```

**Step 2 — The production alternative, for the record (Sloth spec instead of hand-writing the above):**

```
version: "prometheus/v1"
service: "nginx-via-haproxy"
labels: { team: "platform" }
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(haproxy_frontend_http_requests_total{code=~"5.."}[{{.window}}]))
        total_query: sum(rate(haproxy_frontend_http_requests_total[{{.window}}]))
    alerting:
      name: HAProxySLOAlert
      pageAlert: { labels: { severity: critical } }
      ticketAlert: { labels: { severity: warning } }
```

sloth generate -i slo.yaml -o slo-rules.yaml produces exactly the recording+alert rules above, automatically, correctly, every time — this is what you'd actually run day-to-day.

**Step 3 — Verify by forcing a burn:** rerun the /error traffic loop from Module 5 at a higher volume and watch slo:haproxy_requests:ratio_rate5m drop in Grafana, then confirm HAProxySLOBurnRateCritical fires once both the 5m and 1h windows cross threshold together — not before.

## Module 7 (Bonus): OpenTelemetry & Distributed Tracing — The Evolution

**The 'Why' (The Problem)**

Metrics (Module 1) tell you something is wrong in aggregate. Logs (Module 3) tell you what happened at one point. Neither tells you where in a multi-hop request path the time actually went. Once your nginx demo grows into something resembling your six-project portfolio plan — a real API behind it, calling a database, calling another service — a latency spike becomes genuinely hard to debug: metrics show duration went up, logs show separate, disconnected events per service, and you're left manually eyeballing timestamps across log streams to guess which hop was slow. That manual correlation is exactly what distributed tracing was built to eliminate.

**Deep-Dive Mechanics**

**Trace = a tree of spans.** Each span is one unit of work (a request hitting nginx, a call to a backend, a DB query) with a start time, duration, and parent-child link back to the span that caused it.

**Context propagation is the load-bearing mechanism.** A trace_id/span_id pair is generated at the entry point and passed downstream via the W3C Trace Context standard — a traceparent HTTP header. Every service in the path must read that header, create its own child span, and forward it onward. This is why tracing needs either code instrumentation or a proxy capable of injecting/propagating headers — it doesn't happen for free.

**OpenTelemetry (OTel) is the vendor-neutral standard unifying this across all three pillars:** one SDK/API per language, one wire protocol (OTLP). The OTel Collector sits between your services and your backend, batching/filtering/transforming telemetry so you can swap backends (Tempo, Jaeger, a vendor) without touching application config.

**Sampling is the practical constraint.** Tracing every request at scale is expensive. Head-based sampling decides upfront (e.g., "sample 1% of requests") — cheap, but you might never capture the one slow/erroring trace you actually care about. Tail-based sampling buffers the whole trace and decides afterward, so you can guarantee capturing every trace with an error or high latency — much more useful, but requires a buffering collector tier.

**Grafana Tempo** is the tracing equivalent of Loki's design philosophy — indexes only by trace ID, stores span data cheaply in object storage. The real payoff is exemplars: a metric data point that carries an embedded trace ID, letting you click a latency spike on a Prometheus-backed Grafana graph and jump straight to the exact trace that caused it — the "one pane of glass across all three pillars" this entire day has been building toward.

**The Alternative Landscape**

<img width="927" height="537" alt="image" src="https://github.com/user-attachments/assets/84f9fa20-f24d-4bcc-90de-89aa7a9e26ea" />

**Interview POV & Edge Cases**

- **Classic question:** "How would you debug a latency spike across microservices?" — the strong answer names distributed tracing and context propagation explicitly, and explains correlating it with metrics/logs via shared trace IDs — not just "check the logs."
- **Gotcha #1:** tracing is only as good as its weakest propagation hop. If one service in the chain — a message queue, a proxy that strips unrecognized headers — drops the traceparent, the trace fragments into disconnected pieces with no way to stitch them back together. This is a very real, very common failure mode.
- **Gotcha #2:** 100% sampling is a cost/performance own-goal at scale; pure random head-based sampling risks never capturing the rare slow/erroring request you actually need. Tail-based sampling fixes this but needs a buffering collector tier — a real infra trade-off, not just a config flag.
- **Gotcha #3 — specific to your stack, and important to get right:** nginx can natively participate in tracing via the ngx_otel_module (actively maintained by F5/nginx, available for both OSS and Plus) — it generates spans and propagates traceparent to upstreams automatically. HAProxy cannot. HAProxy has no native distributed-tracing support — it's a first-class citizen for metrics (Module 1) and logs (Module 3), but in a trace it's an invisible pass-through, not a span, unless you add manual instrumentation or a sidecar. Knowing which layer of your own stack can and can't participate in tracing is exactly the kind of architectural precision that separates a strong answer from a hand-wavy one.

**The 'Better Way' (Evolution)**

OTel already is the modern answer. The next step past it is eBPF-based auto-instrumentation (Pixie, Odigos) — deriving traces from kernel/network-level observation with zero code or proxy config changes at all. Worth naming as where the industry is heading, even though OTel + Tempo is today's pragmatic, production-proven choice.

**Roadmap: Next Steps to Explore**

This is intentionally the lighter, "what to build next" module rather than a fully walked lab — here's the concrete path:

**1. Install Tempo** (chart also moved to the community repo, same as Loki — and note the default port changed from 3100 to 3200 in recent chart versions, which trips up a lot of older tutorials):

```
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm install tempo grafana-community/tempo -n monitoring
```

**2. Deploy an OTel Collector** to receive OTLP and forward to Tempo — this decouples "how nginx emits spans" from "where they land."

**3. Enable native tracing on your nginx demo** (Module 5's deployment) via ngx_otel_module:

```
load_module modules/ngx_otel_module.so;
http {
  otel_exporter { endpoint otel-collector.monitoring.svc.cluster.local:4317; }
  otel_service_name "nginx-demo";
  server {
    otel_trace on;
    otel_trace_context propagate;
  }
}
```

**4. Add Tempo as a Grafana data source** and wire up trace-to-logs and trace-to-metrics correlation — this is the actual payoff of the whole day: click a latency spike in your Module 5/6 dashboards and jump straight into the exact trace and the exact log lines for that request.

**5. Set expectations correctly going in:** your trace will show nginx's spans in detail, but HAProxy will appear as a gap — a real, accurate limitation of the architecture, not a misconfiguration, per the gotcha above.

## Observability Masterclass — Interview Cheat Sheet

**M1: Prometheus**

- **Model:** pull, scrapes /metrics every 15–30s; absence of a scrape (up==0) is the signal.
- **4 types:** Counter (rate() only, never read raw) · Gauge · Histogram (aggregatable across instances — prefer this) · Summary (per-instance quantiles, cannot aggregate across pods).
- rate() for alerts/dashboards (handles counter resets) · irate() only for high-res graphing, never in alert rules (flaps).
- **Your topology gotchas:** scheduler/controller-manager/etcd bind to 127.0.0.1 by default — must patch to 0.0.0.0. etcd client (2379, mTLS) ≠ metrics (2381, plain HTTP) — don't conflate them. Cardinality explosion = unbounded label values (never label by user_id/request_id). One Prometheus instance = SPOF → Thanos/Mimir for HA.

**M2: Grafana**

- Stateless; queries datasources live at render time; dashboards are JSON → provision via ConfigMap + sidecar (grafana_dashboard=1 label) = dashboards-as-code.
- Gotcha: Grafana's own alerting engine vs. Alertmanager — keep alert logic in Prometheus/Alertmanager; Grafana = visualization only, or you get duplicate/conflicting alerting systems.

**M3: Loki + Alloy**

- **Indexes labels only**, not full text — cheap vs. ELK's inverted index.
- **Promtail is EOL (March 2, 2026)** → Grafana Alloy is the current standard collector.
- LogQL: {app="x"} |= "err" · unwrap turns a numeric log field into a real metric (quantile_over_time, rate).
- **Gotcha:** same cardinality rule as Prometheus. DaemonSet needs the control-plane toleration or you silently lose etcd/apiserver/kubelet logs from all 3 masters.

**M4: Alertmanager**

- Pipeline: **group → inhibit → silence → route → notify.**
- etcd tolerates floor((N-1)/2) failures — for N=3, tolerate 1 down; alert when 2 are down (not "any down").
- **Gotcha:** PrometheusRule/ServiceMonitor need the release: <helm-release-name> label or the Operator silently ignores them.
- **Gotcha (your architecture):** Prometheus scrapes masters directly, bypassing HAProxy — add a blackbox probe through the VIP to catch "LB down, masters individually fine" outages.

**M5: RED Method**

- **RED (Rate, Errors, Duration)** = request-driven services. USE (Utilization, Saturation, Errors) = resources.
- nginx stub_status ≠ full RED — no status codes, no duration. Get E+D from logs via unwrap; R from the exporter or log line rate.

**M6: SLOs & Burn Rate**

```
burn_rate = (1 − SLI_over_window) / (1 − SLO)
```

<img width="878" height="256" alt="image" src="https://github.com/user-attachments/assets/5f9de780-7289-48e6-9966-bb2f588781db" />

- **Both windows must breach together** — this is what prevents false pages on short blips.
- Error budget = a **deploy-velocity tool**, not just a monitoring concept — zero budget should mean zero risky deploys.
- Generate with **Sloth/Pyrra** in production; don't hand-roll the matrix.

**M7: Tracing**

- **Trace** = tree of spans; context propagates via the W3C traceparent header — one broken hop fragments the whole trace.
- **OTel** = vendor-neutral SDK + OTLP protocol; the Collector decouples instrumentation from backend.
- **Tempo** = Loki-shaped for traces; exemplars link a metric spike directly to its trace.
- **nginx:** native tracing via ngx_otel_module. HAProxy: no native tracing — metrics and logs yes, traces no.

## One-line answers if asked "why X over Y":

- Prometheus over Graphite → labels + pull model handle ephemeral pods; Graphite's flat naming can't.
- Loki over ELK → index-light (labels only) = far cheaper; trades away ad-hoc full-text search at scale.
- Alertmanager + PagerDuty are not alternatives → Alertmanager routes/dedupes, PagerDuty schedules/escalates; they're stacked, not swapped.
- SLO-based alerting over static thresholds → ties paging to actual user-facing budget burn, not an arbitrary number.








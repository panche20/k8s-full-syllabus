# Day 26 — Capstone Expansion: Slack Alerts, Prometheus & Operator Pattern

*Yesterday you built the core analyzer. Today you make it production-grade — continuous monitoring, real-time alerts, metrics, and the full Operator pattern. This is what turns a useful script into infrastructure teams rely on.*

## 🧠 Part 1: What Production-Grade Means

```
Day 28 (script)         Day 29 (production)
─────────────────       ──────────────────────────────
Run manually            Runs continuously as an Operator
Terminal output         Slack alerts + Prometheus metrics
One-shot analysis       Tracks issue state over time
No deduplication        Alert suppression + cooldown
No history              Issue timeline in etcd (via CRD)
Single cluster          Multi-cluster ready
```

The gap between "works on my machine" and "infrastructure team relies on it" is exactly what today covers.

## 🏗️ Part 2: Expanded Architecture

```
k8s-analyzer-operator/
├── main.py                      — Operator entry point
├── operator/
│   ├── controller.py            — reconciliation loop
│   ├── crd.py                   — ClusterAnalysis CRD management
│   └── state.py                 — issue state tracking
├── collector/                   — (from Day 28, unchanged)
│   ├── pods.py
│   ├── nodes.py
│   └── workloads.py
├── detector/                    — (from Day 28, unchanged)
│   └── issues.py
├── analyzer/
│   ├── claude.py                — (from Day 28, enhanced)
│   └── cache.py                 — response caching
├── alerts/
│   ├── slack.py                 — Slack webhook integration
│   ├── webhook.py               — generic webhook (PagerDuty, OpsGenie)
│   └── suppressor.py            — deduplication + cooldown
├── metrics/
│   └── prometheus.py            — custom metrics server
├── output/
│   └── terminal.py              — (from Day 28)
└── config/
    └── settings.py              — configuration management
```

## ⚙️ Part 3: Configuration Management

```
# config/settings.py
from pydantic import BaseSettings, Field
from typing import Optional, List
from enum import Enum

class LogLevel(str, Enum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"

class Settings(BaseSettings):
    # Core
    anthropic_api_key: Optional[str] = Field(None, env="ANTHROPIC_API_KEY")
    kubeconfig_path: Optional[str] = Field(None, env="KUBECONFIG")
    cluster_name: str = Field("unknown", env="CLUSTER_NAME")

    # Scan behavior
    scan_interval_seconds: int = Field(300, env="SCAN_INTERVAL_SECONDS")
    namespaces: List[str] = Field([], env="NAMESPACES")
    excluded_namespaces: List[str] = Field(
        ["kube-system", "kube-public", "kube-node-lease"],
        env="EXCLUDED_NAMESPACES"
    )
    max_issues_per_scan: int = Field(10, env="MAX_ISSUES_PER_SCAN")
    max_log_lines: int = Field(50, env="MAX_LOG_LINES")

    # AI analysis
    ai_enabled: bool = Field(True, env="AI_ENABLED")
    ai_model: str = Field("claude-sonnet-4-20250514", env="AI_MODEL")
    ai_cache_ttl_seconds: int = Field(3600, env="AI_CACHE_TTL_SECONDS")
    ai_max_per_scan: int = Field(5, env="AI_MAX_PER_SCAN")

    # Alerting
    slack_webhook_url: Optional[str] = Field(None, env="SLACK_WEBHOOK_URL")
    slack_channel: str = Field("#k8s-alerts", env="SLACK_CHANNEL")
    alert_cooldown_seconds: int = Field(1800, env="ALERT_COOLDOWN_SECONDS")
    alert_on_severity: List[str] = Field(
        ["CRITICAL", "WARNING"],
        env="ALERT_ON_SEVERITY"
    )

    # Metrics
    metrics_port: int = Field(9090, env="METRICS_PORT")
    metrics_enabled: bool = Field(True, env="METRICS_ENABLED")

    # Operator
    operator_mode: bool = Field(False, env="OPERATOR_MODE")
    leader_election: bool = Field(True, env="LEADER_ELECTION")

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

## 🔔 Part 4: Slack Integration

```
# alerts/slack.py
import requests
import json
from datetime import datetime
from typing import List, Optional
from detector.issues import ClusterIssue, Severity

SEVERITY_EMOJI = {
    Severity.CRITICAL: "🔴",
    Severity.WARNING: "🟡",
    Severity.INFO: "🔵"
}

SEVERITY_COLOR = {
    Severity.CRITICAL: "#FF0000",
    Severity.WARNING: "#FFA500",
    Severity.INFO: "#0080FF"
}

class SlackAlerter:
    def __init__(self, webhook_url: str, channel: str, cluster_name: str):
        self.webhook_url = webhook_url
        self.channel = channel
        self.cluster_name = cluster_name

    def send_issue_alert(
        self,
        issue: ClusterIssue,
        ai_analysis: Optional[str] = None
    ) -> bool:
        """Send a Slack alert for a specific issue."""
        emoji = SEVERITY_EMOJI[issue.severity]
        color = SEVERITY_COLOR[issue.severity]

        # Build the attachment
        fields = [
            {
                "title": "Cluster",
                "value": self.cluster_name,
                "short": True
            },
            {
                "title": "Namespace",
                "value": issue.namespace,
                "short": True
            },
            {
                "title": "Resource",
                "value": issue.affected_resource,
                "short": True
            },
            {
                "title": "Severity",
                "value": f"{emoji} {issue.severity.value}",
                "short": True
            }
        ]

        # Add AI analysis if available
        if ai_analysis:
            # Extract just the root cause section
            root_cause = self._extract_root_cause(ai_analysis)
            if root_cause:
                fields.append({
                    "title": "🤖 AI Root Cause",
                    "value": root_cause,
                    "short": False
                })

        # Add top diagnostic command
        if issue.suggested_commands:
            fields.append({
                "title": "Quick Diagnose",
                "value": f"```{issue.suggested_commands[0]}```",
                "short": False
            })

        payload = {
            "channel": self.channel,
            "username": "K8s Analyzer",
            "icon_emoji": ":kubernetes:",
            "attachments": [
                {
                    "color": color,
                    "title": f"{emoji} {issue.title}",
                    "text": issue.description[:500],
                    "fields": fields,
                    "footer": f"K8s Analyzer • {datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}",
                    "mrkdwn_in": ["text", "fields"]
                }
            ]
        }

        return self._send(payload)

    def send_cluster_summary(
        self,
        issues: List[ClusterIssue],
        ai_summary: Optional[str] = None,
        scan_duration_seconds: float = 0
    ) -> bool:
        """Send a Slack summary of the full cluster scan."""
        if not issues:
            return self._send_healthy_notification(scan_duration_seconds)

        critical = sum(1 for i in issues if i.severity == Severity.CRITICAL)
        warning = sum(1 for i in issues if i.severity == Severity.WARNING)

        # Severity-based color
        color = SEVERITY_COLOR[Severity.CRITICAL] if critical > 0 \
            else SEVERITY_COLOR[Severity.WARNING]

        # Issue list (top 5)
        issue_lines = []
        for issue in issues[:5]:
            emoji = SEVERITY_EMOJI[issue.severity]
            issue_lines.append(
                f"{emoji} *{issue.title}* "
                f"({issue.namespace})"
            )
        if len(issues) > 5:
            issue_lines.append(f"_...and {len(issues) - 5} more_")

        fields = [
            {
                "title": "🔴 Critical",
                "value": str(critical),
                "short": True
            },
            {
                "title": "🟡 Warnings",
                "value": str(warning),
                "short": True
            },
            {
                "title": "Scan Duration",
                "value": f"{scan_duration_seconds:.1f}s",
                "short": True
            },
            {
                "title": "Cluster",
                "value": self.cluster_name,
                "short": True
            },
            {
                "title": "Top Issues",
                "value": "\n".join(issue_lines),
                "short": False
            }
        ]

        if ai_summary:
            # Extract first paragraph of AI summary
            summary_brief = ai_summary.split('\n\n')[0][:400]
            fields.append({
                "title": "🤖 AI Assessment",
                "value": summary_brief,
                "short": False
            })

        payload = {
            "channel": self.channel,
            "username": "K8s Analyzer",
            "icon_emoji": ":kubernetes:",
            "text": f"*Cluster Health Report: {self.cluster_name}*",
            "attachments": [
                {
                    "color": color,
                    "title": f"Found {len(issues)} issues requiring attention",
                    "fields": fields,
                    "footer": (
                        f"K8s Analyzer • "
                        f"{datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}"
                    ),
                    "mrkdwn_in": ["text", "fields"]
                }
            ]
        }

        return self._send(payload)

    def send_resolution_alert(self, issue_title: str, namespace: str) -> bool:
        """Send a resolution notification when an issue clears."""
        payload = {
            "channel": self.channel,
            "username": "K8s Analyzer",
            "icon_emoji": ":white_check_mark:",
            "attachments": [
                {
                    "color": "#36A64F",
                    "title": f"✅ Resolved: {issue_title}",
                    "text": (
                        f"Issue in namespace *{namespace}* "
                        f"on cluster *{self.cluster_name}* "
                        f"has been resolved."
                    ),
                    "footer": (
                        f"K8s Analyzer • "
                        f"{datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}"
                    )
                }
            ]
        }
        return self._send(payload)

    def _send_healthy_notification(
        self,
        scan_duration: float
    ) -> bool:
        """Send a healthy cluster notification."""
        payload = {
            "channel": self.channel,
            "username": "K8s Analyzer",
            "icon_emoji": ":white_check_mark:",
            "attachments": [
                {
                    "color": "#36A64F",
                    "title": f"✅ Cluster Healthy: {self.cluster_name}",
                    "text": (
                        f"No issues detected. "
                        f"Scan completed in {scan_duration:.1f}s."
                    ),
                    "footer": (
                        f"K8s Analyzer • "
                        f"{datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}"
                    )
                }
            ]
        }
        return self._send(payload)

    def _send(self, payload: dict) -> bool:
        """Send payload to Slack webhook."""
        try:
            response = requests.post(
                self.webhook_url,
                data=json.dumps(payload),
                headers={"Content-Type": "application/json"},
                timeout=10
            )
            if response.status_code != 200:
                print(
                    f"Slack error: {response.status_code} "
                    f"{response.text}"
                )
                return False
            return True
        except Exception as e:
            print(f"Slack send failed: {e}")
            return False

    def _extract_root_cause(self, ai_analysis: str) -> str:
        """Extract the root cause section from AI analysis."""
        lines = ai_analysis.split('\n')
        in_root_cause = False
        result = []

        for line in lines:
            if 'root cause' in line.lower() or '**1.' in line.lower():
                in_root_cause = True
                continue
            if in_root_cause:
                if line.startswith('**') and 'root cause' not in line.lower():
                    break
                if line.strip():
                    result.append(line.strip().lstrip('- ').lstrip('* '))
                if len(result) >= 3:
                    break

        return ' '.join(result)[:300] if result else ""
```

## 🚫 Part 5: Alert Deduplication and Suppression

```
# alerts/suppressor.py
import hashlib
import json
from datetime import datetime, timezone
from typing import Dict, Optional, Tuple
from dataclasses import dataclass, field
from detector.issues import ClusterIssue, Severity

@dataclass
class AlertState:
    issue_key: str
    severity: Severity
    first_seen: datetime
    last_seen: datetime
    last_alerted: Optional[datetime]
    alert_count: int = 0
    resolved: bool = False

class AlertSuppressor:
    """
    Tracks alert state to prevent duplicate alerts.

    Rules:
    - Same issue: don't re-alert within cooldown_seconds
    - Issue resolved: send one resolution notification
    - Issue worsening (severity increase): alert immediately
    - New issues: always alert
    """

    def __init__(self, cooldown_seconds: int = 1800):
        self.cooldown_seconds = cooldown_seconds
        self.state: Dict[str, AlertState] = {}

    def should_alert(
        self,
        issue: ClusterIssue
    ) -> Tuple[bool, str]:
        """
        Determine if we should send an alert for this issue.
        Returns (should_alert, reason).
        """
        key = self._issue_key(issue)
        now = datetime.now(timezone.utc)

        if key not in self.state:
            # New issue — always alert
            self.state[key] = AlertState(
                issue_key=key,
                severity=issue.severity,
                first_seen=now,
                last_seen=now,
                last_alerted=None,
                alert_count=0
            )
            return True, "new_issue"

        state = self.state[key]
        state.last_seen = now
        state.resolved = False

        # Severity escalation — alert immediately
        severity_order = {
            Severity.INFO: 0,
            Severity.WARNING: 1,
            Severity.CRITICAL: 2
        }
        if severity_order[issue.severity] > \
                severity_order[state.severity]:
            state.severity = issue.severity
            return True, "severity_escalation"

        # Cooldown check
        if state.last_alerted is None:
            return True, "first_alert"

        elapsed = (now - state.last_alerted).total_seconds()
        if elapsed >= self.cooldown_seconds:
            return True, f"cooldown_expired_{elapsed:.0f}s"

        return False, f"suppressed_{elapsed:.0f}s_of_{self.cooldown_seconds}s"

    def mark_alerted(self, issue: ClusterIssue):
        """Record that an alert was sent."""
        key = self._issue_key(issue)
        now = datetime.now(timezone.utc)
        if key in self.state:
            self.state[key].last_alerted = now
            self.state[key].alert_count += 1

    def get_resolved_issues(
        self,
        current_issues: list
    ) -> list:
        """
        Find issues that were previously seen but are now gone.
        Returns list of (issue_key, alert_state) for resolved issues.
        """
        current_keys = {
            self._issue_key(i) for i in current_issues
        }
        resolved = []

        for key, state in self.state.items():
            if key not in current_keys and \
                    not state.resolved and \
                    state.alert_count > 0:
                state.resolved = True
                resolved.append((key, state))

        return resolved

    def get_state_summary(self) -> dict:
        """Get a summary of current alert state."""
        active = {
            k: v for k, v in self.state.items()
            if not v.resolved
        }
        resolved = {
            k: v for k, v in self.state.items()
            if v.resolved
        }
        return {
            "active_issues": len(active),
            "resolved_issues": len(resolved),
            "total_alerts_sent": sum(
                v.alert_count for v in self.state.values()
            )
        }

    def _issue_key(self, issue: ClusterIssue) -> str:
        """Generate a stable key for an issue."""
        raw = f"{issue.namespace}/{issue.affected_resource}/{issue.category}"
        return hashlib.md5(raw.encode()).hexdigest()[:12]
```

## 📊 Part 6: Prometheus Metrics

```
# metrics/prometheus.py
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    start_http_server, CollectorRegistry
)
from detector.issues import ClusterIssue, Severity
from typing import List

# ── Define metrics ───────────────────────────────────────────────

registry = CollectorRegistry()

# Scan metrics
SCAN_TOTAL = Counter(
    'k8s_analyzer_scans_total',
    'Total number of cluster scans completed',
    ['cluster', 'status'],
    registry=registry
)

SCAN_DURATION = Histogram(
    'k8s_analyzer_scan_duration_seconds',
    'Time taken to complete a cluster scan',
    ['cluster'],
    buckets=[1, 5, 10, 30, 60, 120],
    registry=registry
)

# Issue metrics
ISSUES_DETECTED = Gauge(
    'k8s_analyzer_issues_detected',
    'Current number of issues detected',
    ['cluster', 'severity', 'category'],
    registry=registry
)

ISSUES_TOTAL = Counter(
    'k8s_analyzer_issues_total',
    'Total issues ever detected',
    ['cluster', 'severity', 'category', 'reason'],
    registry=registry
)

# Alert metrics
ALERTS_SENT = Counter(
    'k8s_analyzer_alerts_sent_total',
    'Total alerts sent',
    ['cluster', 'severity', 'channel'],
    registry=registry
)

ALERTS_SUPPRESSED = Counter(
    'k8s_analyzer_alerts_suppressed_total',
    'Total alerts suppressed by deduplication',
    ['cluster', 'reason'],
    registry=registry
)

# AI metrics
AI_REQUESTS = Counter(
    'k8s_analyzer_ai_requests_total',
    'Total AI API calls made',
    ['cluster', 'model', 'status'],
    registry=registry
)

AI_LATENCY = Histogram(
    'k8s_analyzer_ai_latency_seconds',
    'AI API call latency',
    ['cluster', 'model'],
    buckets=[0.5, 1, 2, 5, 10, 30],
    registry=registry
)

AI_TOKENS_USED = Counter(
    'k8s_analyzer_ai_tokens_total',
    'Total tokens used in AI API calls',
    ['cluster', 'model', 'type'],
    registry=registry
)

# Cluster health metrics
CLUSTER_NODES = Gauge(
    'k8s_analyzer_cluster_nodes',
    'Number of nodes in the cluster',
    ['cluster', 'status'],
    registry=registry
)

CLUSTER_PODS = Gauge(
    'k8s_analyzer_cluster_pods',
    'Number of pods in the cluster',
    ['cluster', 'namespace', 'status'],
    registry=registry
)

class MetricsCollector:
    def __init__(self, cluster_name: str, port: int = 9090):
        self.cluster = cluster_name
        self.port = port

    def start_server(self):
        """Start the Prometheus metrics HTTP server."""
        start_http_server(self.port, registry=registry)
        print(f"Metrics server started on port {self.port}")

    def record_scan(
        self,
        status: str,
        duration_seconds: float
    ):
        """Record scan completion metrics."""
        SCAN_TOTAL.labels(
            cluster=self.cluster,
            status=status
        ).inc()

        SCAN_DURATION.labels(
            cluster=self.cluster
        ).observe(duration_seconds)

    def record_issues(self, issues: List[ClusterIssue]):
        """Update current issue gauges."""
        # Reset all gauges (we'll set new values)
        for sev in Severity:
            for cat in ["Pod", "Node", "Deployment", "Other"]:
                ISSUES_DETECTED.labels(
                    cluster=self.cluster,
                    severity=sev.value,
                    category=cat
                ).set(0)

        # Set current values
        for issue in issues:
            ISSUES_DETECTED.labels(
                cluster=self.cluster,
                severity=issue.severity.value,
                category=issue.category
            ).inc()

            ISSUES_TOTAL.labels(
                cluster=self.cluster,
                severity=issue.severity.value,
                category=issue.category,
                reason=issue.raw_data.get("reason", "unknown")
            ).inc()

    def record_alert(
        self,
        severity: Severity,
        channel: str,
        suppressed: bool = False,
        suppress_reason: str = ""
    ):
        """Record alert metrics."""
        if suppressed:
            ALERTS_SUPPRESSED.labels(
                cluster=self.cluster,
                reason=suppress_reason
            ).inc()
        else:
            ALERTS_SENT.labels(
                cluster=self.cluster,
                severity=severity.value,
                channel=channel
            ).inc()

    def record_ai_call(
        self,
        model: str,
        status: str,
        latency: float,
        prompt_tokens: int = 0,
        completion_tokens: int = 0
    ):
        """Record AI API usage metrics."""
        AI_REQUESTS.labels(
            cluster=self.cluster,
            model=model,
            status=status
        ).inc()

        AI_LATENCY.labels(
            cluster=self.cluster,
            model=model
        ).observe(latency)

        if prompt_tokens:
            AI_TOKENS_USED.labels(
                cluster=self.cluster,
                model=model,
                type="prompt"
            ).inc(prompt_tokens)

        if completion_tokens:
            AI_TOKENS_USED.labels(
                cluster=self.cluster,
                model=model,
                type="completion"
            ).inc(completion_tokens)

    def record_cluster_state(
        self,
        node_count: int,
        not_ready_count: int,
        pod_counts: dict
    ):
        """Record cluster state metrics."""
        CLUSTER_NODES.labels(
            cluster=self.cluster,
            status="ready"
        ).set(node_count - not_ready_count)

        CLUSTER_NODES.labels(
            cluster=self.cluster,
            status="not_ready"
        ).set(not_ready_count)
```

## 🔄 Part 7: The Operator Controller Loop

```
# operator/controller.py
import time
import logging
import threading
from datetime import datetime, timezone
from typing import Optional

from kubernetes import client, config
from config.settings import settings
from collector.pods import collect_pod_issues
from collector.nodes import collect_node_status
from detector.issues import detect_issues, Severity
from analyzer.claude import ClusterAnalyzer
from analyzer.cache import AnalysisCache
from alerts.slack import SlackAlerter
from alerts.suppressor import AlertSuppressor
from metrics.prometheus import MetricsCollector

logger = logging.getLogger("k8s-analyzer-operator")

class AnalyzerController:
    """
    The Operator controller — runs the reconciliation loop.
    Continuously scans the cluster and reacts to issues.
    """

    def __init__(self):
        # K8s clients
        try:
            config.load_incluster_config()
            logger.info("Loaded in-cluster config")
        except:
            config.load_kube_config()
            logger.info("Loaded local kubeconfig")

        self.v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()

        # Components
        self.analyzer = ClusterAnalyzer(
            api_key=settings.anthropic_api_key
        ) if settings.ai_enabled and settings.anthropic_api_key else None

        self.cache = AnalysisCache(
            ttl_seconds=settings.ai_cache_ttl_seconds
        )

        self.suppressor = AlertSuppressor(
            cooldown_seconds=settings.alert_cooldown_seconds
        )

        self.alerter = SlackAlerter(
            webhook_url=settings.slack_webhook_url,
            channel=settings.slack_channel,
            cluster_name=settings.cluster_name
        ) if settings.slack_webhook_url else None

        self.metrics = MetricsCollector(
            cluster_name=settings.cluster_name,
            port=settings.metrics_port
        )

        self._running = False
        self._scan_count = 0

    def start(self):
        """Start the operator — blocking."""
        logger.info(
            f"Starting K8s Analyzer Operator "
            f"(cluster: {settings.cluster_name}, "
            f"interval: {settings.scan_interval_seconds}s)"
        )

        # Start metrics server in background thread
        if settings.metrics_enabled:
            metrics_thread = threading.Thread(
                target=self.metrics.start_server,
                daemon=True
            )
            metrics_thread.start()

        self._running = True
        self._run_loop()

    def stop(self):
        """Stop the operator gracefully."""
        logger.info("Stopping operator...")
        self._running = False

    def _run_loop(self):
        """Main reconciliation loop."""
        while self._running:
            try:
                self._reconcile()
            except Exception as e:
                logger.error(f"Reconciliation error: {e}", exc_info=True)
                self.metrics.record_scan("error", 0)

            logger.info(
                f"Scan {self._scan_count} complete. "
                f"Next scan in {settings.scan_interval_seconds}s"
            )
            time.sleep(settings.scan_interval_seconds)

    def _reconcile(self):
        """One full reconciliation cycle."""
        start_time = time.time()
        self._scan_count += 1
        logger.info(f"Starting scan #{self._scan_count}")

        # ── Collect ──────────────────────────────────────────────
        logger.info("Collecting pod data...")
        pod_issues = collect_pod_issues(self.v1)

        logger.info("Collecting node data...")
        node_statuses = collect_node_status(self.v1)

        # ── Detect ───────────────────────────────────────────────
        logger.info("Detecting issues...")
        all_issues = detect_issues(pod_issues, node_statuses)

        # Filter by configured namespaces
        if settings.namespaces:
            all_issues = [
                i for i in all_issues
                if i.namespace in settings.namespaces
                or i.namespace == "cluster-scoped"
            ]

        filtered_issues = all_issues[:settings.max_issues_per_scan]

        logger.info(
            f"Found {len(all_issues)} total issues, "
            f"processing top {len(filtered_issues)}"
        )

        # ── Update metrics ───────────────────────────────────────
        self.metrics.record_issues(all_issues)

        not_ready_nodes = sum(
            1 for n in node_statuses if not n.ready
        )
        self.metrics.record_cluster_state(
            node_count=len(node_statuses),
            not_ready_count=not_ready_nodes,
            pod_counts={}
        )

        # ── Check for resolutions ─────────────────────────────────
        resolved = self.suppressor.get_resolved_issues(filtered_issues)
        for issue_key, state in resolved:
            logger.info(f"Issue resolved: {issue_key}")
            if self.alerter:
                self.alerter.send_resolution_alert(
                    issue_title=issue_key,
                    namespace="unknown"
                )

        # ── AI Analysis + Alerting ───────────────────────────────
        ai_summary = None
        ai_analyses = {}

        if self.analyzer and filtered_issues:
            # Get cluster summary
            cluster_info = {
                "cluster": settings.cluster_name,
                "node_count": len(node_statuses),
                "critical_issues": sum(
                    1 for i in all_issues
                    if i.severity == Severity.CRITICAL
                ),
                "warning_issues": sum(
                    1 for i in all_issues
                    if i.severity == Severity.WARNING
                )
            }

            ai_call_start = time.time()
            try:
                ai_summary = self.analyzer.analyze_cluster_summary(
                    filtered_issues,
                    cluster_info
                )
                self.metrics.record_ai_call(
                    model=settings.ai_model,
                    status="success",
                    latency=time.time() - ai_call_start
                )
            except Exception as e:
                logger.error(f"AI summary failed: {e}")
                self.metrics.record_ai_call(
                    model=settings.ai_model,
                    status="error",
                    latency=time.time() - ai_call_start
                )

            # Analyze individual issues
            ai_budget = settings.ai_max_per_scan
            for issue in filtered_issues:
                if ai_budget <= 0:
                    break

                # Check cache first
                cached = self.cache.get(issue)
                if cached:
                    ai_analyses[id(issue)] = cached
                    continue

                ai_call_start = time.time()
                try:
                    analysis = self.analyzer.analyze_issue(issue)
                    self.cache.set(issue, analysis)
                    ai_analyses[id(issue)] = analysis
                    ai_budget -= 1

                    self.metrics.record_ai_call(
                        model=settings.ai_model,
                        status="success",
                        latency=time.time() - ai_call_start
                    )
                except Exception as e:
                    logger.error(f"AI issue analysis failed: {e}")
                    self.metrics.record_ai_call(
                        model=settings.ai_model,
                        status="error",
                        latency=time.time() - ai_call_start
                    )

        # ── Send alerts ──────────────────────────────────────────
        if self.alerter:
            # Send cluster summary alert
            duration = time.time() - start_time
            self.alerter.send_cluster_summary(
                issues=filtered_issues,
                ai_summary=ai_summary,
                scan_duration_seconds=duration
            )

            # Send individual alerts for each issue
            for issue in filtered_issues:
                if issue.severity.value not in settings.alert_on_severity:
                    continue

                should_alert, reason = self.suppressor.should_alert(issue)

                if should_alert:
                    analysis = ai_analyses.get(id(issue))
                    success = self.alerter.send_issue_alert(
                        issue=issue,
                        ai_analysis=analysis
                    )
                    if success:
                        self.suppressor.mark_alerted(issue)
                        self.metrics.record_alert(
                            severity=issue.severity,
                            channel=settings.slack_channel
                        )
                        logger.info(
                            f"Alert sent: {issue.title} "
                            f"(reason: {reason})"
                        )
                else:
                    self.metrics.record_alert(
                        severity=issue.severity,
                        channel=settings.slack_channel,
                        suppressed=True,
                        suppress_reason=reason
                    )
                    logger.debug(
                        f"Alert suppressed: {issue.title} "
                        f"(reason: {reason})"
                    )

        # ── Record scan metrics ───────────────────────────────────
        scan_duration = time.time() - start_time
        self.metrics.record_scan("success", scan_duration)
        logger.info(f"Scan #{self._scan_count} completed in {scan_duration:.2f}s")
```

## 💾 Part 8: Analysis Cache

```
# analyzer/cache.py
import hashlib
import json
from datetime import datetime, timezone
from typing import Optional, Dict
from dataclasses import dataclass
from detector.issues import ClusterIssue

@dataclass
class CacheEntry:
    analysis: str
    created_at: datetime
    ttl_seconds: int

    def is_valid(self) -> bool:
        age = (datetime.now(timezone.utc) - self.created_at).total_seconds()
        return age < self.ttl_seconds

class AnalysisCache:
    """
    In-memory cache for AI analysis results.
    Prevents redundant API calls for the same issue.
    """

    def __init__(self, ttl_seconds: int = 3600):
        self.ttl_seconds = ttl_seconds
        self._cache: Dict[str, CacheEntry] = {}

    def get(self, issue: ClusterIssue) -> Optional[str]:
        """Get cached analysis if available and not expired."""
        key = self._cache_key(issue)
        entry = self._cache.get(key)

        if entry and entry.is_valid():
            return entry.analysis

        # Remove expired entry
        if entry:
            del self._cache[key]

        return None

    def set(self, issue: ClusterIssue, analysis: str):
        """Cache an analysis result."""
        key = self._cache_key(issue)
        self._cache[key] = CacheEntry(
            analysis=analysis,
            created_at=datetime.now(timezone.utc),
            ttl_seconds=self.ttl_seconds
        )

    def invalidate(self, issue: ClusterIssue):
        """Remove a specific entry from cache."""
        key = self._cache_key(issue)
        self._cache.pop(key, None)

    def clear(self):
        """Clear all cache entries."""
        self._cache.clear()

    def stats(self) -> dict:
        """Return cache statistics."""
        valid = sum(1 for e in self._cache.values() if e.is_valid())
        return {
            "total_entries": len(self._cache),
            "valid_entries": valid,
            "expired_entries": len(self._cache) - valid
        }

    def _cache_key(self, issue: ClusterIssue) -> str:
        """
        Generate cache key from issue fingerprint.
        Same issue + same logs = same key = cache hit.
        """
        fingerprint = {
            "namespace": issue.namespace,
            "resource": issue.affected_resource,
            "category": issue.category,
            "title": issue.title,
            # Include first 200 chars of logs in fingerprint
            "log_prefix": (
                issue.raw_data.get("logs", "")[:200]
                if issue.raw_data else ""
            )
        }
        return hashlib.sha256(
            json.dumps(fingerprint, sort_keys=True).encode()
        ).hexdigest()[:16]
```

## 🚀 Part 9: Operator Entry Point

```
# main.py (operator mode)
import click
import logging
import signal
import sys
import os
from config.settings import settings

logging.basicConfig(
    level=getattr(logging, os.environ.get("LOG_LEVEL", "INFO")),
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger("k8s-analyzer")

@click.group()
def cli():
    """K8s Intelligent Cluster Analyzer"""
    pass

@cli.command()
def operator():
    """Run as a Kubernetes Operator (continuous monitoring)."""
    from operator.controller import AnalyzerController

    controller = AnalyzerController()

    # Graceful shutdown
    def handle_shutdown(signum, frame):
        logger.info(f"Received signal {signum}, shutting down...")
        controller.stop()
        sys.exit(0)

    signal.signal(signal.SIGTERM, handle_shutdown)
    signal.signal(signal.SIGINT, handle_shutdown)

    controller.start()

@cli.command()
@click.option("--namespace", "-n", default=None)
@click.option("--severity", default="ALL",
              type=click.Choice(["CRITICAL", "WARNING", "INFO", "ALL"]))
@click.option("--max-issues", default=5)
@click.option("--explain", default=None)
@click.option("--runbook", default=None)
def scan(namespace, severity, max_issues, explain, runbook):
    """Run a one-shot cluster scan (original Day 28 mode)."""
    # Import the Day 28 scan logic
    from main_scan import run_scan
    run_scan(namespace, severity, max_issues, explain, runbook)

@cli.command()
def status():
    """Show operator status and alert suppressor state."""
    from operator.controller import AnalyzerController
    controller = AnalyzerController()
    state = controller.suppressor.get_state_summary()
    cache_stats = controller.cache.stats()

    click.echo(f"Alert state: {state}")
    click.echo(f"Cache stats: {cache_stats}")

if __name__ == "__main__":
    cli()
```

## 📦 Part 10: Full Kubernetes Deployment

```
# k8s/operator-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-analyzer
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-analyzer
  namespace: k8s-analyzer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-analyzer
rules:
- apiGroups: [""]
  resources:
  - pods
  - pods/log
  - nodes
  - namespaces
  - events
  - services
  - endpoints
  - configmaps
  - persistentvolumes
  - persistentvolumeclaims
  - resourcequotas
  - limitranges
  verbs: [get, list, watch]
- apiGroups: [apps]
  resources:
  - deployments
  - statefulsets
  - daemonsets
  - replicasets
  verbs: [get, list, watch]
- apiGroups: [batch]
  resources: [jobs, cronjobs]
  verbs: [get, list, watch]
- apiGroups: [networking.k8s.io]
  resources: [ingresses, networkpolicies]
  verbs: [get, list, watch]
- apiGroups: [rbac.authorization.k8s.io]
  resources: [roles, rolebindings, clusterroles, clusterrolebindings]
  verbs: [get, list, watch]
- apiGroups: [coordination.k8s.io]
  resources: [leases]
  verbs: [get, list, watch, create, update, patch]   # for leader election
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-analyzer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8s-analyzer
subjects:
- kind: ServiceAccount
  name: k8s-analyzer
  namespace: k8s-analyzer
---
apiVersion: v1
kind: Secret
metadata:
  name: analyzer-secrets
  namespace: k8s-analyzer
type: Opaque
stringData:
  anthropic-api-key: "your-anthropic-api-key-here"
  slack-webhook-url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: analyzer-config
  namespace: k8s-analyzer
data:
  CLUSTER_NAME: "k8s-mastery"
  SCAN_INTERVAL_SECONDS: "300"
  AI_ENABLED: "true"
  AI_MAX_PER_SCAN: "3"
  AI_CACHE_TTL_SECONDS: "3600"
  ALERT_COOLDOWN_SECONDS: "1800"
  ALERT_ON_SEVERITY: '["CRITICAL", "WARNING"]'
  EXCLUDED_NAMESPACES: '["kube-system", "kube-public", "kube-node-lease"]'
  METRICS_PORT: "9090"
  METRICS_ENABLED: "true"
  LOG_LEVEL: "INFO"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-analyzer
  namespace: k8s-analyzer
  labels:
    app: k8s-analyzer
spec:
  replicas: 1                   # single replica — leader election handles HA
  selector:
    matchLabels:
      app: k8s-analyzer
  strategy:
    type: Recreate              # stateful — only one should run at a time
  template:
    metadata:
      labels:
        app: k8s-analyzer
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: k8s-analyzer

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault

      containers:
      - name: analyzer
        image: ghcr.io/youruser/k8s-analyzer:latest
        command: ["python", "main.py", "operator"]
        ports:
        - containerPort: 9090
          name: metrics
        envFrom:
        - configMapRef:
            name: analyzer-config
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: analyzer-secrets
              key: anthropic-api-key
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: analyzer-secrets
              key: slack-webhook-url
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        livenessProbe:
          httpGet:
            path: /metrics
            port: 9090
          initialDelaySeconds: 30
          periodSeconds: 30
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /metrics
            port: 9090
          initialDelaySeconds: 15
          periodSeconds: 10

      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-analyzer
  namespace: k8s-analyzer
  labels:
    app: k8s-analyzer
spec:
  selector:
    app: k8s-analyzer
  ports:
  - port: 9090
    targetPort: 9090
    name: metrics
---
# ServiceMonitor for Prometheus scraping
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: k8s-analyzer
  namespace: k8s-analyzer
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: k8s-analyzer
  endpoints:
  - port: metrics
    interval: 60s
    path: /metrics
```

## 🖥️ Part 11: Hands-On Exercises

**Exercise 1: Run the full operator locally**

```
# Set up environment
cd k8s-analyzer
pip install anthropic kubernetes rich click pydantic \
            prometheus-client requests python-dotenv

# Create .env file
cat <<EOF > .env
ANTHROPIC_API_KEY=your-key-here
CLUSTER_NAME=k8s-mastery
SCAN_INTERVAL_SECONDS=60
AI_ENABLED=true
AI_MAX_PER_SCAN=3
ALERT_COOLDOWN_SECONDS=300
METRICS_ENABLED=true
METRICS_PORT=9090
LOG_LEVEL=INFO
EOF

# Create some broken pods first
kubectl run crash-test \
  --image=busybox \
  -- sh -c "sleep 5 && exit 1"

kubectl run bad-img \
  --image=nginx:v999-does-not-exist

# Run the operator
python main.py operator
# Watch it scan every 60 seconds
# See it detect the broken pods
# See metrics exposed at localhost:9090/metrics

# In another terminal — verify metrics
curl -s http://localhost:9090/metrics \
  | grep k8s_analyzer | head -20
```

**Exercise 2: Test alert suppression**

```
# Enable suppressor test mode
# Edit .env: ALERT_COOLDOWN_SECONDS=10

# Run two consecutive scans manually
python main.py scan --severity CRITICAL

# Wait 5 seconds (less than cooldown)
sleep 5

# Second scan — same issues should be suppressed
python main.py scan --severity CRITICAL

# Wait for cooldown
sleep 10

# Third scan — should alert again
python main.py scan --severity CRITICAL

# Check suppressor state
python main.py status
```

**Exercise 3: Deploy to Kubernetes**

```
# Build image
docker build -t k8s-analyzer:latest .

# For kind — load directly (no registry needed)
kind load docker-image k8s-analyzer:latest \
  --name k8s-mastery

# Update image reference in deployment
sed -i 's|ghcr.io/youruser/k8s-analyzer:latest|k8s-analyzer:latest|' \
  k8s/operator-deployment.yaml

# Create secret (edit with your keys first)
kubectl create namespace k8s-analyzer

kubectl create secret generic analyzer-secrets \
  -n k8s-analyzer \
  --from-literal=anthropic-api-key="$ANTHROPIC_API_KEY" \
  --from-literal=slack-webhook-url="https://hooks.slack.com/..."

# Deploy
kubectl apply -f k8s/operator-deployment.yaml

# Watch it start
kubectl get pods -n k8s-analyzer -w

# Read operator logs
kubectl logs -n k8s-analyzer \
  -l app=k8s-analyzer \
  -f

# Verify metrics endpoint
kubectl port-forward -n k8s-analyzer \
  svc/k8s-analyzer 9090:9090 &

curl -s http://localhost:9090/metrics \
  | grep k8s_analyzer | head -20
```

**Exercise 4: Prometheus + Grafana dashboard**

```
# Create PrometheusRule for the analyzer
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: k8s-analyzer-alerts
  namespace: k8s-analyzer
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: k8s-analyzer
    rules:
    - alert: CriticalIssuesDetected
      expr: |
        sum(k8s_analyzer_issues_detected{severity="CRITICAL"}) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "K8s cluster has critical issues"
        description: >
          {{ \$value }} critical issues detected
          on cluster {{ \$labels.cluster }}

    - alert: AnalyzerDown
      expr: |
        absent(k8s_analyzer_scans_total) == 1
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "K8s Analyzer has stopped reporting"
EOF

# Import Grafana dashboard
# Create a dashboard JSON and import via:
# kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Dashboard → Import → paste JSON below

cat <<'EOF'
{
  "title": "K8s Analyzer Dashboard",
  "panels": [
    {
      "title": "Active Critical Issues",
      "type": "stat",
      "targets": [{
        "expr": "sum(k8s_analyzer_issues_detected{severity='CRITICAL'})"
      }]
    },
    {
      "title": "Active Warning Issues",
      "type": "stat",
      "targets": [{
        "expr": "sum(k8s_analyzer_issues_detected{severity='WARNING'})"
      }]
    },
    {
      "title": "Issues Over Time",
      "type": "timeseries",
      "targets": [{
        "expr": "k8s_analyzer_issues_detected",
        "legendFormat": "{{severity}} - {{category}}"
      }]
    },
    {
      "title": "Scan Duration",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.99, rate(k8s_analyzer_scan_duration_seconds_bucket[5m]))"
      }]
    },
    {
      "title": "AI API Calls",
      "type": "timeseries",
      "targets": [{
        "expr": "rate(k8s_analyzer_ai_requests_total[5m])",
        "legendFormat": "{{status}}"
      }]
    },
    {
      "title": "Alerts Sent vs Suppressed",
      "type": "timeseries",
      "targets": [
        {
          "expr": "rate(k8s_analyzer_alerts_sent_total[5m])",
          "legendFormat": "sent"
        },
        {
          "expr": "rate(k8s_analyzer_alerts_suppressed_total[5m])",
          "legendFormat": "suppressed"
        }
      ]
    }
  ]
}
EOF
```

## 🎯 Part 12: Interview Questions — Day 26

**Q1: How do you prevent alert fatigue in a Kubernetes monitoring system?**

Three mechanisms working together. Deduplication: hash the issue signature (namespace + resource + reason) — the same issue only alerts once per cooldown period regardless of how many scans detect it. The suppressor tracks state across scans and enforces the cooldown. Severity escalation bypass: if an issue worsens (INFO → WARNING → CRITICAL), send a new alert immediately — this is a meaningful change. Resolution notifications: close the loop — when an issue clears, send one resolution alert so on-call knows it's done without checking manually. Beyond deduplication: group related alerts into a single Slack message (one cluster summary + individual breakdowns), not one Slack message per pod. Use Prometheus Alertmanager's inhibition rules to suppress lower-severity alerts when a higher-severity root cause alert is firing — if a node is NotReady, suppress all PodPending alerts on that node since they're downstream effects.

**Q2: How do you make the analyzer work safely without leaking sensitive data to the Claude API?**

The raw_data field collects pod logs — which might contain tokens, passwords, or PII. Three mitigations. Data minimization in collection: only collect the last 50 lines of logs (configurable), never collect environment variables, never collect Secret values or ConfigMap data. Scrubbing before AI calls: in _build_issue_context, run the log content through a regex scrubber that replaces common patterns: AWS keys, JWT tokens, passwords, IP addresses, email addresses. Only structured data (status, reason, event messages) goes to the API without scrubbing — these are K8s system messages, not user data. Network controls: the operator's NetworkPolicy allows egress only to api.anthropic.com — a compromised pod can't send data anywhere else. For extra-sensitive environments: replace the Claude API call with a local Ollama instance running Llama — zero data leaves the cluster.

**Q3: The operator is deployed with replicas: 1. How do you make it highly available?**

Leader election via the coordination.k8s.io Lease API. Deploy with replicas: 2 (or more). Each instance races to acquire a Lease object. The winner is the leader and runs the reconciliation loop. Losers watch the Lease and stand by. If the leader dies, its Lease expires (after leaseDuration, default 15s), and a standby wins the next election and takes over. The ClusterRole already includes Lease permissions for this reason. In the controller: use k8s.io/client-go/tools/leaderelection package (Go) or equivalent Python implementation. The standby instances run health checks and expose metrics but don't scan. This gives HA with no duplicate scans or duplicate alerts. Also run pods on different nodes via pod anti-affinity to prevent a node failure from taking out both replicas.

# Day 25 — AI-Powered Kubernetes Tool: Intelligent Cluster Analyzer

This is where everything converges. Today you build a production-grade tool that combines your Kubernetes mastery with AI — an intelligent cluster analyzer that watches your cluster, detects issues, and uses Claude to diagnose and suggest fixes in plain English. This is your capstone project.

## 🧠 Part 1: What You're Building

```
K8s Cluster
    ↓  (kubectl / K8s API)
Data Collector     — gathers pods, events, metrics, logs
    ↓
Issue Detector     — identifies anomalies: CrashLoops, Pending pods,
                     OOMKills, resource pressure, RBAC errors
    ↓
Claude API         — analyzes issues with full cluster context,
                     explains root cause in plain English,
                     suggests exact kubectl commands to fix
    ↓
Output             — rich terminal UI + optional Slack/webhook alerts
```

**Real use cases:**

- On-call engineer gets a 3am alert — tool runs and tells them exactly what's wrong and how to fix it
- Junior engineer faces an unfamiliar error — tool explains it at their level
- Platform team audit — tool scans entire cluster for misconfigurations

## 🏗️ Part 2: Architecture

```
k8s-analyzer/
├── main.py                  — CLI entry point
├── collector/
│   ├── __init__.py
│   ├── pods.py              — pod status, events, logs
│   ├── nodes.py             — node conditions, resources
│   ├── workloads.py         — deployments, statefulsets
│   └── cluster.py           — namespaces, RBAC, quotas
├── detector/
│   ├── __init__.py
│   ├── issues.py            — issue detection logic
│   └── severity.py          — CRITICAL/WARNING/INFO classification
├── analyzer/
│   ├── __init__.py
│   └── claude.py            — Claude API integration
├── output/
│   ├── __init__.py
│   ├── terminal.py          — rich terminal output
│   └── slack.py             — Slack webhook alerts
├── requirements.txt
└── Dockerfile
```

## 💻 Part 3: Build It — Full Implementation

**requirements.txt**

```
anthropic>=0.25.0
kubernetes>=29.0.0
rich>=13.0.0
click>=8.0.0
pydantic>=2.0.0
requests>=2.31.0
python-dotenv>=1.0.0
```

**collector/pods.py**

```
# collector/pods.py
from kubernetes import client, config
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime, timezone

@dataclass
class PodIssue:
    namespace: str
    name: str
    status: str
    reason: str
    message: str
    restart_count: int
    age_minutes: float
    node: str
    logs: Optional[str] = None
    events: Optional[List[str]] = None

def collect_pod_issues(v1: client.CoreV1Api) -> List[PodIssue]:
    """Collect all pods with issues across all namespaces."""
    issues = []

    pods = v1.list_pod_for_all_namespaces(watch=False)

    for pod in pods.items:
        name = pod.metadata.name
        namespace = pod.metadata.namespace
        phase = pod.status.phase or "Unknown"
        node = pod.spec.node_name or "unscheduled"

        # Skip system namespaces for cleaner output
        if namespace in ["kube-system", "kube-public"]:
            continue

        # Calculate age
        creation_time = pod.metadata.creation_timestamp
        now = datetime.now(timezone.utc)
        age_minutes = (now - creation_time).total_seconds() / 60

        # Collect restart counts and container states
        restart_count = 0
        container_issue = None

        if pod.status.container_statuses:
            for cs in pod.status.container_statuses:
                restart_count += cs.restart_count or 0

                # Waiting containers
                if cs.state and cs.state.waiting:
                    waiting = cs.state.waiting
                    reason = waiting.reason or "Unknown"

                    if reason in [
                        "CrashLoopBackOff",
                        "ImagePullBackOff",
                        "ErrImagePull",
                        "OOMKilled",
                        "Error",
                        "CreateContainerError",
                        "InvalidImageName",
                    ]:
                        container_issue = (reason, waiting.message or "")

                # Terminated containers
                if cs.last_state and cs.last_state.terminated:
                    term = cs.last_state.terminated
                    if term.exit_code and term.exit_code != 0:
                        reason = term.reason or f"ExitCode{term.exit_code}"
                        if reason == "OOMKilled":
                            container_issue = ("OOMKilled", f"Exit code: {term.exit_code}")

        # Pending pods
        if phase == "Pending":
            issues.append(PodIssue(
                namespace=namespace,
                name=name,
                status="Pending",
                reason="PodNotScheduled",
                message="Pod cannot be scheduled",
                restart_count=restart_count,
                age_minutes=age_minutes,
                node=node,
                events=_get_pod_events(v1, namespace, name)
            ))

        # Container issues
        elif container_issue:
            reason, message = container_issue
            logs = _get_pod_logs(v1, namespace, name)
            issues.append(PodIssue(
                namespace=namespace,
                name=name,
                status=phase,
                reason=reason,
                message=message,
                restart_count=restart_count,
                age_minutes=age_minutes,
                node=node,
                logs=logs,
                events=_get_pod_events(v1, namespace, name)
            ))

        # High restart count (but currently running)
        elif restart_count > 5 and phase == "Running":
            logs = _get_pod_logs(v1, namespace, name)
            issues.append(PodIssue(
                namespace=namespace,
                name=name,
                status="Running (unstable)",
                reason="HighRestartCount",
                message=f"Pod has restarted {restart_count} times",
                restart_count=restart_count,
                age_minutes=age_minutes,
                node=node,
                logs=logs,
                events=_get_pod_events(v1, namespace, name)
            ))

    return issues

def _get_pod_logs(
    v1: client.CoreV1Api,
    namespace: str,
    name: str,
    lines: int = 50
) -> Optional[str]:
    """Get recent logs from pod — try current then previous."""
    for previous in [False, True]:
        try:
            logs = v1.read_namespaced_pod_log(
                name=name,
                namespace=namespace,
                tail_lines=lines,
                previous=previous,
                timestamps=True
            )
            if logs:
                label = "(previous container)" if previous else "(current)"
                return f"{label}\n{logs}"
        except Exception:
            continue
    return None

def _get_pod_events(
    v1: client.CoreV1Api,
    namespace: str,
    pod_name: str
) -> List[str]:
    """Get recent events for a pod."""
    try:
        events = v1.list_namespaced_event(
            namespace=namespace,
            field_selector=f"involvedObject.name={pod_name}"
        )
        result = []
        for event in sorted(
            events.items,
            key=lambda e: e.last_timestamp or e.event_time or datetime.min.replace(tzinfo=timezone.utc)
        )[-10:]:
            result.append(
                f"[{event.type}] {event.reason}: {event.message}"
            )
        return result
    except Exception:
        return []
```

**collector/nodes.py**

```
# collector/nodes.py
from kubernetes import client
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class NodeStatus:
    name: str
    ready: bool
    conditions: Dict[str, str]
    cpu_capacity: str
    memory_capacity: str
    cpu_allocatable: str
    memory_allocatable: str
    pod_count: int
    taints: List[str]
    issues: List[str]

def collect_node_status(v1: client.CoreV1Api) -> List[NodeStatus]:
    """Collect status and issues for all nodes."""
    nodes = v1.list_node(watch=False)
    result = []

    for node in nodes.items:
        name = node.metadata.name
        conditions = {}
        issues = []
        ready = False

        # Process conditions
        if node.status.conditions:
            for cond in node.status.conditions:
                conditions[cond.type] = cond.status
                if cond.type == "Ready":
                    ready = (cond.status == "True")
                    if cond.status != "True":
                        issues.append(
                            f"Node NotReady: {cond.reason} - {cond.message}"
                        )
                elif cond.status == "True" and cond.type in [
                    "MemoryPressure", "DiskPressure", "PIDPressure"
                ]:
                    issues.append(
                        f"{cond.type}: {cond.message}"
                    )

        # Taints
        taints = []
        if node.spec.taints:
            for taint in node.spec.taints:
                taints.append(f"{taint.key}={taint.value}:{taint.effect}")

        # Resources
        capacity = node.status.capacity or {}
        allocatable = node.status.allocatable or {}

        # Count pods on this node
        pods = v1.list_pod_for_all_namespaces(
            field_selector=f"spec.nodeName={name}",
            watch=False
        )
        pod_count = len(pods.items)
        max_pods = int(allocatable.get("pods", "110"))

        if pod_count > max_pods * 0.9:
            issues.append(
                f"Node near pod limit: {pod_count}/{max_pods} pods"
            )

        result.append(NodeStatus(
            name=name,
            ready=ready,
            conditions=conditions,
            cpu_capacity=capacity.get("cpu", "unknown"),
            memory_capacity=capacity.get("memory", "unknown"),
            cpu_allocatable=allocatable.get("cpu", "unknown"),
            memory_allocatable=allocatable.get("memory", "unknown"),
            pod_count=pod_count,
            taints=taints,
            issues=issues
        ))

    return result
```

**detector/issues.py**

```
# detector/issues.py
from dataclasses import dataclass
from typing import List, Dict, Any
from enum import Enum

class Severity(Enum):
    CRITICAL = "CRITICAL"
    WARNING = "WARNING"
    INFO = "INFO"

@dataclass
class ClusterIssue:
    severity: Severity
    category: str
    title: str
    description: str
    affected_resource: str
    namespace: str
    raw_data: Dict[str, Any]
    suggested_commands: List[str] = None

def detect_issues(
    pod_issues,
    node_statuses,
    deployments=None
) -> List[ClusterIssue]:
    """Convert raw collected data into structured issues."""
    issues = []

    # ── Pod issues ───────────────────────────────────────────────
    for pod in pod_issues:
        severity = _classify_pod_severity(pod)

        suggested_commands = [
            f"kubectl describe pod {pod.name} -n {pod.namespace}",
            f"kubectl logs {pod.name} -n {pod.namespace} --previous",
            f"kubectl get events -n {pod.namespace} --field-selector involvedObject.name={pod.name}",
        ]

        if pod.reason == "ImagePullBackOff":
            suggested_commands.append(
                f"kubectl get pod {pod.name} -n {pod.namespace} "
                f"-o jsonpath='{{.spec.containers[*].image}}'"
            )
        elif pod.reason == "OOMKilled":
            suggested_commands.append(
                f"kubectl set resources deployment "
                f"<deployment-name> --limits=memory=<higher-value> "
                f"-n {pod.namespace}"
            )

        issues.append(ClusterIssue(
            severity=severity,
            category="Pod",
            title=f"{pod.reason}: {pod.name}",
            description=_build_pod_description(pod),
            affected_resource=f"pod/{pod.name}",
            namespace=pod.namespace,
            raw_data={
                "status": pod.status,
                "reason": pod.reason,
                "message": pod.message,
                "restart_count": pod.restart_count,
                "age_minutes": pod.age_minutes,
                "node": pod.node,
                "logs": pod.logs,
                "events": pod.events
            },
            suggested_commands=suggested_commands
        ))

    # ── Node issues ──────────────────────────────────────────────
    for node in node_statuses:
        for issue in node.issues:
            severity = Severity.CRITICAL if "NotReady" in issue \
                else Severity.WARNING

            issues.append(ClusterIssue(
                severity=severity,
                category="Node",
                title=f"Node issue: {node.name}",
                description=issue,
                affected_resource=f"node/{node.name}",
                namespace="cluster-scoped",
                raw_data={
                    "node": node.name,
                    "ready": node.ready,
                    "conditions": node.conditions,
                    "pod_count": node.pod_count,
                    "taints": node.taints
                },
                suggested_commands=[
                    f"kubectl describe node {node.name}",
                    f"kubectl get pods -A --field-selector spec.nodeName={node.name}",
                    f"kubectl drain {node.name} --ignore-daemonsets "
                    f"--delete-emptydir-data"
                ]
            ))

    # Sort by severity
    severity_order = {
        Severity.CRITICAL: 0,
        Severity.WARNING: 1,
        Severity.INFO: 2
    }
    issues.sort(key=lambda x: severity_order[x.severity])

    return issues

def _classify_pod_severity(pod) -> Severity:
    """Determine severity of a pod issue."""
    if pod.reason in ["CrashLoopBackOff", "OOMKilled"]:
        return Severity.CRITICAL if pod.restart_count > 5 \
            else Severity.WARNING
    if pod.reason in ["ImagePullBackOff", "ErrImagePull"]:
        return Severity.CRITICAL
    if pod.reason == "PodNotScheduled" and pod.age_minutes > 10:
        return Severity.CRITICAL
    if pod.reason == "HighRestartCount":
        return Severity.WARNING
    return Severity.INFO

def _build_pod_description(pod) -> str:
    """Build a human-readable description of a pod issue."""
    parts = [
        f"Pod {pod.name} in namespace {pod.namespace} "
        f"is in state: {pod.status}",
        f"Reason: {pod.reason}",
        f"Restart count: {pod.restart_count}",
        f"Running on node: {pod.node}",
        f"Age: {pod.age_minutes:.1f} minutes",
    ]
    if pod.message:
        parts.append(f"Message: {pod.message}")
    return "\n".join(parts)
```

**analyzer/claude.py**

```
# analyzer/claude.py
import anthropic
import json
from typing import List
from detector.issues import ClusterIssue, Severity

class ClusterAnalyzer:
    def __init__(self, api_key: str = None):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model = "claude-sonnet-4-20250514"

    def analyze_issue(self, issue: ClusterIssue) -> str:
        """Get Claude's analysis of a specific cluster issue."""

        # Build rich context for Claude
        context = self._build_issue_context(issue)

        prompt = f"""You are a senior Kubernetes engineer helping diagnose a cluster issue.

Here is the issue data:
{context}

Provide a concise analysis with:
1. **Root cause** — the most likely explanation (2-3 sentences)
2. **Immediate fix** — exact kubectl commands to resolve this (be specific)
3. **Prevention** — how to prevent this in the future (2-3 sentences)
4. **Risk level** — {issue.severity.value} — what this means for the cluster

Keep your response under 300 words. Be direct and actionable."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=600,
            messages=[
                {"role": "user", "content": prompt}
            ]
        )

        return response.content[0].text

    def analyze_cluster_summary(
        self,
        issues: List[ClusterIssue],
        cluster_info: dict
    ) -> str:
        """Get Claude's high-level cluster health summary."""

        if not issues:
            return self._healthy_cluster_response(cluster_info)

        # Prepare issue summary for Claude
        issue_summary = self._summarize_issues(issues)

        prompt = f"""You are a senior Kubernetes platform engineer reviewing cluster health.

Cluster information:
{json.dumps(cluster_info, indent=2)}

Issues found ({len(issues)} total):
{issue_summary}

Provide:
1. **Overall health assessment** — one sentence verdict
2. **Top 3 priorities** — what to fix first and why
3. **Pattern analysis** — are these issues related? Common root cause?
4. **Recommended actions** — ordered list of next steps

Keep response under 400 words. Be direct. Use bullet points."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=800,
            messages=[
                {"role": "user", "content": prompt}
            ]
        )

        return response.content[0].text

    def explain_error(
        self,
        error_message: str,
        context: str = ""
    ) -> str:
        """Explain a specific Kubernetes error in plain English."""

        prompt = f"""Explain this Kubernetes error to an engineer:

Error: {error_message}
{f"Context: {context}" if context else ""}

Provide:
1. **What this means** — plain English explanation
2. **Most common causes** — top 3 reasons this happens
3. **How to fix it** — specific commands
4. **How to verify the fix** — what to check after

Be concise. Use kubectl commands where relevant."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=500,
            messages=[
                {"role": "user", "content": prompt}
            ]
        )

        return response.content[0].text

    def generate_runbook(self, issue: ClusterIssue) -> str:
        """Generate a step-by-step runbook for an issue type."""

        prompt = f"""Create a concise runbook for this Kubernetes issue type:

Issue category: {issue.category}
Issue type: {issue.reason if hasattr(issue, 'reason') else issue.title}
Severity: {issue.severity.value}

Format as a numbered runbook:
1. Immediate triage steps
2. Root cause identification
3. Resolution steps (with exact commands)
4. Verification
5. Post-incident: prevent recurrence

Include actual kubectl commands. Keep under 500 words."""

        response = self.client.messages.create(
            model=self.model,
            max_tokens=800,
            messages=[
                {"role": "user", "content": prompt}
            ]
        )

        return response.content[0].text

    def _build_issue_context(self, issue: ClusterIssue) -> str:
        """Build detailed context string for Claude."""
        lines = [
            f"Category: {issue.category}",
            f"Severity: {issue.severity.value}",
            f"Title: {issue.title}",
            f"Namespace: {issue.namespace}",
            f"Affected resource: {issue.affected_resource}",
            f"Description: {issue.description}",
        ]

        if issue.raw_data:
            data = issue.raw_data

            if data.get("logs"):
                # Truncate logs to avoid token waste
                logs = data["logs"]
                if len(logs) > 2000:
                    logs = "...(truncated)...\n" + logs[-2000:]
                lines.append(f"\nContainer logs:\n```\n{logs}\n```")

            if data.get("events"):
                events = "\n".join(data["events"][-10:])
                lines.append(f"\nKubernetes events:\n{events}")

            if data.get("conditions"):
                lines.append(
                    f"\nNode conditions: {json.dumps(data['conditions'])}"
                )

        if issue.suggested_commands:
            lines.append(
                f"\nSuggested diagnostic commands:\n" +
                "\n".join(f"  {cmd}" for cmd in issue.suggested_commands)
            )

        return "\n".join(lines)

    def _summarize_issues(self, issues: List[ClusterIssue]) -> str:
        """Create a compact summary of all issues for Claude."""
        lines = []
        for issue in issues:
            lines.append(
                f"- [{issue.severity.value}] {issue.title} "
                f"(ns: {issue.namespace}): {issue.description[:100]}..."
            )
        return "\n".join(lines)

    def _healthy_cluster_response(self, cluster_info: dict) -> str:
        return (
            "✅ **Cluster Health: Excellent**\n\n"
            f"No issues detected across "
            f"{cluster_info.get('namespace_count', 'all')} namespaces.\n\n"
            "**Recommendations:**\n"
            "- Continue monitoring with Prometheus/Grafana\n"
            "- Review resource utilization weekly\n"
            "- Ensure etcd backups are running\n"
            "- Verify cert expiry dates quarterly"
        )
```

**output/terminal.py**

```
# output/terminal.py
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.markdown import Markdown
from rich.progress import Progress, SpinnerColumn, TextColumn
from rich import box
from rich.columns import Columns
from rich.text import Text
from detector.issues import ClusterIssue, Severity
from typing import List

console = Console()

SEVERITY_COLORS = {
    Severity.CRITICAL: "bold red",
    Severity.WARNING: "bold yellow",
    Severity.INFO: "bold blue"
}

SEVERITY_ICONS = {
    Severity.CRITICAL: "🔴",
    Severity.WARNING: "🟡",
    Severity.INFO: "🔵"
}

def print_header(cluster_name: str, namespace_filter: str = None):
    """Print the tool header."""
    subtitle = f"Analyzing: {cluster_name}"
    if namespace_filter:
        subtitle += f" | Namespace: {namespace_filter}"

    console.print(Panel(
        f"[bold cyan]🔍 K8s Intelligent Cluster Analyzer[/bold cyan]\n"
        f"[dim]{subtitle}[/dim]\n"
        f"[dim]Powered by Claude AI[/dim]",
        border_style="cyan",
        padding=(1, 4)
    ))

def print_cluster_summary(
    node_count: int,
    namespace_count: int,
    pod_count: int,
    issue_count: int,
    critical_count: int,
    warning_count: int
):
    """Print cluster overview stats."""
    stats = [
        Panel(f"[bold]{node_count}[/bold]\n[dim]Nodes[/dim]",
              border_style="blue", padding=(0, 2)),
        Panel(f"[bold]{namespace_count}[/bold]\n[dim]Namespaces[/dim]",
              border_style="blue", padding=(0, 2)),
        Panel(f"[bold]{pod_count}[/bold]\n[dim]Pods[/dim]",
              border_style="blue", padding=(0, 2)),
        Panel(f"[bold red]{critical_count}[/bold red]\n[dim]Critical[/dim]",
              border_style="red", padding=(0, 2)),
        Panel(f"[bold yellow]{warning_count}[/bold yellow]\n[dim]Warnings[/dim]",
              border_style="yellow", padding=(0, 2)),
    ]
    console.print(Columns(stats))
    console.print()

def print_issues_table(issues: List[ClusterIssue]):
    """Print all issues in a summary table."""
    if not issues:
        console.print("[bold green]✅ No issues detected![/bold green]")
        return

    table = Table(
        title=f"Issues Found: {len(issues)}",
        box=box.ROUNDED,
        border_style="dim",
        show_lines=True
    )

    table.add_column("Severity", style="bold", width=10)
    table.add_column("Category", width=10)
    table.add_column("Resource", width=35)
    table.add_column("Namespace", width=20)
    table.add_column("Issue", width=40)

    for issue in issues:
        icon = SEVERITY_ICONS[issue.severity]
        color = SEVERITY_COLORS[issue.severity]
        sev_text = Text(
            f"{icon} {issue.severity.value}",
            style=color
        )
        table.add_row(
            sev_text,
            issue.category,
            issue.affected_resource,
            issue.namespace,
            issue.title
        )

    console.print(table)
    console.print()

def print_ai_analysis(
    issue: ClusterIssue,
    analysis: str,
    index: int,
    total: int
):
    """Print Claude's analysis for a specific issue."""
    icon = SEVERITY_ICONS[issue.severity]
    color = SEVERITY_COLORS[issue.severity]

    console.print(Panel(
        Markdown(analysis),
        title=f"[{color}]{icon} [{index}/{total}] {issue.title}[/{color}]",
        border_style=color.split()[-1],
        padding=(1, 2)
    ))
    console.print()

def print_cluster_ai_summary(summary: str):
    """Print Claude's overall cluster assessment."""
    console.print(Panel(
        Markdown(summary),
        title="[bold cyan]🤖 AI Cluster Assessment[/bold cyan]",
        border_style="cyan",
        padding=(1, 2)
    ))
    console.print()

def print_suggested_commands(issue: ClusterIssue):
    """Print kubectl commands to investigate an issue."""
    if not issue.suggested_commands:
        return

    console.print("[bold dim]Diagnostic commands:[/bold dim]")
    for cmd in issue.suggested_commands:
        console.print(f"  [green]$[/green] [dim]{cmd}[/dim]")
    console.print()

def spinner(message: str):
    """Return a progress spinner context manager."""
    return Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        transient=True
    )
```

**main.py**

```
#!/usr/bin/env python3
# main.py — K8s Intelligent Cluster Analyzer

import click
import os
import sys
from kubernetes import client, config
from rich.console import Console
from rich.progress import Progress, SpinnerColumn, TextColumn

from collector.pods import collect_pod_issues
from collector.nodes import collect_node_status
from detector.issues import detect_issues
from analyzer.claude import ClusterAnalyzer
from output.terminal import (
    print_header,
    print_cluster_summary,
    print_issues_table,
    print_ai_analysis,
    print_cluster_ai_summary,
    print_suggested_commands
)

console = Console()

@click.command()
@click.option(
    "--namespace", "-n",
    default=None,
    help="Limit analysis to a specific namespace"
)
@click.option(
    "--severity",
    type=click.Choice(["CRITICAL", "WARNING", "INFO", "ALL"]),
    default="ALL",
    help="Filter issues by minimum severity"
)
@click.option(
    "--analyze/--no-analyze",
    default=True,
    help="Use Claude AI to analyze issues"
)
@click.option(
    "--max-issues",
    default=5,
    help="Maximum number of issues to analyze with AI (cost control)"
)
@click.option(
    "--explain",
    default=None,
    help="Explain a specific error message"
)
@click.option(
    "--runbook",
    default=None,
    help="Generate runbook for an issue type (e.g., CrashLoopBackOff)"
)
@click.option(
    "--kubeconfig",
    default=None,
    help="Path to kubeconfig file"
)
def analyze(
    namespace,
    severity,
    analyze,
    max_issues,
    explain,
    runbook,
    kubeconfig
):
    """
    🔍 K8s Intelligent Cluster Analyzer

    Scans your Kubernetes cluster for issues and uses
    Claude AI to diagnose and suggest fixes.
    """

    # ── Setup ────────────────────────────────────────────────────

    # Load kubeconfig
    try:
        if kubeconfig:
            config.load_kube_config(config_file=kubeconfig)
        else:
            try:
                config.load_incluster_config()   # running inside cluster
            except:
                config.load_kube_config()        # running locally
    except Exception as e:
        console.print(f"[red]Error loading kubeconfig: {e}[/red]")
        sys.exit(1)

    # Initialize K8s clients
    v1 = client.CoreV1Api()
    apps_v1 = client.AppsV1Api()

    # Initialize Claude analyzer
    api_key = os.environ.get("ANTHROPIC_API_KEY")
    if not api_key and analyze:
        console.print(
            "[yellow]Warning: ANTHROPIC_API_KEY not set. "
            "Running without AI analysis.[/yellow]"
        )
        analyze = False

    analyzer = ClusterAnalyzer(api_key=api_key) if analyze else None

    # ── Special modes ─────────────────────────────────────────────

    # Explain a specific error
    if explain:
        console.print(f"\n[bold]Explaining:[/bold] {explain}\n")
        if analyzer:
            with console.status("[cyan]Asking Claude...[/cyan]"):
                explanation = analyzer.explain_error(explain)
            from rich.markdown import Markdown
            from rich.panel import Panel
            console.print(Panel(
                Markdown(explanation),
                title="[cyan]🤖 Error Explanation[/cyan]",
                border_style="cyan"
            ))
        else:
            console.print("[red]AI analysis not available[/red]")
        return

    # Generate runbook
    if runbook:
        console.print(f"\n[bold]Generating runbook for:[/bold] {runbook}\n")
        if analyzer:
            from detector.issues import ClusterIssue, Severity
            fake_issue = ClusterIssue(
                severity=Severity.WARNING,
                category="Pod",
                title=runbook,
                description="",
                affected_resource="",
                namespace="",
                raw_data={}
            )
            fake_issue.reason = runbook
            with console.status("[cyan]Generating runbook...[/cyan]"):
                rb = analyzer.generate_runbook(fake_issue)
            from rich.markdown import Markdown
            from rich.panel import Panel
            console.print(Panel(
                Markdown(rb),
                title=f"[cyan]📋 Runbook: {runbook}[/cyan]",
                border_style="cyan"
            ))
        return

    # ── Main analysis ─────────────────────────────────────────────

    # Get cluster context name
    try:
        _, active_context = config.list_kube_config_contexts()
        cluster_name = active_context["name"]
    except:
        cluster_name = "unknown-cluster"

    print_header(cluster_name, namespace)

    # Collect data
    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        transient=True
    ) as progress:
        task = progress.add_task("Collecting pod data...", total=None)
        pod_issues = collect_pod_issues(v1)

        progress.update(task, description="Collecting node data...")
        node_statuses = collect_node_status(v1)

        progress.update(task, description="Collecting cluster info...")
        try:
            namespaces = v1.list_namespace(watch=False)
            namespace_count = len(namespaces.items)
            pods_all = v1.list_pod_for_all_namespaces(watch=False)
            pod_count = len(pods_all.items)
        except:
            namespace_count = 0
            pod_count = 0

    # Detect issues
    all_issues = detect_issues(pod_issues, node_statuses)

    # Filter by severity
    severity_map = {"CRITICAL": 0, "WARNING": 1, "INFO": 2, "ALL": 99}
    if severity != "ALL":
        from detector.issues import Severity
        target_severities = {
            "CRITICAL": [Severity.CRITICAL],
            "WARNING": [Severity.CRITICAL, Severity.WARNING],
            "INFO": [Severity.CRITICAL, Severity.WARNING, Severity.INFO]
        }[severity]
        filtered_issues = [
            i for i in all_issues if i.severity in target_severities
        ]
    else:
        filtered_issues = all_issues

    # Count by severity
    from detector.issues import Severity
    critical_count = sum(
        1 for i in filtered_issues if i.severity == Severity.CRITICAL
    )
    warning_count = sum(
        1 for i in filtered_issues if i.severity == Severity.WARNING
    )

    # Print summary stats
    print_cluster_summary(
        node_count=len(node_statuses),
        namespace_count=namespace_count,
        pod_count=pod_count,
        issue_count=len(filtered_issues),
        critical_count=critical_count,
        warning_count=warning_count
    )

    # Print issues table
    print_issues_table(filtered_issues)

    if not filtered_issues:
        if analyze and analyzer:
            cluster_info = {
                "node_count": len(node_statuses),
                "namespace_count": namespace_count,
                "pod_count": pod_count
            }
            with console.status("[cyan]Getting AI assessment...[/cyan]"):
                summary = analyzer.analyze_cluster_summary([], cluster_info)
            print_cluster_ai_summary(summary)
        return

    # AI Analysis
    if analyze and analyzer:
        # Overall cluster summary first
        cluster_info = {
            "cluster": cluster_name,
            "node_count": len(node_statuses),
            "namespace_count": namespace_count,
            "pod_count": pod_count,
            "critical_issues": critical_count,
            "warning_issues": warning_count
        }

        with console.status(
            "[cyan]🤖 Getting AI cluster assessment...[/cyan]"
        ):
            summary = analyzer.analyze_cluster_summary(
                filtered_issues, cluster_info
            )
        print_cluster_ai_summary(summary)

        # Analyze top issues individually
        issues_to_analyze = filtered_issues[:max_issues]
        console.print(
            f"[dim]Analyzing top {len(issues_to_analyze)} issues "
            f"with Claude...[/dim]\n"
        )

        for i, issue in enumerate(issues_to_analyze, 1):
            with console.status(
                f"[cyan]Analyzing issue {i}/{len(issues_to_analyze)}: "
                f"{issue.title}...[/cyan]"
            ):
                analysis = analyzer.analyze_issue(issue)

            print_ai_analysis(issue, analysis, i, len(issues_to_analyze))
            print_suggested_commands(issue)

    else:
        # No AI — just show commands
        for issue in filtered_issues[:10]:
            print_suggested_commands(issue)

    # Final summary
    console.print(
        f"[dim]Analysis complete. "
        f"Found {len(filtered_issues)} issues "
        f"({critical_count} critical, {warning_count} warnings).[/dim]"
    )

if __name__ == "__main__":
    analyze()
```

## 🐳 Part 4: Containerize and Deploy

**Dockerfile**

```
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source
COPY . .

# Non-root user
RUN useradd -m -u 1000 analyzer
USER 1000

ENTRYPOINT ["python", "main.py"]
CMD ["--help"]
```

**Deploy as a K8s Job (scan on demand)**

```
# k8s-analyzer-job.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-analyzer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-analyzer
rules:
- apiGroups: [""]
  resources:
  - pods
  - pods/log
  - nodes
  - namespaces
  - events
  - services
  - configmaps
  - persistentvolumeclaims
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-analyzer
roleRef:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  name: cluster-analyzer
subjects:
- kind: ServiceAccount
  name: cluster-analyzer
  namespace: default
---
apiVersion: batch/v1
kind: Job
metadata:
  name: cluster-analysis
  namespace: default
spec:
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      serviceAccountName: cluster-analyzer
      restartPolicy: Never
      containers:
      - name: analyzer
        image: ghcr.io/youruser/k8s-analyzer:latest
        args: ["--severity", "WARNING", "--max-issues", "5"]
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: analyzer-secrets
              key: anthropic-api-key
        resources:
          requests: {cpu: "200m", memory: "256Mi"}
          limits:   {cpu: "500m", memory: "512Mi"}
```

**Deploy as a CronJob (scheduled scans)**

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-analyzer
  namespace: default
spec:
  schedule: "0 */6 * * *"      # every 6 hours
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cluster-analyzer
          restartPolicy: OnFailure
          containers:
          - name: analyzer
            image: ghcr.io/youruser/k8s-analyzer:latest
            args:
            - --severity
            - WARNING
            - --max-issues
            - "3"
            - --no-analyze    # just collect, don't analyze with AI every 6h
            env:
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: analyzer-secrets
                  key: anthropic-api-key
```

## 🖥️ Part 5: Hands-On — Run the Analyzer

**Exercise 1: Setup and run locally**

```
# Create project structure
mkdir k8s-analyzer && cd k8s-analyzer
mkdir -p collector detector analyzer output

# Create all the files above
# (copy each section into the appropriate file)

# Install dependencies
pip install anthropic kubernetes rich click pydantic python-dotenv

# Set your API key
export ANTHROPIC_API_KEY=your-key-here

# Run against your kind cluster
python main.py

# Filter by severity
python main.py --severity CRITICAL

# Analyze specific namespace
python main.py --namespace production

# Explain a specific error
python main.py --explain "Back-off restarting failed container"
python main.py --explain "0/3 nodes are available: Insufficient memory"
python main.py --explain "ImagePullBackOff: failed to pull image nginx:doesnotexist"

# Generate runbook
python main.py --runbook CrashLoopBackOff
python main.py --runbook ImagePullBackOff
python main.py --runbook OOMKilled
```

**Exercise 2: Break things and watch the analyzer catch them**

```
# Create broken scenarios for the analyzer to find

# Scenario 1: CrashLoopBackOff
kubectl run crasher \
  --image=busybox \
  -- sh -c "echo 'crashing' && exit 1"

# Scenario 2: ImagePullBackOff
kubectl run bad-image \
  --image=nginx:this-tag-does-not-exist-v99999

# Scenario 3: Pending pod (impossible resource request)
kubectl run pending-pod \
  --image=nginx:1.25 \
  --overrides='{"spec":{"containers":[{"name":"pending-pod","image":"nginx:1.25","resources":{"requests":{"memory":"999Gi","cpu":"999"}}}]}}'

# Scenario 4: OOMKilled simulation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: app
    image: python:3.11-slim
    command: ["python", "-c"]
    args: ["data = ' ' * (500 * 1024 * 1024); import time; time.sleep(60)"]
    resources:
      limits:
        memory: "64Mi"    # too small — will OOMKill
EOF

# Wait for issues to develop
sleep 30

# Run analyzer — should find all 4 issues
python main.py

# Expected output:
# 🔴 CRITICAL | Pod | crasher         | CrashLoopBackOff
# 🔴 CRITICAL | Pod | bad-image       | ImagePullBackOff
# 🔴 CRITICAL | Pod | pending-pod     | PodNotScheduled
# 🟡 WARNING  | Pod | oom-test        | OOMKilled

# Cleanup
kubectl delete pod crasher bad-image pending-pod oom-test --force 2>/dev/null
```

**Exercise 3: Build and run as a Docker container**

```
# Build the image
docker build -t k8s-analyzer:dev .

# Run against your cluster
docker run --rm \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v ~/.kube/config:/home/analyzer/.kube/config:ro \
  k8s-analyzer:dev \
  --severity WARNING

# Run the explainer
docker run --rm \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v ~/.kube/config:/home/analyzer/.kube/config:ro \
  k8s-analyzer:dev \
  --explain "Error from server (Forbidden): pods is forbidden"
```

## 🎯 Part 6: Interview Questions — Day 28

**Q1: How do you design a Kubernetes tool that uses AI safely — without leaking sensitive cluster data?**

Three layers of control. Data minimization: collect only what's needed for diagnosis — pod status, events, logs — never secrets, configmap values, or environment variables. Log scrubbing: before sending to the AI API, strip anything matching secret patterns (passwords, tokens, private IPs). The --explain mode only sends user-provided error strings, not live cluster data. Audit trail: log every API call made to the AI service with timestamp, what data was sent (schema, not values), and what response was received — gives compliance team visibility. Network controls: run the analyzer as a K8s Job with NetworkPolicy that only allows egress to api.anthropic.com — prevents data exfiltration to other endpoints. For highly regulated environments: run a local model (Llama via Ollama) instead of a cloud API — zero data leaves the cluster.

**Q2: How would you extend this tool to handle multi-cluster environments?**

The ClusterAnalyzer already uses kubeconfig for connection — extend it to iterate over multiple contexts. Store per-cluster configs in a ConfigMap or Secret. Run as an ApplicationSet in ArgoCD — deploy one analyzer Job per cluster, each with its own kubeconfig Secret mounted as a volume. Aggregate results: each cluster's analyzer writes its findings to a shared database (Postgres or InfluxDB) keyed by cluster name. A central Grafana dashboard queries all clusters' findings. For correlation: if the same pod name is failing across 3 clusters, Claude can detect the pattern — "this appears to be a global config push issue, not cluster-specific." Multi-cluster summary: an additional mode that pulls findings from the database and asks Claude to identify cross-cluster patterns.

**Q3: What are the token cost implications of using AI for cluster analysis and how do you control it?**

Cost drivers: log content is the biggest token consumer — 50 lines of logs per pod × 20 broken pods = significant tokens per analysis run. Controls: --max-issues N limits how many issues get full AI analysis. Log truncation — the _build_issue_context method caps logs at 2000 characters. Caching — hash the issue signature (pod name + reason + log fingerprint) and cache Claude's response for 1 hour — repeated analysis of the same unchanged issue reuses the cached response. Tiered analysis — use a fast cheap model (Haiku) for initial triage, only escalate to Sonnet/Opus for CRITICAL issues. Scheduled vs on-demand — the CronJob runs --no-analyze every 6 hours (pure collection, no AI cost), only trigger AI analysis when a human requests it or a PagerDuty alert fires.


















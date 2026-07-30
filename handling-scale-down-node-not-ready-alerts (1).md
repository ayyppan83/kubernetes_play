# Handling "Node Not Ready" False Alerts During Cluster Autoscaling

## 1. The Problem

Cluster Autoscaler (or Karpenter/GKE Autopilot/Nodegroup autoscaling) spins up new
nodes when traffic increases during business hours and terminates them when load
drops. When a node is cordoned, drained, and terminated as part of a **planned
scale-down**, it transitions through `NotReady` briefly (or the node object
disappears). Your monitoring stack (Prometheus + Alertmanager, or similar) sees
this as `KubeNodeNotReady` and fires an alert → incident, even though nothing is
actually broken.

**Goal:** Stop treating expected, autoscaler-driven node removal as an incident,
while still catching genuine node failures (kubelet crash, VM failure, network
partition, etc.).

There is no single switch for this — it's a combination of **alert tuning**,
**inhibition/suppression rules**, and **lifecycle-aware automation**. Below is a
layered approach, roughly in order of effort vs. impact.

---

## 2. Layer 1 — Tune the Alert Rule Itself (Quick Win)

Most default `KubeNodeNotReady` rules (e.g. from kube-prometheus-stack) fire too
fast for autoscaled environments.

### 2.1 Increase the `for` duration
Scale-down draining usually completes in 1–5 minutes. If your alert fires after
15s–1m of NotReady, you'll catch every scale-down.

```yaml
- alert: KubeNodeNotReady
  expr: kube_node_status_condition{condition="Ready", status="true"} == 0
  for: 10m   # was likely 15s or 5m — raise it above your typical drain time
  labels:
    severity: warning
  annotations:
    summary: "Node {{ $labels.node }} not ready for over 10m"
```

This alone eliminates most transient blips but won't fully solve it if scale-down
takes longer than genuine outages you want to catch quickly — so combine with the
layers below rather than relying on this alone.

### 2.2 Exclude nodes already marked for deletion/unschedulable
Autoscalers cordon (`spec.unschedulable = true`) before removing a node. Filter
those out so a node that's *intentionally* being retired doesn't also trigger
NotReady noise:

```yaml
expr: |
  kube_node_status_condition{condition="Ready", status="true"} == 0
  unless on(node)
  kube_node_spec_unschedulable == 1
```

### 2.3 Exclude nodes tagged as spot/preemptible/autoscaled pools
If autoscaled nodes carry an identifying label (common in GKE/EKS/AKS node
pools), scope the alert away from that pool or route it to a lower-severity
channel:

```yaml
expr: |
  kube_node_status_condition{condition="Ready", status="true"} == 0
  unless on(node)
  kube_node_labels{label_eks_amazonaws_com_capacityType="SPOT"}
```

---

## 3. Layer 2 — Alertmanager Inhibition Rules (Recommended)

Instead of guessing durations, actively **suppress `NodeNotReady` when a known
scale-down event is in progress** for that same node. This is the most robust
config-only fix.

### 3.1 Emit a "scale-down in progress" signal
- Cluster Autoscaler exposes metrics like `cluster_autoscaler_scaled_down_nodes_total`
  and emits Kubernetes **Events** (`ScaleDown`, `NodeDeleted`) — scrape/alert on these.
- Karpenter emits `karpenter_nodes_terminated` metrics and `Disrupted`/`Deleted` events.

Create a low-severity alert/marker from that signal:

```yaml
- alert: ClusterAutoscalerScaleDownInProgress
  expr: increase(cluster_autoscaler_scaled_down_nodes_total[5m]) > 0
  labels:
    severity: info
```

### 3.2 Inhibit the real alert using that marker
```yaml
inhibit_rules:
  - source_matchers:
      - severity = "info"
      - alertname = "ClusterAutoscalerScaleDownInProgress"
    target_matchers:
      - alertname = "KubeNodeNotReady"
    equal: ["node"]
```

Now `KubeNodeNotReady` never pages if it's correlated with a known scale-down on
the same node. It will still fire normally for unplanned failures.

---

## 4. Layer 3 — Kubernetes-Side Lifecycle Hygiene

Reduce the *duration and visibility* of the NotReady window itself:

1. **Graceful drain, not force-delete** — ensure Cluster Autoscaler/Karpenter
   drain settings (`--max-graceful-termination-sec`, Karpenter's
   `terminationGracePeriod`) allow pods to shift cleanly, so nodes go
   `Cordoned → Drained → Terminated` predictably instead of vanishing abruptly.
2. **Set PodDisruptionBudgets** on critical workloads so scale-down doesn't
   rush and cause secondary alerts (pod-level, not just node-level).
3. **Deregister the node from monitoring before termination** — if using
   node-exporter/kubelet scraping via service discovery, ensure the scrape
   target is removed via K8s SD (`kubernetes_sd_configs`) relabeling as soon as
   the node is deleted, not left as a stale/failing target.
4. **Use `karpenter.sh/disruption` or `ToBeDeletedByClusterAutoscaler` taints**
   as another filter key in your PromQL `unless` clauses (Layer 1.2 pattern).

---

## 5. Layer 4 — Process / Automation Layer

Even with good rules, add a safety net at the incident-management layer:

1. **Auto-resolve / auto-silence via webhook**: Have Cluster Autoscaler/Karpenter
   scale-down events trigger a short-lived Alertmanager silence for that node
   via `amtool silence add -m node=<name> -d 15m` (scripted from a K8s Event
   watcher or a small controller/Lambda/CronJob).
2. **Route to a "scaling-events" low-priority channel** instead of the main
   paging channel for anything correlated with autoscaler activity, using
   Alertmanager routing (`routes:` + `matchers:`), so it's visible but doesn't
   page on-call.
3. **Runbook tagging**: annotate the alert with a runbook link explaining "if
   this correlates with a scale-down event, no action needed" so responders can
   self-triage quickly if it does still fire.
4. **Periodic review**: track false-positive rate weekly; adjust `for:`
   duration and inhibition windows based on actual observed drain times.

---

## 6. Suggested Implementation Order

| Step | Action | Effort | Impact |
|------|--------|--------|--------|
| 1 | Raise `for:` duration on `KubeNodeNotReady` | Low | Medium |
| 2 | Add `unschedulable`/spot-label exclusion to PromQL | Low | Medium |
| 3 | Add Cluster Autoscaler/Karpenter scale-down metric + alert | Medium | High |
| 4 | Add Alertmanager inhibition rule linking the two | Medium | High |
| 5 | Tune graceful termination / drain settings | Medium | Medium |
| 6 | Add routing to low-priority channel as a safety net | Low | Medium |
| 7 | Automate silences via webhook (optional, for maturity) | High | High |

---

## 7. Summary

- **Yes, this is controllable purely through alert/Alertmanager configuration**
  in most cases — you don't need to change your autoscaling behavior.
- The most reliable fix is **correlating the `NodeNotReady` alert with a
  concurrent, known scale-down event** (inhibition rule), rather than just
  widening the `for:` window, because widening alone is a blunt instrument.
- Treat it as defense in depth: tune the rule, add exclusions, correlate with
  autoscaler signals, and route as a fallback — so real node failures still
  page you promptly.

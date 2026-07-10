# Guide to FPF and DPF Principles for Grafana Dashboards in Virtualization Environments

## Purpose

Grafana is the presentation and query-scoping layer for virtualization monitoring. It turns metric carriers from the observability backend into readable views that help operators compare capacity, detect saturation, and understand behavior over time.

In this setting, the dashboard is not the system of record. It is a controlled publication surface that answers operational questions such as which virtual machine is overloaded, what changed in business hours, and how that compares to the full day.

## Ontology

A useful ontology here has four levels: carriers, selectors, readings, and views.

- Carriers are metrics and label-bearing series such as CPU, memory, swap, and host identity signals.
- Selectors are dashboard variables that narrow the scope.
- Readings are the derived values and charts.
- Views are the panels and dashboard layout that present those readings.

The same host can appear under several representations without being a different object. For example, `node_uname_info` can act as a host catalog, while `node_cpu_seconds_total` carries the CPU signal, and the dashboard binds them through selectors and queries.

## Variables

Dashboard variables are selector variables, not the data itself. They let the user choose a project, a scrape job, a hostname, or a technical instance, and Grafana substitutes those values into panel queries and titles.

In a virtualization dashboard, a common chain is project → job → hostname → node. The exact labels may differ by environment, but the pattern is the same: a higher-level context narrows to a concrete host target that the panels can query.

## Panels

Panels are the visual expressions of the selected readings. For vCPU, the most useful first panels are usually current utilization trend, p50/p95 utilization, and percent of time above threshold.

A good dashboard does not stop at what the CPU is right now. It should also show how CPU behaves across the chosen slice so that operators can see stable load, spikes, business-hour pressure, and potential rightsizing opportunities.

## Time windows

Time windows are part of the meaning of the reading, not just a UI convenience. For virtual machines, it is often more informative to compare business hours against full-day or multi-day windows than to stare at one unqualified number.

Grafana supports panel-specific time range, time shift, and time comparison, which makes it possible to view the same metric under different windows without rewriting the metric logic itself. In practice, this means you can keep one CPU query and display it in two modes: the working-day view and the full-time baseline.

## Risks

The main risk is confusing the selector layer with the observed system. A label such as `instance` is not the virtual machine itself; it is a technical identity used to retrieve the VM’s metrics.

Another risk is overfitting the dashboard to one naming scheme. If the labels change upstream, a dashboard that silently assumes a fixed variable chain can break or, worse, show the wrong slice. A final risk is treating the dashboard as proof; Grafana is a presentation layer, so it needs clear provenance and time context when its outputs are used as evidence for decisions.

## Example mapping

These examples illustrate the concept:

- `node_uname_info` as a host catalog for selector recovery.
- `project` as a context selector.
- `job` as a scrape or observation channel selector.
- `hostname` as a human-readable host selector.
- `node` as the technical instance selector used in panel queries.

These are examples, not the definition itself. The underlying idea is that dashboards should separate carriers, selectors, readings, and views.

## Short glossary

- **FPF**: the framing used to separate method, evidence, work, and publication concerns.
- **DPF**: the dashboard-oriented framing used here to describe the ontology of Grafana dashboards.
- **Carrier**: a metric or label-bearing series.
- **Selector**: a variable or label filter used to scope a dashboard query.
- **Reading**: a derived metric view or chart.
- **View**: the dashboard or panel that displays readings.

## Explore as an experiment surface

Grafana Explore is an experimental query surface, separate from a published dashboard. Use it to inspect carriers, vary selectors, compare time windows, and test a candidate reading before it becomes a panel, recording rule, or alert expression.

A query in Explore is an executable hypothesis over current or historical observations. To validate a future alert, run the exact `expr` over an interval containing both normal and suspected failure states. The returned series shows when the expression is true. Explore does not evaluate the alert lifecycle itself: `for` is evaluated only by the rule engine (Prometheus, VMAlert, or Grafana Alerting).

When a panel contains multiple queries, isolate a reading before interpreting it. Temporarily hide a query with its visibility control instead of deleting it. For binary and state metrics, prefer lines or points over stacked rendering, because stacking adds simultaneous series and can create a misleading apparent value.

## Query language and portability

For VictoriaMetrics-backed dashboards, readings are expressed in MetricsQL. MetricsQL is compatible with PromQL and extends it, so a PromQL-compatible expression can normally be explored in Grafana and reused in VMAlert.

The query language is part of a reading definition. A reading is not only a metric name: it also includes label scope, aggregation, range window, vector matching, and missing-data semantics. For example, `avg_over_time(metric[5m])` summarizes observed samples, whereas `absent_over_time(metric[5m])` asks whether the selected series has no samples in the range.

## Missing data and evidence boundaries

A missing series is evidence about the observation path, not automatically evidence that the observed system has failed. A gap can result from a target failure, exporter failure, scrape failure, a network partition between collector and target, service discovery or relabeling changes, or a changed label set.

Dashboards and alerts should therefore distinguish three evidence layers:

- **Object state**: what the observed service reports about itself.
- **Observation-path state**: whether the monitoring system can collect that report.
- **Service outcome**: whether a consumer can use the service through its intended endpoint.

A claim about service failure needs evidence from the service-outcome layer or from independent observation paths. A missing internal metric can justify an observability alert, but it cannot by itself conclusively prove a service outage.

## Alert-expression validation

An alert rule has two separate parts: `expr`, which decides whether the alert condition is true at an evaluation instant, and `for`, which requires that condition to remain true continuously before the alert fires.

To validate an alert in Explore, paste only its exact `expr`, select a time interval around a known event, and use a resolution no coarser than the rule evaluation interval. Then verify visually or through tabular results that the expression remains non-empty (or remains `1`, depending on the expression) for at least the configured `for` duration. Validate missing-data rules separately because an empty result and a result with value zero are not equivalent in PromQL or MetricsQL.

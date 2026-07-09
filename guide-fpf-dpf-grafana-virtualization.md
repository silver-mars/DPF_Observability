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

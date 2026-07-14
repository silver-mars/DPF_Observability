# Operational Observability Dashboards

## Purpose

This guide defines a platform-neutral approach to operational dashboards used to classify problems, choose the next investigation, and support bounded operational decisions. It applies to virtual machines, services, databases, queues, network components, and other observable systems.

A dashboard is a publication and exploration surface. It is not the observed system, a system of record, evidence by itself, an approval, or an automated decision-maker.

## Scope and non-goals

The guide covers dashboard reasoning: how to select an entity, express a decision question, construct readings, compare time windows, preserve evidence boundaries, and route a user from a fleet-level view to a specific investigation.

It does not prescribe one observability product, one query language, universal thresholds, a complete service-management process, or alert lifecycle design. Grafana, PromQL, MetricsQL, recording rules, and dashboards-as-code are implementation choices.

## Decision first

Start with the operational decision or problem class, not with the available metric list. Examples include:

- Is a service breaching its user-facing objective?
- Which entities require investigation first?
- Is a virtual machine a candidate for a capacity review?
- Is the observed condition likely a resource constraint, an application fault, or an observation-path failure?

The decision question declares what a reading may support and what it cannot establish. A low CPU reading can support a compute-rightsizing review; it cannot by itself prove that a service is unimportant or safe to decommission.

## Bounded context

Every dashboard operates in a named semantic frame. The context must define:

- The intended reader and their first action
- The entity type and recognition rule
- The decision or investigation use
- The local meaning of terms such as healthy, saturation, candidate, pressure, breach, and available
- The relevant time window, data source, and freshness expectations
- The non-use boundary and escalation route

A useful context name is specific, for example `OperationalObservability.ServiceInvestigation` or `VirtualMachine.ComputeRightsizing`.

## Ontology

### Entity and scale

Choose the Entity of Concern before aggregation:

| Scale | Typical concern | Example question |
|---|---|---|
| Fleet or collection | Prioritisation and outlier detection | Which instances deserve review? |
| Entity | Diagnosis or bounded decision | Can this VM be resized? |
| Resource part | Local mechanism | Which vCPU, filesystem, process, or interface is constrained? |
| Service outcome | Consumer-visible behaviour | Are users receiving the promised result? |

A collection is not automatically an acting system. A fleet map is a view over a selected collection; it does not make the collection the owner of a decision.

### Carriers, selectors, readings, and views

| Object | Meaning | Example |
|---|---|---|
| Carrier | A source metric, event, trace, log, inventory record, or label-bearing series | `node_cpu_seconds_total` |
| Selector | A scope control applied to a query or view | project, job, hostname, instance |
| Reading | A derived, unit-bearing interpretation of one or more carriers | CPU p95 %, error rate, available RAM % |
| Claim | A bounded statement supported by readings | `resize-review candidate` |
| View | A panel, table, report, or dashboard that publishes readings | Fleet table, VM detail panel |
| Decision or work | A human or automated action performed under a separate process | approve and execute resize |

A technical label such as `instance` is a retrieval identity, not the observed system itself. A dashboard view is not the claim, and the claim is not the performed work.

## Outcome and mechanism signals

Use the decision context to decide whether outcome or mechanism signals lead the view.

For service-reliability decisions, lead with user- or consumer-relevant outcomes: availability, latency, error rate, correctness, freshness, or another declared service objective. CPU, memory, I/O, retries, and queue depth are mechanism signals that help explain likely causes.

For resource-capacity decisions, the allocated resource and its consumption can lead the view. For example, CPU and memory readings are primary evidence for compute rightsizing, while service outcomes remain a guardrail against harmful change.

SLO-first therefore means outcome-or-decision first, not that every dashboard must display the same availability graph.

## Reading construction

A reading definition includes:

- Carrier identity and label scope
- Aggregation and grouping rule
- Unit and normalization
- Range or rate window
- Missing-data behaviour
- Time window and comparison baseline
- Interpretation and non-interpretation boundary

Preserve units visibly. A percentage, a byte count, an event rate, an absolute resource equivalent, and a percentile are different readings even when they share a label such as `utilization`.

## Time and comparison

Time is part of a reading's meaning. Use a declared observation period and, where useful, compare:

- Current value for immediate orientation
- Trend for shape, drift, and burst detection
- p50 for typical behaviour
- p95 for high-but-normal behaviour
- Time above a threshold for duration of pressure
- Business-hours and full-time views
- Current period and prior baseline

A single average can hide short saturation periods; a single maximum can overstate a rare event. Percentiles and time-above-threshold should be interpreted only with their range, sampling, and aggregation semantics stated.

## Drill-down design

A useful dashboard supports an explicit investigation route:

1. Start with a fleet or service summary to locate outliers.
2. Select one entity using stable identity and context selectors.
3. Inspect trend, percentile, and time-window readings.
4. Open resource or subsystem detail only when the decision question requires it.
5. Return to source carriers, inventory, runbooks, traces, or owners when a stronger claim is needed.

The panel layout should expose this route rather than merely display all available metrics.

## Evidence boundaries

A metric reading is an evidence carrier for a claim only within a declared scope, method, and relevance window. A screenshot or a dashboard tile can support orientation but cannot on its own prove safety, approval, root cause, outage, or completed work.

For a reliance-bearing claim, retain enough provenance to answer: which carriers, source systems, query method, time window, entity scope, and rival explanations were considered? Refresh the claim when the data source, labels, instrumentation, workload, or operating context changes.

## General failure modes

- **Metric-first construction:** panels are chosen because data exists, not because they support a decision.
- **Average-only reasoning:** typical load masks regular peaks, contention, or burst behaviour.
- **Label-as-identity:** a scrape target is treated as the system without an inventory or recognition rule.
- **Dashboard-as-proof:** a visualisation is treated as assurance, approval, or completed work.
- **Outcome-mechanism collapse:** low-level resource metrics are mistaken for user experience, or vice versa.
- **Scope-free threshold:** one value is applied across workloads without a local context, window, or rationale.
- **Missing-data overread:** absence of a series is treated as proof of an object or service failure.

## Related DPFs

- [VM Compute Rightsizing](dpf-vm-compute-rightsizing.md) applies this guide to CPU and RAM capacity decisions for virtual machines.
- [Alerting and Observability Assurance](dpf-alerting-and-observability-assurance.md) applies the evidence and outcome principles to alert conditions, missing data, and validation.

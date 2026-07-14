# DPF: VM Compute Rightsizing

## Purpose

This DPF supports reviewable CPU and RAM capacity decisions for virtual machines. Its purpose is to identify candidates for a rightsizing review, explain the evidence, and distinguish a recommendation from the actual resize work.

It is platform-neutral. It assumes guest-level infrastructure metrics may be available, but it does not require one cloud, hypervisor, monitoring product, query language, or flavour catalogue.

## Scope and non-goals

The primary decision is whether a VM appears to have sustained CPU and memory headroom sufficient to review a smaller compute allocation. The framework covers evidence construction, percentile interpretation, safety checks, and decision boundaries.

It does not define universal utilisation thresholds, automatically approve a resize, define storage rightsizing, prove application criticality, or replace owner, change-management, licensing, affinity, NUMA, or rollback requirements.

## Context and entities

Use a named context such as `VirtualMachine.ComputeRightsizing`. The entity under review is one virtual machine, recognised through an inventory identity and linked technical observation identities.

A fleet is a collection used to rank or find candidate VMs. A VM is the entity for a concrete rightsizing claim. Its vCPUs, allocated memory, guest processes, filesystems, and interfaces are resource parts or adjacent observed objects.

## Resource claims

Keep these claims distinct:

| Claim | Meaning | Typical unit |
|---|---|---|
| Allocated vCPU capacity | vCPUs made available to the guest | vCPU count |
| CPU utilisation | Average busy share of allocated guest vCPU capacity | percent, 0-100% |
| CPU consumed | Aggregate guest CPU work over the interval | vCPU-equivalent |
| Allocated memory | Memory available to the guest | bytes or GiB |
| Memory use | A declared memory-use reading | bytes, GiB, or percent |
| Memory headroom | Memory still available without reclaim pressure | bytes, GiB, or percent |
| Memory pressure | Signals of reclaim, swap, faults, or OOM | rate, count, or state |

Do not call `vCPU-equivalent` a VM percentage. Do not call a memory-use percentage proof that memory pressure is absent.

## CPU readings

A guest CPU counter normally exposes cumulative seconds in per-vCPU modes, including `idle`. A rate over a declared range converts the counter to an idle-time ratio for each observed vCPU.

```text
busy share per vCPU = 1 - rate(idle CPU time)
```

The following readings answer different questions:

| Reading | Interpretation | Unit |
|---|---|---|
| `avg by(instance)(1 - rate(idle))` | Average busy share across guest vCPUs | ratio, 0-1 |
| `100 * avg by(instance)(1 - rate(idle))` | Average VM vCPU utilisation | percent, 0-100% |
| `sum by(instance)(1 - rate(idle))` | Aggregate CPU work consumed | vCPU-equivalent |
| `100 * sum(...)` | Aggregate work expressed as 100 per fully busy vCPU | percent-vCPU, not VM utilisation |
| `sum(...) / count(idle series)` | Aggregate consumption normalised by observed vCPU capacity | ratio, 0-1 |

For ordinary per-vCPU guest series, `100 * avg(...)` and `100 * sum(...) / count(...)` are equivalent utilisation percentages. The first form is simpler; the second makes the capacity denominator explicit.

Apply `rate` before aggregation so counter resets can be handled per series. The rate interval, labels, CPU modes included or excluded, and aggregation scope must be visible in the implementation.

## CPU percentile interpretation

Use a time series plus a fixed-period summary:

- **Trend:** reveals bursts, drift, regular schedules, and changed workload shape.
- **p50:** typical CPU demand during the declared period.
- **p95:** high-but-normal demand during the declared period; it is not a hard maximum.
- **Time above threshold:** duration of elevated use under a local threshold policy.
- **Consumed vCPU-equivalent:** enables comparison of observed work with proposed vCPU capacity.

A rolling p95 trend is useful for diagnosis, but it is not a substitute for one final p95 calculated over a declared decision window. A p95 or average must not be read without its period and whether it covers business hours, full time, or a workload-specific schedule.

## Memory readings

Memory rightsizing requires both consumption and pressure evidence. A minimal reading set is:

- Allocated memory
- Memory used under a documented definition
- Available memory under the guest operating system's available-memory semantics
- p50 and p95 memory-use readings
- Swap-in and swap-out activity
- Major page faults
- OOM events or kills where exported

Low `used` alone is insufficient because operating systems use memory for cache and may reclaim it safely. Low available memory together with sustained swap, major faults, or OOM is a stronger pressure signal than used percentage alone.

## Virtualisation boundary

Guest-level CPU and memory metrics describe what the guest reports. They do not fully describe hypervisor scheduling, physical-host oversubscription, storage latency, network throughput limits, or application behaviour.

Guest CPU `steal` time is a useful separate signal where available: it can indicate that a runnable guest vCPU waited because physical CPU time was unavailable. It is evidence of possible host contention, not direct proof that the guest needs more vCPU.

## Candidate classification

Use classifications as claims with evidence:

| Status | Meaning |
|---|---|
| `observe` | Data is incomplete, too recent, or workload context is unknown |
| `reclaim candidate` | CPU and RAM show sustained headroom sufficient for review |
| `resize review` | A proposed target capacity can be assessed against p95, pressure checks, and constraints |
| `resize recommended` | A local policy, evidence window, owner/application constraints, and change path support a specific proposal |
| `manual review` | A scheduled workload, contention, missing data, special configuration, or criticality requires human assessment |

Thresholds, required headroom, and minimum evidence periods are local policy and should be added only after they are implemented, reviewed, and linked to an observed change process.

## What SSD and network mean here

SSD capacity and network traffic are not prerequisites for an initial decision to review a CPU/RAM-only resize. They become primary evidence for different claims:

| Decision | Primary evidence |
|---|---|
| Reduce vCPU | CPU demand profile, consumed vCPU-equivalent, CPU pressure/steal, constraints |
| Reduce RAM | Memory demand profile, available memory, swap/fault/OOM signals, constraints |
| Reduce disk allocation | Filesystem or volume capacity, growth, recovery and persistence constraints |
| Diagnose storage bottleneck | I/O throughput, latency, queue/saturation, application evidence |
| Decommission VM | Role, owner, dependency, service outcome, activity, and lifecycle evidence |
| Assess network-bound behaviour | Throughput, packet rate, errors, latency, and service/application evidence |

Do not use low CPU/RAM to conclude that a VM is unimportant, unused, or removable.

## Decision evidence

A rightsizing claim should record:

- VM identity and inventory source
- Allocated and proposed capacities
- CPU and memory carriers, query definitions, units, and labels
- Observation period and workload windows
- p50, p95, trend, and threshold-duration readings where used
- Memory-pressure and CPU-contention checks
- Application, owner, licensing, topology, and scheduling constraints
- Proposed work path, rollback, and reopen conditions

The dashboard publishes this evidence. A human or authorized system performs the resize through a separate change process.

## Risks

- A short quiet period can hide monthly, batch, failover, or seasonal demand.
- Average CPU can mask bursts; p95 can also hide the most extreme tail.
- Guest idle can coexist with host-level contention or non-CPU bottlenecks.
- Low memory used can coexist with memory pressure if definitions are unclear.
- CPU/RAM evidence does not establish disk shrink safety, decommission safety, or service criticality.
- A target flavour may be constrained by licensing, NUMA, CPU pinning, hardware topology, or application configuration.

## Relationship to the general guide

This is a bounded application of [Operational Observability Dashboards](guide-operational-observability-dashboards.md). Query examples, dashboard JSON, and provider-specific inventory integrations belong in a separate implementation appendix.

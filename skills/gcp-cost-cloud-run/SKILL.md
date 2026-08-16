---
name: gcp-cost-cloud-run
description: >
  This skill should be used when Cloud Run costs more than its traffic explains:
  "why is Cloud Run so expensive", "my service has min-instances 0 but CPU stays
  high", "why does the service never scale to zero", "instance-based vs
  request-based billing", "why does staging cost less than prod with the same
  config", "cpu_idle", "cpu-throttling", "CPU always allocated", "why is my
  sidecar billing", "Min Instance CPU SKU but min instances is zero", or when a
  Cloud Run bill is flat and decoupled from request volume. Covers the two
  billing models, why CPU allocation is a property of the instance rather than
  the container, the API that reports it honestly, and how identical
  configuration bills differently in two environments. Use gcp-cost-analysis for
  finding which resource costs money in the first place.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.1.0
---

# Cloud Run billing modes and the sidecar trap

Covers what makes a Cloud Run service bill for time it is not working, and how
to tell which billing model a deployed service is actually on — which is not
always the one its configuration appears to request.

Claims marked **verified** were established by observation on a running
deployment: billing-export SKUs, Cloud Monitoring series, and the live service
API, cross-checked against each other. Claims marked *documented* come from
Google's published pricing model.

## Quick answers

| Question | Answer |
| :--- | :--- |
| `min-instances` is 0 but CPU bills continuously — is that the cause? | **No.** `min-instances` is almost never the cause. Check the billing model first (CR1) |
| What decides instance-based vs request-based billing? | Whether **any** container in the instance has CPU always allocated (CR2) |
| Is CPU allocation per container? | **It is configured per container and billed per instance.** That asymmetry is the whole trap (CR2) |
| One sidecar lacks `cpu_idle` — what does it cost? | The **entire instance**, at full rate, for its **full lifetime** — not the sidecar's CPU share (CR3) |
| How do I read the real setting? | v2 API, `.template.containers[].resources.cpuIdle`. **Absent means always allocated** (CR5) |
| Does `gcloud run services describe` tell me? | **No — it will mislead you.** v1 exposes one service-level annotation that hides per-container divergence (CR6) |
| Same Terraform, two environments, different cost — how? | A `gcloud` write normalises the setting onto every container; a Terraform write does not (CR7) |
| Symptom that confirms it | `instance_count` reports **zero idle** instance-hours, and only instance-based SKUs appear (CR4) |
| `Min Instance CPU` SKU bills but min instances is 0 | That SKU is **idle-retained instance time** under request-based billing, not configured min-instances (CR8) |
| Which sidecars genuinely need always-on CPU? | Only ones that must do work between requests — background pollers, queue consumers (CR9) |
| Is this fixed by caching or scaling changes? | **No.** It is a two-line configuration fix; scaling changes treat the symptom (CR10) |

## The two billing models

*Documented.* Cloud Run bills a service one of two ways, and the difference is
large enough that it dominates every other Cloud Run cost decision.

| Model | Bills for | Idle time | Set by |
| :--- | :--- | :--- | :--- |
| **Request-based** | time spent processing requests, plus a cheap idle rate for retained instances | cheap | CPU throttled outside requests (`cpu_idle = true`) |
| **Instance-based** | the **entire instance lifetime**, at full rate | full rate | CPU always allocated (`cpu_idle` false or absent) |

Both models keep instances warm after a request. The difference is what that
warm time costs. This is why "scale to zero" intuitions mislead: a service with
`min-instances = 0` still retains instances between requests, and under
instance-based billing that retention is charged at the full rate.

## Forces

Rules, not preferences. `⇒` means the platform gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| CR1 | `min-instances` controls how many instances are **held**, not how the running ones are **billed** | ⇒ a service with `min-instances = 0` can still bill continuously ⇒ checking it first sends you down the wrong path |
| CR2 | CPU allocation is configured **per container** but billed **per instance** — if any container has CPU always allocated, the instance does | ⇒ one sidecar sets the billing model for the whole instance ⇒ the main container's setting can be entirely cosmetic |
| CR3 | Instance-based billing charges the instance's **full CPU total** for its **full lifetime**, not just the offending container's share | ⇒ a 0.25-vCPU sidecar on a 1.45-vCPU instance costs the whole 1.45 vCPU, continuously ⇒ cost is decoupled from the sidecar's own size |
| CR4 | An instance with an always-allocated container is **never in the throttled state** that `instance_count`'s `idle` label describes | ⇒ `idle` reads **exactly zero** ⇒ this is the cleanest single confirmation, because a healthy request-based service always accumulates idle time |
| CR5 | Per-container CPU allocation is only exposed on the **v2** API, as `.template.containers[].resources.cpuIdle`, and the field is **omitted when false** | ⇒ absent ≠ unset-and-harmless; absent **means always allocated** ⇒ a presence check reads the wrong answer |
| CR6 | The v1 API models CPU allocation as a **single service-level annotation** (`run.googleapis.com/cpu-throttling`) | ⇒ `gcloud run services describe` reports one value for a service whose containers disagree ⇒ it will report `true` while the service bills instance-based |
| CR7 | `gcloud run services update` round-trips the service through v1, and writing back that single annotation **normalises it onto every container** | ⇒ a gcloud-deployed environment silently acquires the correct setting ⇒ a Terraform-deployed environment with identical source does not ⇒ the divergence is invisible in the repository |
| CR8 | The `Min Instance CPU/Memory (Request-based billing)` SKU meters **idle-retained instance time**, not configured minimum instances | ⇒ it appears with `min-instances = 0` ⇒ and is **absent** on an instance-based service, because such a service records no idle time |
| CR9 | A throttled sidecar gets CPU **only while a request is in flight** | ⇒ background pollers stall between requests ⇒ batched exporters flush late and can lose a tail at shutdown |
| CR10 | Instance-based cost scales with **instance-hours**, which scale with request *arrival spacing*, not request *count* | ⇒ a handful of requests spread through the day costs more than the same count in one burst ⇒ and the bill grows as traffic arrives more evenly, long before volume matters |

## Diagnosing it

Work from the bill inward. Each step is independently conclusive, so stop when
one answers.

**1. Which SKUs bill?** *(verified — the fastest discriminator)*

Query the billing export for the project's Cloud Run SKUs. A service on
request-based billing produces `Services CPU (Request-based billing)` and
usually `Min Instance CPU (Request-based billing)`. A project showing **only**
`Services CPU (Instance-based billing)` and no request-based line at all is on
instance-based billing regardless of what its configuration says.

**2. Is there any idle time?** *(verified — confirms the mechanism)*

```
metric:    run.googleapis.com/container/instance_count
aligner:   ALIGN_MEAN over 3600s
group by:  resource.labels.service_name, metric.labels.state
```

`state` splits into `active` and `idle`. A request-based service accumulates
substantial idle instance-hours. **Zero idle, across every service, is the
signature of CR4.**

**3. Which container is responsible?** *(verified)*

Read the v2 API directly — not `gcloud run services describe`:

```
GET https://run.googleapis.com/v2/projects/<project>/locations/<region>/services/<service>
```

and inspect `.template.containers[].resources.cpuIdle` for **every** container.
Absent means always allocated (CR5). Expect the main container to say `true` and
a sidecar to say nothing at all — that is the trap in its usual form.

**4. Compare the cost against the work.** *(verified)*

Sum request latency over the window and compare with billed vCPU-hours. A
healthy request-based service lands within an order of magnitude. A ratio in the
hundreds — hours billed against minutes worked — means the instance is being
charged for existing.

Take request counts from `run.googleapis.com/request_count`, **not** from
`gcloud logging read`, which silently truncates at `--limit` and can undercount
by more than an order of magnitude.

## Why two environments diverge

The most confusing presentation of this bug is two environments deploying the
**same** Terraform module where one bills correctly and the other does not.

The cause is the write path, not the source (CR7):

| Deploy path | API used | Effect on sidecars |
| :--- | :--- | :--- |
| `gcloud run services update` | v1 round-trip | service-level annotation is normalised onto **every** container — sidecars silently acquire `cpu_idle` |
| `terraform apply` / `tofu apply` | v2, per-container | the spec is honoured **literally** — a sidecar with no `cpu_idle` stays always-allocated |

Two consequences worth stating plainly:

- The environment that looks correct is often the one that is **drifted**, not
  the one that is configured correctly. Its state does not match the repository.
- That drift is **load-bearing and fragile**: the next authoritative
  `terraform apply` against the good environment reverts it to the broken shape,
  turning a working environment into a regression with no source change.

Confirm which wrote last by reading `.client` and `.lastModifier` on the v2
service response — a Terraform apply and a `gcloud` update are distinguishable
there.

## Fixing it

Set CPU throttling on **every** container in the instance, not just the one
serving traffic:

```hcl
containers {
  name = "otel-collector"
  resources {
    limits = { cpu = "0.25", memory = "256Mi" }
    # CPU allocation is an instance-level billing property: one container with
    # CPU always allocated bills the whole instance for its full lifetime,
    # which silently defeats the main container's cpu_idle.
    cpu_idle = true
  }
}
```

Leave a container always-allocated only when it must do work between requests
(CR9) — a queue consumer, or a service whose whole job is background
processing. For those, instance-based billing is correct rather than a bug.

### What you give up

Throttling a sidecar means it receives CPU only while a request is in flight.
Two consequences are common enough to plan for:

- **Polling sidecars go stale.** A policy engine or config agent that refreshes
  on an interval only advances that interval during request processing. If the
  data changes rarely and staleness is tolerable, this is fine; if it gates
  authorization decisions that must be current, it is not.
- **Batched exporters flush late.** A telemetry collector buffers and flushes on
  a timer. Throttled, it flushes when traffic resumes, and a tail of buffered
  data can be lost when the instance is recycled.

Neither is a reason to keep the whole instance on instance-based billing by
default — but both are reasons to decide per sidecar rather than globally.

### What not to do instead

The symptom — a service that bills continuously on trivial traffic — invites
fixes that do not address it:

| Tempting fix | Why it misses |
| :--- | :--- |
| Cache the polling endpoint at the edge | Reduces requests, but retention is driven by arrival *spacing*; the instance still bills whenever anything arrives |
| Raise or lower `min-instances` | Orthogonal — it never controlled the billing model (CR1) |
| Shrink the sidecar's CPU limit | Instance-based billing charges the instance total, not the sidecar's share (CR3) |
| Remove the sidecar | Solves the bill by deleting the capability; `cpu_idle` keeps both |

## Sizing the exposure

Instance-based cost tracks **instance-hours**, so it grows with how evenly
requests arrive rather than how many there are (CR10). A service receiving a
trickle spread across the day can bill near-continuously while one receiving the
same daily count in a single burst bills for minutes.

The practical implication: the bug is cheapest exactly when a service is new and
idle, and grows steadily as real traffic arrives. Finding it on a quiet
pre-launch environment and dismissing it as small is the expensive mistake —
that is the moment it costs least to fix and the point after which it only
grows.

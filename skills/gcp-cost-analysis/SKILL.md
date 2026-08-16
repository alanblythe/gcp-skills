---
name: gcp-cost-analysis
description: >
  This skill should be used when finding out where GCP money actually goes:
  "why is my GCP bill up", "what is driving cost", "how do I query GCP spend",
  "billing export", "cost anomaly", "which resource costs the most", "what is my
  run rate", "why did this month jump", "how do I attribute cost to a service or
  a team", or when a cost figure needs to be trusted before acting on it. Covers
  the BigQuery billing export and its query traps, what the export cannot tell
  you and which metric fills the gap, separating standing cost from usage cost,
  and the extrapolation and comparison mistakes that produce confident wrong
  numbers. Use gcp-cost-cloud-run when Cloud Run specifically bills more than
  its traffic explains.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.1.0
---

# Measuring and attributing GCP cost

Covers how to get a cost number that survives scrutiny, and the specific ways
this analysis goes wrong — most of which produce a plausible figure rather than
an obvious error.

Claims marked **verified** were established by querying a live billing export
and reconciling it against Cloud Monitoring and the resource APIs.

## Quick answers

| Question | Answer |
| :--- | :--- |
| Is there a `gcloud` command for cost? | **No.** The BigQuery billing export is the only real source (A1) |
| Gross or net? | Always **net** — credits live in a nested array and must be summed in (A2) |
| Why does my total disagree with the console? | Usage date vs `invoice.month`; they genuinely differ (A3) |
| Can I see which resource a row is? | **Only in the *detailed* export.** The standard export has no resource name (A4) |
| Can the export split Cloud Run by service? | **No.** Use Cloud Monitoring `billable_instance_time` (A5) |
| Why is today's cost low? | The export lags ~1 day; every window must end **yesterday** (A6) |
| Can I extrapolate a 7-day window to a month? | Only after checking for burst days — a few can carry most of the total (A7) |
| Month-over-month is up 47% — is that real? | Not if an environment was built mid-window; compare equal-length slices and decompose by project (A8) |
| A credit appears on some days only | Separate always-on credits from **promotions**, which start mid-window and expire (A9) |
| Why won't `window` work as an alias? | Reserved word in BigQuery (A10) |
| Most of my spend is unattributed | Labels are the only attribution the export carries; unlabelled call sites are invisible (A11) |
| Which costs are actually cuttable? | Separate standing cost from usage cost first — they respond to different actions (A12) |

## The source

*Verified.* Real cost lives in a **BigQuery billing export** that you enable per
billing account. There is no simple `gcloud` cost command, and the console's
figures are hard to reproduce or automate.

Two exports exist and the difference matters (A4):

| Export | Carries | Use for |
| :--- | :--- | :--- |
| **Standard usage cost** | service, SKU, project, labels, usage amount, cost, credits | almost everything |
| **Detailed usage cost** | the above **plus** `resource.name` / `resource.global_name` | telling two identical SKUs in one project apart |

If you have only the standard export, a row reading "PSC consumer endpoint,
$3.25" in a project holding three endpoints cannot be attributed further from
billing data alone — you must inventory the live project and reason about which
resources exist. **Enabling the detailed export only populates going forward**;
it will not explain last week.

### Net cost

*Verified.* Credits are negative amounts in a nested `credits` array, so a bare
`SUM(cost)` overstates every figure (A2):

```sql
cost + IFNULL((SELECT SUM(c.amount) FROM UNNEST(credits) c), 0) AS net
```

### Query traps

| Trap | Detail |
| :--- | :--- |
| `window` is reserved | Never use it as a column or CTE alias (A10) |
| Usage date ≠ invoice month | `DATE(usage_start_time)` and `invoice.month` disagree — usage late in a month can land on the next invoice. Pick one per figure and say which (A3) |
| Today is always partial | The export lags roughly a day; end every rolling window at **yesterday** (A6) |
| `bq` and non-interactive auth | `bq` hits reauth failures in non-interactive shells and rejects an `--account` flag; pass credentials through environment variables instead |

## Forces

Rules, not preferences. `⇒` means the data gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| A1 | Cost detail exists only in the billing export; the APIs report configuration, not spend | ⇒ any cost claim not traceable to the export is an estimate ⇒ including every figure a resource inventory suggests |
| A2 | Credits are a nested array, and some SKUs are 100% discounted | ⇒ gross and net differ per SKU, not by a flat factor ⇒ a gross figure ranks SKUs wrongly |
| A3 | `invoice.month` is what is actually billed; usage date is when it happened | ⇒ the two never fully reconcile ⇒ mixing them across sections of one report makes it internally inconsistent |
| A4 | The standard export carries no resource identity | ⇒ several instances of one SKU in one project collapse into a single row ⇒ attribution below that row requires inventorying live resources |
| A5 | Some services bill as one project-level line regardless of internal structure — Cloud Run does not split by service | ⇒ per-service attribution must come from Cloud Monitoring ⇒ `billable_instance_time`, aligned and grouped by `service_name` |
| A6 | The export lags ~1 day | ⇒ today always reads low ⇒ a window including today understates, and a daily anomaly check on today's data fires wrongly |
| A7 | Bursty workloads concentrate spend into a few days | ⇒ a 7-day window's placement changes the run-rate materially ⇒ always look at the daily series before multiplying |
| A8 | Environments get built, and new infrastructure has no prior-period baseline | ⇒ month-over-month growth in a young estate mostly measures construction ⇒ decompose by project **and** service before attributing a rise to a cause |
| A9 | Promotional credits start and stop mid-window, and are usually SKU-specific | ⇒ a net figure can be quietly half the real rate ⇒ and reverts without any change on your side |
| A10 | `window` is a BigQuery reserved word | ⇒ queries fail confusingly when it is used as an alias |
| A11 | Labels are the only attribution the export carries beyond project and SKU | ⇒ unlabelled call sites are permanently unattributable ⇒ and the fastest-growing service is usually the least labelled |
| A12 | Standing cost bills on existence; usage cost bills on work | ⇒ they respond to different actions ⇒ optimising usage on a bill that is mostly standing cost achieves nothing |

## Standing cost vs usage cost

*Verified.* The single most useful cut of a cloud bill is not by service — it is
by **what makes it bill** (A12).

- **Standing cost** bills because a resource exists. It is flat, it survives
  every idle weekend, and it is only removed by deleting or consolidating the
  resource.
- **Usage cost** bills because work happened. It tracks demand and responds to
  caching, batching, and efficiency work.

Classify from the daily series rather than by intuition: a SKU billing all seven
days with a low coefficient of variation is standing cost; one with a high CV is
usage cost. State the thresholds you used — it is your classification, not
Google's, and a seven-point sample is small.

Standing cost is where surprises hide, because nobody watches a line that never
changes. Recurring examples, with observed unit rates:

| Standing item | Observed rate | Note |
| :--- | :--- | :--- |
| Load balancer forwarding rule minimum | ~$18/month **each** | Often the largest single fixed line; consolidating onto one shared frontend is the lever |
| Private Service Connect consumer endpoint | ~$0.0100/hour ≈ $7.30/month **each** | Multiplies quietly, one per connected service per environment |
| Cloud Armor policy + rules | flat per policy and per rule | Frequently **identical** in non-production and production |
| Cloud NAT gateway + reserved IP | flat per gateway | Load-bearing where a static egress IP is required — check before cutting |
| DNS managed zone | ~$0.19/month each | Negligible individually; only worth counting in aggregate |

### Non-production parity is a cost category

A staging environment reproducing production's network stack — WAF, NAT, private
endpoints, dedicated load balancing — pays production's **standing** cost while
producing none of production's value. This rarely appears as a line item because
it is spread across services.

Worth asking explicitly: which production controls does the non-production
environment need for fidelity, and which are there only because the module is
shared? "The environments should match" is a defensible answer; it is just not
usually a priced one.

## Attribution

*Verified.* Beyond project and SKU, the export carries only **labels** (A11).
Whatever is unlabelled is unattributable, permanently and retroactively.

This bites hardest on managed AI and data services, where a single project-level
SKU aggregates every call site. When the fastest-growing line on the bill is
also the least attributed, labelling those call sites is usually worth more than
any single optimisation — you cannot prioritise what you cannot split.

When the export cannot split a service, Cloud Monitoring often can (A5):

| Question the export can't answer | Metric |
| :--- | :--- |
| Which Cloud Run service costs what | `run.googleapis.com/container/billable_instance_time` |
| Is an instance working or merely retained | `run.googleapis.com/container/instance_count`, grouped by `metric.labels.state` |
| How many requests actually arrived | `run.googleapis.com/request_count` |

**Do not count requests from logs.** `gcloud logging read` truncates silently at
`--limit`; verified undercounts of more than 30× against `request_count` on the
same window. Logs are for *what* was requested; metrics are for *how much*.

## Reporting cost honestly

Cost analysis is unusually prone to confident wrong answers, because every
intermediate number looks reasonable. Four disciplines carry most of the weight:

**Show the daily series before extrapolating** (A7). A run-rate derived from a
window containing bursts is an artifact of where the window fell. If three days
carry 80% of a total, say so next to the run-rate rather than in a footnote.

**Decompose before attributing** (A8). "The increase is service X" is a claim
about a delta, and deltas decompose by project and service. A rise concentrated
in one project while a different service grew in another is two movements, not
one — and they may have opposite causes.

**Separate credits from discounts** (A9). An always-on 100%-discounted SKU and a
time-limited promotion both show as credits and behave completely differently.
Check *which days* each credit appears on: a credit present on every day at a
constant rate is structural; one appearing partway through is a promotion with
an expiry you have not priced.

**Reconcile independent sources, and say when they disagree.** Billing export,
`billable_instance_time`, and `instance_count` can give three different answers
for the same window. That is not a reason to discard the analysis — but it is a
reason to state which conclusions depend on absolute values and which depend
only on proportions. Conclusions resting on a ratio of hundreds survive a 30%
measurement disagreement; conclusions resting on a 20% difference do not.

## Anomaly detection that works

A useful daily check needs both a floor and a relative test, or it pages on
noise:

- Compare **yesterday** (never today — A6) against a trailing median rather than
  a mean, so one spike does not raise the bar against itself.
- Require an absolute floor as well as a multiple, so cents-scale services cannot
  trigger it.
- Scan for **new SKUs** separately. A SKU that did not exist last week is the
  earliest signal that something was deployed, and it precedes the cost curve
  that follows it.

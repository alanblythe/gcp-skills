# gcp-skills

Skills for running **Google Cloud** without overpaying for it — the parts that are
easy to get wrong and slow to notice: which resources bill for existing rather than
working, why a service with `min-instances = 0` can bill continuously, why identical
Terraform costs different amounts in two environments, and the query mistakes that
make a cost report internally consistent and wrong.

Derived from running deployments. Where a claim was verified by observation —
billing-export SKUs reconciled against Cloud Monitoring and the resource APIs —
the skill marks it; where the configuration and the bill disagree, both are
recorded, because that disagreement is usually the finding.

## Skills

| Skill | Use it for |
| :--- | :--- |
| [`gcp-cost-analysis`](skills/gcp-cost-analysis/SKILL.md) | **Measuring.** Where the money actually goes, and how to get a number that survives scrutiny: the BigQuery billing export and its query traps, what the export cannot tell you and which metric fills the gap, standing cost versus usage cost, attribution through labels, and the extrapolation and comparison mistakes that produce confident wrong answers. |
| [`gcp-cost-cloud-run`](skills/gcp-cost-cloud-run/SKILL.md) | **Diagnosing.** Cloud Run costing more than its traffic explains: the two billing models, why CPU allocation is configured per container but billed per instance, the API that reports it honestly and the one that misleads, why a gcloud-deployed environment silently diverges from a Terraform-deployed one, and what you give up by fixing it. |

The split is measurement and diagnosis. The analysis skill is what you reach for
when the bill is up and the cause is unknown; the Cloud Run skill is what you
reach for once the cause is narrowed to a service whose cost has come loose from
its traffic.

## Why these two first

Both came out of the same investigation. A production Cloud Run deployment was
billing **122 vCPU-hours per week** against roughly **half a vCPU-hour** of actual
request work — a ratio of about 250:1 — while `min-instances` was `0` everywhere
and the configuration appeared to request the cheap billing model.

The cause was that CPU allocation in Cloud Run is configured per container but
**billed per instance**: two sidecars had never been given `cpu_idle`, so the whole
instance billed for its full lifetime. The v1 API reports a single service-level
value that hides the divergence, which is why the contradiction looked
unresolvable from `gcloud run services describe`. A sibling environment on the
same Terraform module looked correct only because its deploy path round-trips
through v1 and normalises the setting onto every container — its correct state was
drift, and the next authoritative apply would have reverted it.

Nothing in that chain is exotic. It is the ordinary shape of cloud cost bugs:
a setting that is silently per-instance, a diagnostic surface that reports the
wrong layer, and two environments that disagree for reasons invisible in the
repository.

## Install

Both installers read the `skills/` directory at the repo root.

```bash
# Claude Code (global)
npx -y skills add https://github.com/alanblythe/gcp-skills -y --agent claude-code -g

# Project scope instead of global — drop the -g and run from the project directory
npx -y skills add https://github.com/alanblythe/gcp-skills -y --agent claude-code

# Antigravity / Gemini CLI
npx -y skills add https://github.com/alanblythe/gcp-skills -y --agent antigravity -g
```

To use them without installing, point your agent at
[`skills/gcp-cost-cloud-run/SKILL.md`](skills/gcp-cost-cloud-run/SKILL.md)
directly — each is a single self-contained Markdown file.

## Layout

```
plugin.json               # Claude Code plugin manifest (duplicated in .claude-plugin/)
.claude-plugin/plugin.json
gemini-extension.json     # Antigravity / Gemini extension manifest
skills/
  README.md
  gcp-cost-analysis/
    SKILL.md              # YAML frontmatter (name, description, metadata) + body
  gcp-cost-cloud-run/
```

## Conventions

Each skill follows the same shape:

- **Quick answers** — the questions that bring people here, answered in one line
  each, with a pointer to the force that explains it.
- **Forces** — numbered rules with their consequences. `⇒` means the platform
  gives no choice. These are cited from the rest of the skill so a claim can
  always be traced to the constraint behind it.
- **Verified vs documented** — observed behaviour is marked, so a reader can tell
  a measured claim from a published one. Where they conflict, both are kept.

## License

Apache-2.0.

# DC/DR Drill — Documentation & Runbook

Reference documentation for the SBIePay DC/DR (Data Center / Disaster Recovery) drill pipeline: a GitOps-based switchover and switchback mechanism that toggles traffic between the DC and DR clusters for `prod` and `preprod` environments.

**This repository is documentation only.** The pipeline itself lives in the `cicd-templates` repo and runs from the `deployment` repo's GitLab CI/CD. Copies of the four source YAML files are kept under [`reference-yaml/`](reference-yaml/) for offline reading, but the templates repo is always the authoritative, runnable source — if this documentation and the live YAML ever disagree, the YAML wins and this repo needs updating.

---

## What this drill does, in one paragraph

An operator runs a pipeline in the `deployment` repo, sets one variable (`DRILL_TIER`), and a set of manual buttons appear grouped into four stages: put up a maintenance page, scale down one cluster's application services, scale up the other cluster's, and flip Kafka MirrorMaker2 replication direction. Every action is GitOps — the job edits a Helm `values.yaml` file in the deployment repo, commits, and pushes; ArgoCD detects the change and reconciles the cluster. The scale jobs additionally log into OpenShift directly and scale the live deployments, so the effect is immediate rather than waiting on ArgoCD's sync interval.

---

## Quick facts

| | |
|---|---|
| **Where it runs** | `deployment` repo → CI/CD → Pipelines → **Run pipeline** |
| **How it's triggered** | Manually setting the `DRILL_TIER` variable at pipeline run time |
| **Valid values** | `prod`, `preprod`, `nonprod` *(see [Known Issues](docs/06-known-issues.md) — `nonprod` currently does nothing)* |
| **Deployment mechanism** | GitOps (git push → ArgoCD sync) + direct `oc scale` for the scale jobs |
| **Every job** | Manual — nothing runs automatically once the pipeline starts |
| **Source of truth repo** | `itepaypg-sbiepay2/infra/devops/cicd-templates`, path `templates/cd/deploy.drill.yml` + `ops/drill/*.yml` |
| **Triggered from** | `itepaypg-sbiepay2/infra/devops/deployment`, `.gitlab-ci.yml` |

---

## Start here

| If you want to... | Read |
|---|---|
| Understand how this fits together and why | [`docs/01-architecture.md`](docs/01-architecture.md) |
| Know exactly which variables/tokens must exist before you run this | [`docs/02-variables.md`](docs/02-variables.md) |
| See every job, what it does, and when it appears | [`docs/03-job-reference.md`](docs/03-job-reference.md) |
| Run a DC → DR switchover | [`docs/04-switchover-runbook.md`](docs/04-switchover-runbook.md) |
| Run a DR → DC switchback | [`docs/05-switchback-runbook.md`](docs/05-switchback-runbook.md) |
| See what's verified vs. what still needs confirmation before a real prod run | [`docs/06-known-issues.md`](docs/06-known-issues.md) — **read this before your first prod drill** |

---

## Absolute minimum you need to know before clicking anything

1. **This is production infrastructure.** The `prod` tier scales real application services and flips real Kafka replication on the live payment gateway. There is no dry-run mode.
2. **Read [`docs/06-known-issues.md`](docs/06-known-issues.md) first.** Three of the four OpenShift cluster API URLs used by the scale jobs are marked as unconfirmed directly in the source YAML. Confirm them before a real prod drill — running against a wrong or stale URL is the single highest-risk failure mode of this pipeline.
3. **`DRILL_TIER=nonprod` currently launches a pipeline with no jobs in it.** This is a known gap, not a mistake on your part — see Known Issues.
4. **Follow the stage order.** Maintenance → DC → DR → MirrorMaker2 is not arbitrary; each stage assumes the previous one completed. Jump the order and you risk serving traffic from a cluster with no active replication feeding it.

---

## Repository layout

```
dcdr-drill-runbook/
├── README.md                          — this file
├── docs/
│   ├── 01-architecture.md             — how the pipeline is wired, GitOps flow, diagrams
│   ├── 02-variables.md                — every variable and token, required vs. computed
│   ├── 03-job-reference.md            — every job: name, stage, tier, target, effect
│   ├── 04-switchover-runbook.md       — DC → DR, step by step
│   ├── 05-switchback-runbook.md       — DR → DC, step by step
│   └── 06-known-issues.md             — verified gaps and unconfirmed items
└── reference-yaml/                    — read-only copies of the 4 source files
    ├── deploy.drill.yml               — orchestrator (stages, includes, tier map)
    ├── drill.maintenance.yml          — maintenance page ON/OFF jobs
    ├── drill.scale.yml                — DC/DR application scale jobs
    └── drill.mirrormaker2.yml         — Kafka MirrorMaker2 toggle jobs
```

---

## Change history

Keep this section updated whenever the drill YAML changes in the templates repo, so this documentation doesn't silently go stale.

| Date | Change | Source commit / note |
|---|---|---|
| 2026-07 | Initial documentation drop, verified against `cicd-templates` as of this date | — |

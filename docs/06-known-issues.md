# Known Issues & Unverified Items

Everything on this page was found by reading the actual source YAML, not inferred. Each item states exactly what the source says, why it matters operationally, and what needs to happen before it stops being a risk.

**Read this in full before your first production drill.**

---

## 1. `DRILL_TIER=nonprod` currently launches an empty pipeline

**What the source says:** `templates/cd/deploy.drill.yml`'s header comment documents a `nonprod` tier that should scale `dev` (scale-test only, no maintenance page, no MirrorMaker2). But the orchestrator only includes three files — `drill.maintenance.yml`, `drill.scale.yml`, `drill.mirrormaker2.yml` — and **none of them contain a single job with `rules: - if: '$DRILL_TIER == "nonprod"'`**. There is no fourth, nonprod-specific file anywhere in the repository.

**Effect:** running the pipeline with `DRILL_TIER=nonprod` creates a pipeline with all four stages present but **zero jobs in any of them**. Nothing fails — there is simply nothing to click.

**Status:** documented gap, not yet fixed. If a nonprod scale capability is needed, a new job file (e.g. `ops/drill/drill.scale.nonprod.yml`) needs to be written and included in `deploy.drill.yml`, following the same pattern as the prod/preprod scale jobs but targeting the `dev` namespace.

**Until fixed:** do not set `DRILL_TIER=nonprod` expecting it to do anything.

---

## 2. Three of four OpenShift cluster API URLs are marked unconfirmed in the source

**What the source says**, verbatim, from `ops/drill/drill.scale.yml`:

| Job pair | `OC_API_URL` | Source annotation |
|---|---|---|
| `Prod_DC_Scale_All_Down/Up` | `https://api.dc.prod.epay.sbi:6443` | `# ← confirm URL` |
| `Prod_DR_Scale_All_Down/Up` | `https://api.dr.prod.epay.sbi:6443` | *(no annotation — appears confirmed)* |
| `PreProd_DC_Scale_All_Down/Up` | `https://api.dcpreprod.epay.sbi:6443` | `# ← confirm URL` |
| `PreProd_DR_Scale_All_Down/Up` | `https://api.preprod.epay.sbi:6443` | `# ← confirm URL` |

**Effect:** if any of the three flagged URLs is wrong or stale, the corresponding `oc login` step fails immediately — the job errors out cleanly rather than silently doing nothing, so this is a fail-safe rather than a fail-silent risk. But it means a first real run against a flagged URL should be treated as unverified until proven otherwise.

**Status:** owner-confirmed as pending — to be verified and corrected in the source before relying on this pipeline for a real prod or preprod drill.

**Action before first real drill:** confirm each of the three flagged URLs against the actual OpenShift API endpoint for that cluster (`oc whoami --show-server` run against a known-good session, or your cluster inventory), update `ops/drill/drill.scale.yml` if any are wrong, and remove the `# ← confirm URL` comment once verified so this table can be updated to reflect "confirmed."

---

## 3. Preprod's live-scale namespace map only covers the `admin` service

**What the source says**, from `ops/drill/drill.scale.yml`, the `.preprod_ns_map` block:

```yaml
.preprod_ns_map:
  variables:
    NS_MAP: |
      declare -A NS_DEPLOYMENTS
      NS_DEPLOYMENTS["${TARGET_ENV}-admin"]="admin-adminservice"
      # NS_DEPLOYMENTS["${TARGET_ENV}-frontend"]="merchantfe-merchant-frontend transactionfe-transaction-frontend"
      # NS_DEPLOYMENTS["${TARGET_ENV}-kms"]="kms-kmsservice"
      # NS_DEPLOYMENTS["${TARGET_ENV}-rns"]="ops-operationsservice refund-refundservice"
      # NS_DEPLOYMENTS["${TARGET_ENV}-merchant"]="merchant-merchantservice report-reportservice"
      # NS_DEPLOYMENTS["${TARGET_ENV}-simulators"]="demo-merchantsimulator javasimulator-javautilityapisimulator orderhash"
      # NS_DEPLOYMENTS["${TARGET_ENV}-transaction"]="payment-paymentservice txn-transactionservice"
```

Every group except `-admin` is commented out. Compare this to `.prod_ns_map`, which has all seven groups active.

**Effect:** this creates a **mismatch between the two halves of what a preprod scale job does.** The GitOps half (patching `values.yaml` for `replicaCount` across all 14 services in `ALL_CHARTS`) still runs for every service — that part is tier-agnostic. But the **live** `oc scale` half — the part that takes immediate effect on the running cluster without waiting for ArgoCD — only touches the `admin-adminservice` deployment in preprod. Every other service's live pod count is left untouched until ArgoCD's next sync reconciles the GitOps change.

**Practical consequence:** a `PreProd_DC_Scale_All_Down` run will look complete (job succeeds, GitOps commit pushed for all 14 services) but only one service actually scales immediately. If you're using a preprod drill to rehearse timing or to validate the live-scale mechanism itself, be aware you are currently only rehearsing it for one service.

**Status:** appears to be an intentional, partial rollout (the commented lines are a template, not a mistake in formatting) — but confirm this is the intended current scope for preprod before relying on preprod drill timing as a proxy for prod drill timing.

---

## 4. `Scale_All_Up` always sets `REPLICA_COUNT=1` — never the service's original count

**What the source says:** every `*_Scale_All_Up` job across both tiers hardcodes `REPLICA_COUNT: "1"`. There is no mechanism anywhere in these files that records what a service's replica count was *before* a `Scale_All_Down`, and no mechanism that restores that original count on the corresponding `Scale_All_Up`.

**Effect:** for any service whose normal steady-state replica count is greater than 1, a switchover-then-switchback cycle will leave it running at exactly 1 replica afterward — silently undersized — unless something else (an HPA reconciling `autoscaling.enabled: true`, or a person) corrects it. `AUTOSCALING_ENABLED` is also set to `"true"` on every `Scale_All_Up` job, so if HPAs are configured with a sensible minimum replica count for these services, they may self-correct — but this depends entirely on HPA configuration per service, which these drill files neither set nor verify.

**Status:** by design, not a bug — but a real operational risk if any of the 14 services runs more than 1 replica normally. Both runbooks ([switchover](04-switchover-runbook.md), [switchback](05-switchback-runbook.md)) include an explicit checklist item for this.

**Action:** before relying on this pipeline for a real DR event, confirm which of the 14 `ALL_CHARTS` services have HPA configured with an appropriate minimum, and for any that don't, add a manual replica-count verification step to your runbook checklist (already present in the switchback runbook, Step 4).

---

## 5. Commit message tagging is inconsistent across the three job files

**What the source says:**

| File | Commit message format |
|---|---|
| `drill.scale.yml` | `drill(<env>): replicas=<n> autoscaling=<bool> [ci skip]` |
| `drill.maintenance.yml` | `drill(<env>): maintenance=<bool> replicas=<n>` — **no `[ci skip]`** |
| `drill.mirrormaker2.yml` | `drill(<env>): mirrorMaker2.enabled=<bool>` — **no `[ci skip]`** |

**Effect:** checked against the `deployment` repo's actual router rules (see [Architecture](01-architecture.md)), this is currently **functionally inert** — neither `deploy.single.yml` nor `deploy.drill.yml` has a rule matching plain `CI_PIPELINE_SOURCE == "push"`, so a bare push to the `deployment` repo (with or without `[ci skip]`) does not create any drill or app-deploy jobs regardless. The `[ci skip]` tag on the scale commits is currently redundant, not load-bearing.

**Status:** cosmetic inconsistency, not a functional bug, under the router rules as they exist today. Worth normalizing for consistency if the job files are touched again, but not urgent — and if the router rules ever change to match on `push`, revisit this immediately, since the inconsistency would then become load-bearing.

---

## Summary table

| # | Issue | Severity | Status |
|---|---|---|---|
| 1 | `nonprod` tier produces an empty pipeline | Medium — misleading, not dangerous | Documented gap |
| 2 | 3 of 4 OCP cluster URLs unconfirmed | **High — verify before first real prod run** | Owner confirming |
| 3 | Preprod live-scale only covers `admin` service | Medium — limits preprod drill scope | Appears intentional, confirm scope |
| 4 | Scale-up always resets to 1 replica | **High for any multi-replica service** | By design — mitigate via checklist / HPA |
| 5 | Inconsistent `[ci skip]` tagging | Low — currently inert | Cosmetic |

# Variables & Tokens Reference

## Variable you set — required for every run

| Variable | Where to set it | Valid values | Notes |
|---|---|---|---|
| `DRILL_TIER` | GitLab UI → Run pipeline → Variables | `prod` \| `preprod` \| `nonprod` | This is the ONLY variable an operator needs to set. It both triggers the drill router (see [Architecture](01-architecture.md)) and gates which jobs appear. **`nonprod` currently produces an empty pipeline — see [Known Issues](06-known-issues.md).** |

That is the entire operator-facing surface. Everything else below is either a pre-provisioned CI/CD variable (set once by an admin, not per-run) or a variable computed internally by the pipeline that you never touch.

---

## Secrets required (provisioned once, group or project level)

These must exist as **masked and protected** GitLab CI/CD variables before any drill can run. None of them are passed at pipeline-run time.

| Variable | Used by | Purpose | Scope needed |
|---|---|---|---|
| `GITLAB_CI_DEPLOY_TOKEN` | all three job files | Authenticates the `git clone`/`git push` against the `deployment` repo | `write_repository` on the `deployment` repo (and any repo this pipeline needs to push to) |
| `OCP_TOKEN_PROD_DC` | `drill.scale.yml` — `Prod_DC_Scale_All_*` | `oc login` to the prod DC cluster for live scaling | Service account with permission to `oc scale` in the prod-dc namespaces |
| `OCP_TOKEN_PROD_DR` | `drill.scale.yml` — `Prod_DR_Scale_All_*` | `oc login` to the prod DR cluster | Same, for prod-dr |
| `OCP_TOKEN_PREPROD_DC` | `drill.scale.yml` — `PreProd_DC_Scale_All_*` | `oc login` to the preprod DC cluster | Same, for pre-prod-dc |
| `OCP_TOKEN_PREPROD_DR` | `drill.scale.yml` — `PreProd_DR_Scale_All_*` | `oc login` to the preprod DR cluster | Same, for pre-prod |

**Five secrets total.** Before your first drill run in a given environment, confirm all five exist, are masked, and are scoped correctly — a missing or expired token fails the job outright (`oc login` or `git push` errors), and because every job is manual, a failure here is caught the moment you click the button rather than mid-sequence.

---

## Variables computed internally (declared in `deploy.drill.yml`, not set by the operator)

| Variable | Value | Purpose |
|---|---|---|
| `DEPLOYMENT_BRANCH` | `"main"` | Branch the drill jobs clone, edit, and push to |
| `DEPLOYMENT_REPO_URL` | `https://oauth2:$GITLAB_CI_DEPLOY_TOKEN@gitlab.sbi/itepaypg-sbiepay2/infra/devops/deployment.git` | Full authenticated clone URL, built once and reused by `drill.scale.yml` / `drill.mirrormaker2.yml` |
| `MAINTENANCE_CHART` | `"epay_maintenance_fe"` | The Helm chart whose `values.yaml` the maintenance jobs edit |
| `ALL_CHARTS` | Comma-separated list, 14 services (see below) | The chart list the scale jobs iterate over for the GitOps `replicaCount` patch |
| `GIT_STRATEGY` | `none` | Every drill job sets this — the runner does not need to check out the `cicd-templates` repo itself, since each job clones the `deployment` repo directly inside its script |

### `ALL_CHARTS` — the 14 services patched by every scale operation

```
epay_admin_service, epay_key_managment, epay_merchant_fe, epay_merchant_service,
epay_merchant_simulator, epay_notification_service, epay_operations_service,
epay_order_hash, epay_payment_service, epay_refund_service, epay_reports_service,
epay_transaction_fe, epay_transaction_service, java-utility-api-simulator
```

This list drives the **GitOps** side of a scale operation (the `values.yaml` `replicaCount` patch) for every scale job in both `prod` and `preprod` tiers. It does **not** drive the **live `oc scale`** step — that uses a separate, hand-maintained namespace map (`NS_MAP`, below), and the two lists do not currently cover the same set of services for `preprod`. See [Known Issues](06-known-issues.md).

---

## Per-job variables (set on each individual job, not shared)

These vary by job and are what actually differentiates, say, `Maintenance_ON` from `Maintenance_OFF`.

| Variable | Used in | Meaning |
|---|---|---|
| `ENV_LIST` | maintenance jobs | Space-separated list of environments to update in one job run, e.g. `"prod-dc prod-dr"` |
| `REPLICA_COUNT` | maintenance jobs, scale jobs | Target replica count. `1` = maintenance page ON (or service scaled up); `0` = maintenance page OFF (or service scaled down) |
| `MAINTENANCE_ENABLED` | maintenance jobs | `"true"`/`"false"` — patches the chart's `maintenance.enabled` field alongside `replicaCount` |
| `TARGET_ENV` | scale jobs, mirrormaker2 jobs | Single environment name this job acts on, e.g. `"prod-dc"`, `"pre-prod"` (note: DR environments are **not** suffixed `-dr` in the folder path — see the table in [Job Reference](03-job-reference.md)) |
| `AUTOSCALING_ENABLED` | scale jobs | `"true"`/`"false"` — patches the chart's `autoscaling.enabled` field alongside `replicaCount` |
| `CHARTS` | scale jobs | Always set to `"$ALL_CHARTS"` — the 14-service list above |
| `OC_API_URL` | scale jobs | The OpenShift API endpoint this job logs into. **3 of 4 are marked unconfirmed in source — see Known Issues** |
| `OC_TOKEN` | scale jobs | Which of the four `OCP_TOKEN_*` secrets this job uses |
| `NS_MAP` | scale jobs | A literal block of bash (`declare -A NS_DEPLOYMENTS; NS_DEPLOYMENTS[...]=...`) that the job `eval`s to build the namespace → deployment-name map for the live `oc scale` step. Declared once per tier (`.prod_ns_map`, `.preprod_ns_map`) and inherited via `extends:` — not duplicated per job |
| `MIRRORMAKER2_ENABLED` | mirrormaker2 jobs | `"true"`/`"false"` — patches `mirrorMaker2.enabled` in the `epay_common` chart |

### The prod namespace map (`NS_MAP`), fully expanded

Every prod scale job inherits this map via `.prod_ns_map`. `${TARGET_ENV}` is substituted at runtime (`prod-dc` or `prod-dr`).

| Namespace suffix | Deployments scaled |
|---|---|
| `-admin` | `admin-adminservice` |
| `-frontend` | `merchantfe-merchant-frontend`, `transactionfe-transaction-frontend` |
| `-kms` | `kms-kmsservice` |
| `-rns` | `ops-operationsservice`, `refund-refundservice` |
| `-merchant` | `merchant-merchantservice`, `report-reportservice` |
| `-simulators` | `demo-merchantsimulator`, `javasimulator-javautilityapisimulator`, `orderhash` |
| `-transaction` | `payment-paymentservice`, `txn-transactionservice` |

One line is present but commented out in the source: `-notification` → `notify-notificationservice`. It is not currently scaled by the live `oc scale` step in either tier.

### The preprod namespace map — see Known Issues

The `preprod` version of this map has every group **except `-admin` commented out**. This is documented in detail in [Known Issues](06-known-issues.md) because it materially changes what a preprod drill actually scales on the live cluster.

# Job Reference

Every job the drill pipeline can create, verified directly against the source YAML. **20 jobs total** — 10 appear when `DRILL_TIER=prod`, 10 when `DRILL_TIER=preprod`, 0 when `DRILL_TIER=nonprod` (see [Known Issues](06-known-issues.md)). All jobs are `when: manual`.

---

## Stage: Maintenance_Page

*Source: `ops/drill/drill.maintenance.yml`*

| Job | Tier | `ENV_LIST` | `REPLICA_COUNT` | `MAINTENANCE_ENABLED` | Effect |
|---|---|---|---|---|---|
| `Maintenance_ON` | prod | `prod-dc prod-dr` | `1` | `true` | Puts the maintenance page up on **both** prod clusters in one job |
| `Maintenance_OFF` | prod | `prod-dc prod-dr` | `0` | `false` | Removes the maintenance page from both prod clusters |
| `PreProd_Maintenance_ON` | preprod | `pre-prod-dc pre-prod` | `1` | `true` | Puts the maintenance page up on both preprod clusters |
| `PreProd_Maintenance_OFF` | preprod | `pre-prod-dc pre-prod` | `0` | `false` | Removes the maintenance page from both preprod clusters |

Note: each job here updates **both** environments in its tier in a single run (the job loops over `ENV_LIST`) — there is no separate DC-only or DR-only maintenance toggle. `REPLICA_COUNT: 1` means the maintenance-page pod is scaled to 1 replica (i.e. running / visible); `0` means it's scaled down (not serving).

File this job writes to: `<env>/charts/epay_maintenance_fe/values.yaml`

---

## Stage: DC_Application_Services

*Source: `ops/drill/drill.scale.yml`*

| Job | Tier | `TARGET_ENV` | `REPLICA_COUNT` | `OC_API_URL` | `OC_TOKEN` |
|---|---|---|---|---|---|
| `Prod_DC_Scale_All_Down` | prod | `prod-dc` | `0` | `https://api.dc.prod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PROD_DC` |
| `Prod_DC_Scale_All_Up` | prod | `prod-dc` | `1` | `https://api.dc.prod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PROD_DC` |
| `PreProd_DC_Scale_All_Down` | preprod | `pre-prod-dc` | `0` | `https://api.dcpreprod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PREPROD_DC` |
| `PreProd_DC_Scale_All_Up` | preprod | `pre-prod-dc` | `1` | `https://api.dcpreprod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PREPROD_DC` |

⚠️ = marked `# ← confirm URL` directly in the source file. See [Known Issues](06-known-issues.md) before running against a real cluster.

Each of these jobs does two things (see [Architecture](01-architecture.md)): patches `replicaCount`/`autoscaling.enabled` in every chart's `values.yaml` under `<TARGET_ENV>/charts/<chart>/` for all 14 services in `ALL_CHARTS`, **and** logs into the OCP cluster at `OC_API_URL` to `oc scale` the deployments listed in that tier's `NS_MAP` (see [Variables](02-variables.md)).

---

## Stage: DR_Application_Services

*Source: `ops/drill/drill.scale.yml`*

| Job | Tier | `TARGET_ENV` | `REPLICA_COUNT` | `OC_API_URL` | `OC_TOKEN` |
|---|---|---|---|---|---|
| `Prod_DR_Scale_All_Down` | prod | `prod-dr` | `0` | `https://api.dr.prod.epay.sbi:6443` | `$OCP_TOKEN_PROD_DR` |
| `Prod_DR_Scale_All_Up` | prod | `prod-dr` | `1` | `https://api.dr.prod.epay.sbi:6443` | `$OCP_TOKEN_PROD_DR` |
| `PreProd_DR_Scale_All_Down` | preprod | `pre-prod` | `0` | `https://api.preprod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PREPROD_DR` |
| `PreProd_DR_Scale_All_Up` | preprod | `pre-prod` | `1` | `https://api.preprod.epay.sbi:6443` ⚠️ *unconfirmed* | `$OCP_TOKEN_PREPROD_DR` |

Note the asymmetric naming: prod's DR environment folder is `prod-dr`, but preprod's DR environment folder is `pre-prod` (no `-dr` suffix) — the DC one is `pre-prod-dc`. This is not a typo in this documentation; it is exactly what the source YAML sets `TARGET_ENV` to. Confirm this matches your actual deployment repo folder layout (`<env>/charts/...`) before running.

`Prod_DR_Scale_All_*` is the only pair of scale jobs in the file **without** an unconfirmed-URL flag in the source.

---

## Stage: MirrorMaker2

*Source: `ops/drill/drill.mirrormaker2.yml`*

| Job | Tier | `TARGET_ENV` | `MIRRORMAKER2_ENABLED` |
|---|---|---|---|
| `Prod_DC_MirrorMaker2_Down` | prod | `prod-dc` | `false` |
| `Prod_DC_MirrorMaker2_Up` | prod | `prod-dc` | `true` |
| `Prod_DR_MirrorMaker2_Down` | prod | `prod-dr` | `false` |
| `Prod_DR_MirrorMaker2_Up` | prod | `prod-dr` | `true` |
| `PreProd_DC_MirrorMaker2_Down` | preprod | `pre-prod-dc` | `false` |
| `PreProd_DC_MirrorMaker2_Up` | preprod | `pre-prod-dc` | `true` |
| `PreProd_DR_MirrorMaker2_Down` | preprod | `pre-prod` | `false` |
| `PreProd_DR_MirrorMaker2_Up` | preprod | `pre-prod` | `true` |

`MIRRORMAKER2_ENABLED=true` means replication is **active** (flowing from that environment outward); `false` means replication is **stopped**. These jobs are GitOps-only — there is no live-cluster step, unlike the scale jobs. The file patched is `<TARGET_ENV>/charts/epay_common/values.yaml`, and the update is done with a precision `awk` script that touches only the `mirrorMaker2.enabled` field, leaving every other `enabled:` key elsewhere in the file untouched.

---

## Quick lookup — which jobs appear for a given `DRILL_TIER`

| `DRILL_TIER` | Jobs shown | Count |
|---|---|---|
| `prod` | `Maintenance_ON/OFF`, `Prod_DC_Scale_All_Down/Up`, `Prod_DR_Scale_All_Down/Up`, `Prod_DC/DR_MirrorMaker2_Down/Up` | 10 |
| `preprod` | `PreProd_Maintenance_ON/OFF`, `PreProd_DC_Scale_All_Down/Up`, `PreProd_DR_Scale_All_Down/Up`, `PreProd_DC/DR_MirrorMaker2_Down/Up` | 10 |
| `nonprod` | none — see [Known Issues](06-known-issues.md) | 0 |

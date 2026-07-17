# Runbook: Switchover (DC → DR)

Use this to move production traffic from the DC cluster to the DR cluster — a planned drill or a real failover.

**Before you start:** confirm the five secrets in [Variables](02-variables.md) exist, and read the [Known Issues](06-known-issues.md) callouts for your tier — in particular, the fixed `REPLICA_COUNT=1` behavior on scale-up matters for this exact runbook (see the note after Step 3 below).

---

## 1. Start the pipeline

1. Go to the `deployment` repo → **CI/CD → Pipelines → Run pipeline**.
2. Branch: `main`.
3. Add a variable:
   - Key: `DRILL_TIER`
   - Value: `prod` (or `preprod` for a non-production drill rehearsal)
4. Click **Run pipeline**.

The pipeline starts with all four stages visible, each containing the manual job(s) for your chosen tier (10 jobs — see [Job Reference](03-job-reference.md)).

---

## 2. Stage 1 — Maintenance Page

Click:

- **`Maintenance_ON`** (prod tier) or **`PreProd_Maintenance_ON`** (preprod tier)

This is a single job that puts the maintenance page up on **both** DC and DR simultaneously (`ENV_LIST` covers both). Wait for it to complete before proceeding — this ensures no user traffic is mid-flight before you start moving replicas.

---

## 3. Stage 2 — Scale DC down

Click:

- **`Prod_DC_Scale_All_Down`** (prod) or **`PreProd_DC_Scale_All_Down`** (preprod)

This patches every chart in `ALL_CHARTS` to `replicaCount: 0` via GitOps, then directly `oc scale`s every deployment in the DC namespace map to 0 replicas. Confirm this job completes successfully before moving to DR — do not proceed on a failed or partial run.

> **⚠️ Note on the eventual switchback:** this job scales DC to exactly 0. When you later switch back (see the [switchback runbook](05-switchback-runbook.md)), the corresponding `Scale_All_Up` job brings everything back to exactly **1** replica — not whatever each service's original replica count was before the drill. If any service normally runs more than 1 replica in steady state, it will come back undersized until your HPA (if configured) scales it up, or someone manually corrects it. Plan for this now, not during the switchback.

---

## 4. Stage 3 — Scale DR up

Click:

- **`Prod_DR_Scale_All_Up`** (prod) or **`PreProd_DR_Scale_All_Up`** (preprod)

This is the point where DR starts actually serving traffic. Confirm the deployments come up healthy (readiness probes passing, no crash-loops) before proceeding to MirrorMaker2 — this is the highest-impact step in the whole sequence.

---

## 5. Stage 4 — Flip MirrorMaker2

Click, in this exact order:

1. **`Prod_DC_MirrorMaker2_Down`** (prod) or **`PreProd_DC_MirrorMaker2_Down`** (preprod) — stop replication flowing out of DC.
2. **`Prod_DR_MirrorMaker2_Up`** (prod) or **`PreProd_DR_MirrorMaker2_Up`** (preprod) — start replication flowing out of DR.

Do not reverse this order. Stopping DC's outbound replication first, then starting DR's, avoids a window where both sides are simultaneously replicating and risking a split-brain conflict on the Kafka topics involved.

---

## 6. Verify

At minimum, confirm:

- DR application pods are `Running` and passing readiness checks.
- Traffic is landing on DR (check your load balancer / ingress metrics, or application-level health endpoints).
- MirrorMaker2 is actively replicating from DR (check its own health/metrics, not just that the job succeeded).
- The maintenance page is still up until you're confident DR is stable, then run `Maintenance_OFF` / `PreProd_Maintenance_OFF` to restore normal traffic.

---

## 7. Audit trail

Every step above leaves three independent records:

1. **GitLab pipeline record** — who ran it, when, which `DRILL_TIER`, which jobs were clicked and in what order (visible in the pipeline's job timeline).
2. **Git commits in the `deployment` repo** — one commit per environment per change, message format `drill(<env>): <field>=<value>`.
3. **ArgoCD sync history** — exactly what was applied to each cluster and when.

No separate audit logging is needed; these three sources together are sufficient for a post-drill review.

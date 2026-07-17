# Runbook: Switchback (DR → DC)

Use this to move production traffic back from DR to DC after a switchover, once DC is confirmed healthy and ready to resume serving traffic.

The sequence is the switchover run **in reverse** — MirrorMaker2 first, then DR down, then DC up, then maintenance off last. This matters for the same reason the switchover order matters: each stage assumes the previous one is complete.

---

## 1. Start the pipeline

Same as the switchover runbook: `deployment` repo → **Run pipeline** → set `DRILL_TIER` to `prod` or `preprod` → **Run pipeline**.

If you are continuing directly from a switchover in the *same* pipeline run, the jobs from stages already used are still visible and re-clickable — but for a switchback happening later (a separate incident/drill window), start a fresh pipeline run.

---

## 2. Stage 4 — Flip MirrorMaker2 back

Click, in this exact order:

1. **`Prod_DR_MirrorMaker2_Down`** (prod) or **`PreProd_DR_MirrorMaker2_Down`** (preprod) — stop replication flowing out of DR.
2. **`Prod_DC_MirrorMaker2_Up`** (prod) or **`PreProd_DC_MirrorMaker2_Up`** (preprod) — start replication flowing out of DC.

Same split-brain reasoning as the switchover: stop the currently-active side first, then start the target side.

---

## 3. Stage 3 — Scale DR down

Click:

- **`Prod_DR_Scale_All_Down`** (prod) or **`PreProd_DR_Scale_All_Down`** (preprod)

Confirm this completes and DR's deployments are actually at 0 before proceeding — you don't want two clusters simultaneously serving traffic during the transition.

---

## 4. Stage 2 — Scale DC up

Click:

- **`Prod_DC_Scale_All_Up`** (prod) or **`PreProd_DC_Scale_All_Up`** (preprod)

> **⚠️ Same replica-count caveat as the switchover runbook applies here in reverse:** this job scales DC back up to exactly **1** replica per service, regardless of what DC was running before the original switchover. If any service's steady-state replica count is normally higher than 1, verify and correct it after this job completes — do not assume the drill restored the "normal" state. This is a fixed value in the source YAML (`REPLICA_COUNT: "1"`), not something the pipeline remembers or restores automatically.

---

## 5. Stage 1 — Maintenance page off

Once DC is confirmed healthy and serving traffic correctly, click:

- **`Maintenance_OFF`** (prod) or **`PreProd_Maintenance_OFF`** (preprod)

This is the last step — don't remove the maintenance page until you've actually verified DC is taking traffic correctly. There is no harm in leaving the maintenance page up a few extra minutes while you check; there is real harm in removing it prematurely and exposing users to a half-migrated state.

---

## 6. Post-switchback checklist

- [ ] Confirm every service's live replica count matches its normal steady-state count (see the Step 4 caveat above) — this is the item most likely to be silently wrong after a switchback.
- [ ] Confirm MirrorMaker2 is healthy and actively replicating from DC.
- [ ] Confirm the maintenance page is down and real traffic is flowing.
- [ ] Review the three audit sources (GitLab pipeline, git commits, ArgoCD sync history — see the switchover runbook) for the full sequence of both the switchover and switchback.
- [ ] If any `Scale_All_Up`/`Down` job was re-run or retried, check the pipeline log for the retry-with-rebase messages to confirm no push conflicts were silently swallowed.

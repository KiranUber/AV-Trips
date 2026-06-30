# Identifying AV Trips — What We Found & Validated

_Last updated: 2026-06-30 • Scope reconciliation month: April 2026 • Dialect: Presto/Trino (Spark twins exist)_

## TL;DR

- The right universe for "AV trips" is the **AV-native spine `amd.fact_amd_trip`**, filtered to **AV-delivered** with two signals: the served vehicle is a flagged AV **or** the trip ran on the self-driving flow.
- This method is **provably complete**: tested against the independent AV vehicle registry, it misses ~**4 trips in 2.38M** (0 completed).
- It is a **strict superset of the DS/PM pull's real AV trips**. For April 2026 it captured **+25,603 AV trips (+6.0%)** that DS/PM misses, while the only **253** trips DS/PM had that we drop were **all non-completed and not even in the AV spine** (i.e., not real AV trips).
- Two corrections were validated and applied: **exclude Polestar** (an `is_autonomous` false-positive) and **drop the `fact_amd_offer` partner lookup** (it changed 0 labels).
- Canonical query: **`av_trips_final_monthly_presto.sql`**.

---

## 1. The question

> Of every AV trip that actually happened in a given month, does our method capture all of them — and how does it differ from the Data Science / PM SQL that's been used so far?

## 2. The two methods, side by side

| | **DS/PM method** (legacy) | **Our validated method** (canonical) |
|---|---|---|
| Source table | `dwh.fact_trip` (general mobility) | `amd.fact_amd_trip` (**AV-native** — every row is an AV job) |
| How a trip is "AV" | INNER JOIN to `driver.dim_earner_gig` where `gig_type ∈ (autonomous transport)` **and `status='ACTIVE'`**, on `ft.driver_uuid = earner_uuid` | `dwh.dim_vehicle.is_autonomous(last_vehicle_uuid)=true` **OR** normalized `flow_type='selfdriving'` |
| Trip key | `ft.uuid` | `job_uuid` (deduped to 1 row/trip) |
| Partner | `kirby_external_data.lookup_amd_partner` org_name | vehicle make → matched make → `partner_name` |
| Known weaknesses | gig churn, wrong join key, mileage fan-out | none material (see validation) |

Both methods share the **same trip-uuid space** (`ft.uuid` == `amd.fact_amd_trip.job_uuid`), so they can be reconciled exactly.

## 3. Our validated definition (what "an AV trip" means)

```
FROM amd.fact_amd_trip
KEEP IF  dim_vehicle.is_autonomous(last_vehicle_uuid) = true
         OR normalized flow_type = 'selfdriving'        -- lower(), strip 'flow_type_' prefix
EXCLUDE  served vehicle make = 'Polestar'               -- is_autonomous false-positive
DEDUP    to one row per job_uuid
```

- **AV-requested** = any row that passes the filter.
- **AV-completed/delivered** = `last_matched_av_offer_ts.completed_timestamp_local IS NOT NULL`.

## 4. Why this is different from — and better than — the DS/PM SQL

The DS/PM SQL identifies AV trips by **who the driver is** (an active AV gig earner), reading from the general `dwh.fact_trip` table. That introduces three structural problems:

1. **Gig churn (`status='ACTIVE'`).** Earners that have since gone inactive are dropped, silently removing their historical trips.
2. **Join-key mismatch.** Matching `fact_trip.driver_uuid` to the gig `earner_uuid` misses AV-native jobs that aren't tied to an active earner row.
3. **It's not the AV-native universe.** `fact_trip` also contains non-AV legs by AV earners and rows that never became real AV jobs.

Our method instead identifies AV trips by **what the trip actually was** (served on an AV vehicle / ran on the AV flow), reading from the AV-native spine. That removes all three problems.

## 5. The evidence

### 5a. Completeness vs an independent authority (`amd.fact_amd_vehicle`)

We re-tested the 2-signal filter against the AV vehicle **registry** (every `vehicle_uuid` there is an AV by construction), over a recent window of **2.38M jobs**:

| signal | result |
|---|---|
| genuine AV-served trips dropped by our filter | **4** (0.00017%) |
| of those, actually completed on the AV | **0** |
| AV-**matched**-but-human-served (redispatch), correctly excluded | ~853,000 |

Adding the registry as a 3rd signal recovered **0** completed trips per month — so the 2-signal filter is complete as-is.

### 5b. The "WeRide Abu Dhabi" question — settled

Per-partner split of the trips our filter drops:

- **WeRide: 20,119 dropped, 100% `matched_only` (a human car served), 0 served by a WeRide AV.**
- Across all partners, only **4** dropped trips were genuinely served by a registered AV car (2 Waymo + 2 Avride), none completed.

→ The dropped WeRide volume is **AV-matched-then-human-redispatch**, not AV-delivered. Correctly excluded.

### 5c. Head-to-head reconciliation — April 2026

| bucket | trips | interpretation |
|---|---|---|
| in both | 419,464 | agree |
| **DS/PM only** | **253** | **all `is_completed=FALSE` AND absent from the AV spine → not real AV trips** |
| **ours only** | **25,603** | **real AV trips DS/PM misses (gig churn + join gaps)** |
| DS/PM total | 419,717 | |
| ours total | 445,067 | **+6.0% vs DS/PM** |

99.94% of the DS/PM pull sits inside ours; their only "extra" 253 rows are non-trips.

### 5d. Polestar correction

Polestar cars are flagged `is_autonomous=true` in `dim_vehicle` but are **not real AVs** (they qualified only via the flag, never via `flow=selfdriving`). Excluded across all method files. Impact on April: 445,067 → **445,062** (5 trips).

### 5e. Partner attribution — `fact_amd_offer` not needed

Tested whether pulling `offer_partner` from `amd.fact_amd_offer` improves labels: **100% of trips (445,062/445,062) resolve on vehicle make**, so `offer_partner` changed 0 labels and rescued 0 `Unknown`. Dropped it — pure scan cost for no benefit.

## 6. Definitions to keep straight

- **AV-delivered ≠ AV-matched.** A trip can be offered to/matched with an AV but completed by a human (redispatch). Those are **not** AV trips and are correctly excluded.
- **Requested vs completed.** Report `av_trips_requested` (passed the filter) and `av_trips_completed` (completed on the AV) separately.
- **`'\N'`** is the warehouse null sentinel; always wrap make/partner/city/flow reads in `NULLIF(..., '\N')`.

## 7. Files

| File | Purpose |
|---|---|
| `av_trips_final_monthly_presto.sql` | **Canonical** "all AV trips for a given month" (trip list + monthly roll-up) |
| `investigate_av_trip_capture_completeness.sql` / `_presto.sql` | Completeness diagnostics (S0–S6) vs the AV vehicle registry |
| `compare_trips_ds_pm_vs_ours_apr2026_presto.sql` | Trip-uuid reconciliation DS/PM vs ours (April 2026) |
| `test_offer_partner_vs_original_presto.sql` | Evidence that `offer_partner` adds nothing |
| `compare_trip_spine_legacy_vs_kb.sql`, `waymo_incident_rate_investigation_spark.sql` | Canonical Spark methods (Polestar exclusion applied) |

## 8. Recommendation

Adopt `av_trips_final_monthly_presto.sql` as the single source of truth for AV trip counts. Re-run **S6** of the completeness diagnostic periodically as a cheap monitor: if a brand-new partner ever ships cars absent from `dim_vehicle` with a non-`selfdriving` flow, it would surface there first, at which point we'd add the AV-vehicle registry as a third signal.

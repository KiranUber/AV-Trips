# How I count AV trips, and why my number doesn't match the old one

_My working notes. Last touched 2026-07-01. Figures are April 2026 unless I say otherwise. Written for Presto/Trino; I kept Spark copies of the main queries too._

## The gist

Count AV trips straight out of `amd.fact_amd_trip`. Every row in that table is already an AV job, so I'm not guessing at anything. A trip makes the cut if either the car that served it is flagged autonomous, or the trip ran on the self-driving flow.

What I nailed down along the way:

- It's not missing anything that matters. I held it up against a separate registry of every AV vehicle we have on record, and it dropped roughly 4 trips out of 2.38 million. None of those 4 actually finished on an AV.
- It's ahead of the old "active driver" method. April alone: 25,603 trips it caught that the old method never saw, about 6% more. The old method did have 253 trips I didn't, but every single one was an unfinished trip that isn't even in the AV table.
- I cleaned up two things: pulled out a few car makes that are mislabeled as autonomous (Polestar, Fisker, Honda), and dropped the offer table for partner names once I was sure it wasn't earning its keep.

The query I actually run is `av_trips_final_monthly_presto.sql`.

## What I was trying to answer

For a given month, am I really catching every AV trip that happened? And why doesn't my count line up with how AV trips have been counted so far?

## Two completely different ways to count

These two approaches don't just differ in the details, they start from opposite ideas of what makes a trip an "AV trip."

The old way counts by the driver. It starts from the general trips table `dwh.fact_trip` and only keeps a trip if whoever was on it is flagged as an active AV gig driver at this moment:

```sql
-- old way: keep a trip only if its driver is an ACTIVE AV gig driver
FROM dwh.fact_trip AS ft
INNER JOIN driver.dim_earner_gig AS gig
        ON ft.driver_uuid = gig.earner_uuid
WHERE gig.gig_type IN ('uber/autonomous/transport/goods',
                       'uber/autonomous/transport/passenger')
  AND gig.status = 'ACTIVE'          -- this line is the whole problem, more below
```

The way I do it counts by the trip. I start from `amd.fact_amd_trip`, which is already nothing but AV jobs, and keep a trip if the car that served it is a known AV or it ran on the self-driving flow:

```sql
-- my way: keep a trip if the trip itself was served by an AV
FROM amd.fact_amd_trip AS t
LEFT JOIN dwh.dim_vehicle AS fv ON fv.vehicle_uuid = t.last_vehicle_uuid
WHERE fv.is_autonomous = true                                             -- served by a known AV
   OR COALESCE(lower(t.flow_type),'') IN ('selfdriving','flow_type_selfdriving')  -- or it ran the AV flow
```

One thing that made the comparison easy: both tables key off the same trip id (`ft.uuid` is the same value as `amd.fact_amd_trip.job_uuid`), so I can line them up one to one and see exactly which trips each side kept.

## What my query is doing, step by step

It's not complicated:

1. Start from the AV-native table.
2. Keep the trip if the serving car is a known AV, or it ran the self-driving flow.
3. Drop Polestar, Fisker, and Honda. They carry the autonomous tag but aren't real AVs (I get into this later).
4. Collapse duplicates to one row per trip, because the table repeats a job every so often.

```sql
FROM amd.fact_amd_trip AS t
LEFT JOIN dwh.dim_vehicle AS fv ON fv.vehicle_uuid = t.last_vehicle_uuid
WHERE ( fv.is_autonomous = true
        OR COALESCE(lower(t.flow_type),'') IN ('selfdriving','flow_type_selfdriving') )
  AND COALESCE(NULLIF(lower(fv.make),'\N'),'') NOT IN ('polestar','fisker','honda')
GROUP BY t.job_uuid
```

I was picky about one thing here. I match `flow_type` against the exact values I know it takes instead of writing `LIKE '%selfdriving'`. A suffix match is sloppy both ways: it would happily keep something like `flow_type_non_selfdriving` (that string does end in "selfdriving"), and it would quietly miss a future `flow_type_selfdriving_v2`. The `COALESCE(..., '')` is just so a missing value reads as "not self-driving" rather than turning into a null that muddies the logic.

I report two numbers, and I keep them apart on purpose:

```sql
COUNT(*)                                                                 AS av_trips_requested,  -- passed the rule
SUM(IF(t.last_matched_av_offer_ts.completed_timestamp_local IS NOT NULL, 1, 0)) AS av_trips_completed  -- finished on the AV
```

- Requested is every trip that passed the rule.
- Completed is the subset that actually finished on the AV.

## Where the old way loses trips

Because it decides "is this an AV trip?" from the driver's current roster status, it springs leaks in three spots:

1. **Drivers drop off the roster.** It only keeps trips for drivers who are active right now. The second a driver goes inactive, their entire AV history disappears from the count. That single `AND gig.status = 'ACTIVE'` line is where most of the loss comes from:

```sql
AND gig.status = 'ACTIVE'   -- active yesterday, inactive today, and all their history is gone
```

2. **The driver-to-trip join is leaky.** Matching `ft.driver_uuid` to `gig.earner_uuid` doesn't always land, so real AV jobs slip through the cracks.
3. **It's the wrong table to start from.** `dwh.fact_trip` also holds non-AV trips by AV drivers, plus trips that never actually became AV jobs, so it manages to both miss real AV trips and let in things that aren't.

Mine avoids all of that by judging the trip itself, not the driver, and by starting from a table where every row is already an AV job.

## The evidence

### Am I quietly dropping any real AV trips?

To check, I grabbed a separate registry of every AV vehicle on record (`amd.fact_amd_vehicle`) and used it to look for trips where a registered AV clearly did the driving but my rule still let it go:

```sql
-- "silent miss": served by a registered AV, but my 2-signal rule dropped it
COUNT(DISTINCT CASE WHEN served_is_registered_av
                     AND NOT COALESCE(kept_by_flag, false)
                     AND NOT kept_by_flow_selfdriving
                    THEN job_uuid END) AS served_av_dropped
```

Across a recent 2.38 million trips:

- Only 4 trips genuinely served by a registered AV got dropped. That's about 2 in a million.
- None of those 4 had finished on an AV anyway.
- A big chunk, around 853,000, was left out on purpose. Those were trips where an AV got offered or matched but a human car ended up doing the ride (a re-dispatch), which isn't an AV trip.

I also tried bolting the registry on as a third signal to see if it rescued anything. It rescued zero completed trips, so the two signals I already have are doing the job.

### The WeRide Abu Dhabi worry

Someone raised the concern that I might be dropping WeRide trips in Abu Dhabi, so I checked it head on. I took every trip my rule drops and split it by partner and by what the AV was actually doing on that trip:

```sql
CASE
    WHEN served_is_registered_av  THEN 'served_by_av (real AV trip we would miss)'
    WHEN matched_is_registered_av THEN 'matched_only (a human car actually served)'
    ELSE 'no_av_car'
END AS av_role
```

- WeRide had 20,119 dropped trips, and every one was a case where the AV got matched but a human car did the ride. Not a single one was actually served by a WeRide AV.
- Lumping all partners together, only 4 dropped trips were truly AV-served (2 Waymo, 2 Avride), and none of them completed.

So that WeRide volume isn't missing AV trips. It's human-served re-dispatches, which I'm right to leave out.

### Head to head, April 2026

I lined the two approaches up trip by trip with a full outer join on the trip id:

```sql
COUNT(*) FILTER (WHERE current.trip_uuid IS NOT NULL AND mine.trip_uuid IS NOT NULL) AS in_both,
COUNT(*) FILTER (WHERE current.trip_uuid IS NOT NULL AND mine.trip_uuid IS NULL)     AS current_only,
COUNT(*) FILTER (WHERE current.trip_uuid IS NULL     AND mine.trip_uuid IS NOT NULL) AS mine_only
FROM current_approach AS current
FULL OUTER JOIN my_approach AS mine ON current.trip_uuid = mine.trip_uuid
```

| | trips | what it means |
|---|---|---|
| In both | 419,464 | the two agree |
| **Only the old way** | **253** | **all unfinished trips that aren't even in the AV table, so not real AV trips** |
| **Only mine** | **25,603** | **real AV trips the old way misses, mostly drivers who went inactive** |
| Old way total | 419,717 | |
| Mine | 445,067 | **about 6% more** |

Almost everything the old way finds (99.94%) is already sitting inside my result. The 253 it had on top didn't hold up: every one had `is_completed = FALSE` and wasn't in `amd.fact_amd_trip` at all.

### The mislabeled makes (Polestar, Fisker, Honda)

Some car makes carry the autonomous tag in the vehicle table but aren't real AVs. They only ever got the tag by mistake, they never ran the self-driving flow (their trips are on the normal p2p/human flow), and none of them ever finished a trip on an AV. So I drop them everywhere:

```sql
AND COALESCE(NULLIF(lower(fv.make),'\N'),'') NOT IN ('polestar','fisker','honda')
```

I didn't just eyeball this. I listed every make the autonomous flag keeps and checked how much of each one's volume actually runs the self-driving flow (`validate_and_vs_or_flow_flag_presto.sql`, Statement 4, last 12 months). The split is about as clean as it gets:

| make | flag-kept trips | self-driving trips | completed on AV | % self-driving |
|---|---|---|---|---|
| Waymo | 3,595,993 | 3,591,836 | 3,382,777 | 99.88% |
| Avride | 89,480 | 89,209 | 81,040 | 99.7% |
| WeRide | 61,234 | 61,229 | 47,878 | 99.99% |
| Motional | 3,867 | 3,867 | 3,063 | 100% |
| Nuro | 2,987 | 2,987 | 1,767 | 100% |
| Baidu | 428 | 428 | 224 | 100% |
| May Mobility | 43 | 43 | 11 | 100% |
| **Polestar** | **353** | **0** | **0** | **0%** |
| **Honda** | **1** | **0** | **0** | **0%** |

Every genuine AV make sits near 100% self-driving and has racked up real completions. Polestar (353 trips) and Honda (1 trip) are the odd ones out: zero self-driving trips, zero completions. That's the giveaway, so both are gone. The cost of dropping them is tiny and never hits a real trip. In April it took out exactly 5 trips (445,067 down to 445,062), all Polestar, none completed.

Fisker is a slightly different case. It didn't show up at all in this 12-month window; it only surfaced earlier as a few never-completed p2p rows back in the wider 2.38-million-trip check. I left it in the exclusion list anyway so the rule doesn't break the next time that make turns up. Worth a second look if Fisker starts showing real volume again.

### Why I stopped using the offer table for partner names

I wanted to know if pulling partner names from the offer table would give me cleaner labels. Turns out no. April looked perfect (445,062 of 445,062 resolved straight from the vehicle make), but one month didn't feel like enough, so I re-ran it over everything from January 2025 to now. Same story: **4,233,061 of 4,233,061 trips got a correct partner name from the vehicle make**. Not one trip ever fell through to the offer-table fallback, which means it was changing no labels and rescuing no "Unknown" partners. So I cut it. The label is just this now:

```sql
COALESCE(NULLIF(fv.make,'\N'),           -- served vehicle make
         NULLIF(vm.make,'\N'),           -- matched vehicle make
         NULLIF(t.partner_name,'\N'),    -- raw partner name
         'Unknown') AS partner
```

## Two distinctions I try not to blur

- **Served by an AV is not the same as matched to an AV.** A trip can be offered to or matched with an AV and still get finished by a human (a re-dispatch). That's not an AV trip.
- **Requested is not completed.** "Requested" means it passed my rule. "Completed" means it actually finished on the AV. I always keep those two numbers separate.

## Where everything lives

| File | What it's for |
|---|---|
| `av_trips_final_monthly_presto.sql` | The query I actually run: all AV trips for a month, plus a monthly summary |
| `investigate_av_trip_capture_completeness.sql` (and `_presto`) | My completeness checks against the AV-vehicle registry |
| `compare_trips_ds_pm_vs_ours_apr2026_presto.sql` | The April trip-by-trip comparison of the two approaches |
| `test_offer_partner_vs_original_presto.sql` | The test that showed the offer table adds nothing to partner names |
| `compare_trip_spine_legacy_vs_kb.sql`, `waymo_incident_rate_investigation_spark.sql` | Spark copies of my method (Polestar, Fisker, Honda already excluded) |
| `validate_and_vs_or_flow_flag_presto.sql` | OR vs AND on the two signals, plus the make check that flagged Polestar and Honda |

## What I'd recommend

Treat `av_trips_final_monthly_presto.sql` as the source of truth for AV trip counts. The one bit of upkeep I'd keep doing is re-running the per-partner split (S6 in the completeness file) now and then. If a brand-new partner ever shows up with cars that aren't tagged autonomous and don't run the self-driving flow, that split is where it'll surface first, and that's the moment to fold the AV-vehicle registry in as a third signal.

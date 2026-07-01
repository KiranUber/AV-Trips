# Identifying AV Trips: What I Found & Validated

_Last updated: 2026-07-01. Comparison month: April 2026. Dialect: Presto/Trino (I also have Spark versions)._

## The short version

- I found that the best place to count AV trips is the AV-native trip table `amd.fact_amd_trip`, where every row is already an AV job. I keep a trip if the car that served it is a known autonomous vehicle, or the trip ran on the self-driving flow.
- I proved this is complete. When I checked it against an independent list of every registered AV vehicle, it missed about 4 trips out of 2.38 million, and none of those 4 had actually completed on an AV.
- I found it to be better than the current approach in every way that matters. For April 2026 my method found 25,603 more real AV trips (+6.0%) than the current approach. The only 253 trips the current approach had that mine didn't were trips that never completed and were not AV trips at all.
- I tested and applied two clean-ups: I remove a small set of cars that are wrongly flagged as autonomous (Polestar, Fisker, Honda) and I stopped using the offer table for partner names (it made no difference).
- My final query lives in `av_trips_final_monthly_presto.sql`.

---

## 1. The question I set out to answer

> For any given month, am I capturing every AV trip that actually happened, and how does that differ from the way AV trips have been counted so far?

## 2. Two ways to count AV trips

I compared two fundamentally different ways to decide whether a trip is an AV trip.

**The current approach (counting by the driver).** It starts from the general trips table `dwh.fact_trip` and keeps a trip only if the driver on it is currently an active AV gig driver. In other words, it asks "was this trip done by someone who is on the AV driver roster right now?"

```sql
-- current approach: keep a trip only if its driver is an ACTIVE AV gig driver
FROM dwh.fact_trip AS ft
INNER JOIN driver.dim_earner_gig AS gig
        ON ft.driver_uuid = gig.earner_uuid
WHERE gig.gig_type IN ('uber/autonomous/transport/goods',
                       'uber/autonomous/transport/passenger')
  AND gig.status = 'ACTIVE'          -- the fragile part: roster status "right now"
```

**My approach (counting by the trip itself).** I start from the AV-native trip table `amd.fact_amd_trip`, where every row is already an AV job, and keep a trip if the vehicle that served it is a known autonomous car, or the trip ran on the self-driving flow. In other words, I ask "was this trip actually served by an AV?"

```sql
-- my approach: keep a trip if the trip itself was served by an AV
FROM amd.fact_amd_trip AS t
LEFT JOIN dwh.dim_vehicle AS fv ON fv.vehicle_uuid = t.last_vehicle_uuid
WHERE fv.is_autonomous = true                                             -- served by a known AV
   OR COALESCE(lower(t.flow_type),'') IN ('selfdriving','flow_type_selfdriving')  -- or ran on the AV flow
```

Both approaches label trips with the same trip ID (`ft.uuid` is the same value as `amd.fact_amd_trip.job_uuid`), so I can line them up one-to-one and compare them exactly.

## 3. What my approach actually does

In plain terms:

1. I start from the AV-native trip table.
2. I keep a trip if the serving car is a known autonomous vehicle, or the trip ran on the self-driving flow.
3. I drop a few makes that are incorrectly tagged as autonomous (Polestar, Fisker, Honda) and are not real AVs.
4. I collapse to one row per trip, because the source table occasionally repeats a trip.

```sql
FROM amd.fact_amd_trip AS t
LEFT JOIN dwh.dim_vehicle AS fv ON fv.vehicle_uuid = t.last_vehicle_uuid
WHERE ( fv.is_autonomous = true
        OR COALESCE(lower(t.flow_type),'') IN ('selfdriving','flow_type_selfdriving') )
  AND COALESCE(NULLIF(lower(fv.make),'\N'),'') NOT IN ('polestar','fisker','honda')   -- drop false-positive makes
GROUP BY t.job_uuid                                            -- one row per trip

I match `flow_type` against an exact list of its known encodings rather than a
`LIKE '%selfdriving'` suffix match. The suffix match is looser and errs both ways: it would
wrongly keep a value like `flow_type_non_selfdriving` (it ends in "selfdriving") and would
silently drop a value like `flow_type_selfdriving_v2` (it does not). The `COALESCE(...,'')`
wrapper keeps a missing `flow_type` evaluating as "not self-driving" instead of unknown.
```

I then report two numbers:

```sql
COUNT(*)                                                                 AS av_trips_requested,  -- passed the rule
SUM(IF(t.last_matched_av_offer_ts.completed_timestamp_local IS NOT NULL, 1, 0)) AS av_trips_completed  -- finished on the AV
```

- **AV trips requested**: every trip that passes the rule above.
- **AV trips completed**: the subset that actually finished on the AV.

## 4. Why the current approach undercounts

Because the current approach decides "is this an AV trip?" based on the driver's current roster status, I found it leaks trips in three ways:

1. **Drivers fall off the roster.** It only keeps trips for drivers marked active right now. The moment a driver becomes inactive, all of their past AV trips silently disappear from the count. This is the `AND gig.status = 'ACTIVE'` line doing the damage:

```sql
AND gig.status = 'ACTIVE'   -- yesterday's active driver, inactive today, takes all their history out
```

2. **The driver-to-trip link is imperfect.** It matches trips to drivers on an ID (`ft.driver_uuid = gig.earner_uuid`) that doesn't always line up, so genuine AV jobs get missed.
3. **It's reading the wrong table.** The general trips table also contains non-AV trips by AV drivers and trips that never actually became AV jobs, so it both misses real AV trips and lets in things that aren't AV trips.

My approach avoids all three because it judges the trip, not the driver, and reads the table where every row is already an AV job.

## 5. The evidence

### 5a. I checked it against an independent list of AV vehicles

I took an independent registry that lists every registered AV vehicle (`amd.fact_amd_vehicle`) and used it to see whether my rule was quietly dropping any real AV trips. The check counts trips where the serving car is a registered AV but my rule did not keep it:

```sql
-- "silent miss": served by a registered AV, but my 2-signal rule dropped it
COUNT(DISTINCT CASE WHEN served_is_registered_av
                     AND NOT COALESCE(kept_by_flag, false)
                     AND NOT kept_by_flow_selfdriving
                    THEN job_uuid END) AS served_av_dropped
```

Over a recent window of 2.38 million trips:

- Only 4 trips that were genuinely served by a registered AV were dropped by my rule (about 2 in a million).
- None of those 4 had actually completed on an AV.
- A large group (about 853,000) was correctly left out. These were trips where an AV was offered or matched but a human car actually did the trip (a re-dispatch). Those are not AV trips.

Adding the AV-vehicle registry as a third rule recovered zero completed trips, so I'm confident the two-signal rule is already complete.

### 5b. The "WeRide Abu Dhabi" question: I answered it

There was a worry that I might be missing WeRide trips in Abu Dhabi. I split every trip my rule drops by partner and by what role the AV played:

```sql
CASE
    WHEN served_is_registered_av  THEN 'served_by_av (real AV trip we would miss)'
    WHEN matched_is_registered_av THEN 'matched_only (a human car actually served)'
    ELSE 'no_av_car'
END AS av_role
```

- For WeRide, 20,119 trips were dropped, and every single one was a case where the AV was matched but a human car actually served the trip. Zero were served by a WeRide AV.
- Across all partners combined, only 4 dropped trips were truly served by an AV (2 Waymo, 2 Avride), and none completed.

So the dropped WeRide volume is not missed AV trips. It is human-served re-dispatches, which I correctly exclude.

### 5c. Head-to-head for April 2026

I lined up the two approaches trip-by-trip. The counts come straight from a full outer join on the trip ID:

```sql
COUNT(*) FILTER (WHERE current.trip_uuid IS NOT NULL AND mine.trip_uuid IS NOT NULL) AS in_both,
COUNT(*) FILTER (WHERE current.trip_uuid IS NOT NULL AND mine.trip_uuid IS NULL)     AS current_only,
COUNT(*) FILTER (WHERE current.trip_uuid IS NULL     AND mine.trip_uuid IS NOT NULL) AS mine_only
FROM current_approach AS current
FULL OUTER JOIN my_approach AS mine ON current.trip_uuid = mine.trip_uuid
```

| | trips | what it means |
|---|---|---|
| In both | 419,464 | the two approaches agree |
| **Only in the current approach** | **253** | **every one was a trip that never completed and wasn't in the AV trip table at all, so not a real AV trip** |
| **Only in my approach** | **25,603** | **real AV trips the current approach misses, mostly because the driver was no longer "active"** |
| Current approach total | 419,717 | |
| My total | 445,067 | **about 6% more trips** |

Nearly everything the current approach finds (99.94%) is already inside my result. The handful it had on top, 253 trips, turned out not to be AV trips. When I looked at those 253, every one had `is_completed = FALSE` and did not exist in `amd.fact_amd_trip` at all.

### 5d. The non-AV make clean-up (Polestar, Fisker, Honda)

A few makes are mistakenly tagged as autonomous in the vehicle table but are not real AVs. They only ever got in through that wrong tag, never through the self-driving flow (their trips run on the normal p2p/human flow), and none of them ever completed on an AV. I now exclude them everywhere:

```sql
AND COALESCE(NULLIF(lower(fv.make),'\N'),'') NOT IN ('polestar','fisker','honda')
```

I identified these by listing every make the autonomous flag keeps and checking how much of each make's volume actually runs the self-driving flow (`validate_and_vs_or_flow_flag_presto.sql`, Statement 4, last 12 months). The separation is razor-sharp:

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

Every real AV make runs the self-driving flow on ~100% of its trips and completes millions of rides. Polestar (353 trips) and Honda (1 trip) run it on **zero** and complete **zero** — the classic false-positive signature, so I exclude both. The impact is tiny and never touches a real AV trip: in April the exclusion removed just 5 trips (445,067 goes to 445,062), all Polestar, none completed.

**Fisker** did not appear in this 12-month window at all; it showed up only as a handful of never-completed `p2p` rows in the wider 2.38-million-trip check. I keep it in the list as harmless future-proofing so the rule stays robust as that make reappears.

### 5e. I don't need the offer table for partner names

I tested whether pulling partner names from the offer table gave better labels. It did not: 100% of trips already get a correct partner name from the vehicle make (445,062 of 445,062). Using the offer table changed no labels and rescued no "Unknown" partners, so I removed it. My partner label is now just:

```sql
COALESCE(NULLIF(fv.make,'\N'),           -- served vehicle make
         NULLIF(vm.make,'\N'),           -- matched vehicle make
         NULLIF(t.partner_name,'\N'),    -- raw partner name
         'Unknown') AS partner
```

## 6. A few definitions I keep straight

- **Served by an AV is not the same as matched to an AV.** A trip can be offered to or matched with an AV but then completed by a human (a re-dispatch). Those are not AV trips.
- **Requested vs completed.** "Requested" means the trip passed my AV rule. "Completed" means it actually finished on the AV. I always report them separately.

## 7. Where everything lives

| File | What it's for |
|---|---|
| `av_trips_final_monthly_presto.sql` | My final query: all AV trips for a given month (trip list plus monthly summary) |
| `investigate_av_trip_capture_completeness.sql` (and `_presto`) | My completeness checks against the AV-vehicle registry |
| `compare_trips_ds_pm_vs_ours_apr2026_presto.sql` | My April 2026 trip-by-trip comparison of the two approaches |
| `test_offer_partner_vs_original_presto.sql` | My test showing the offer table adds nothing to partner names |
| `compare_trip_spine_legacy_vs_kb.sql`, `waymo_incident_rate_investigation_spark.sql` | The Spark versions of my method (Polestar, Fisker, Honda excluded) |
| `validate_and_vs_or_flow_flag_presto.sql` | My test of OR vs AND on the two signals, and the "false-positive make" check that identified Polestar and Fisker |

## 8. My recommendation

Use `av_trips_final_monthly_presto.sql` as the single source of truth for AV trip counts. As a cheap safety check, I'd re-run the per-partner split (S6 in the completeness file) every so often. If a brand-new partner ever shows up with cars that aren't tagged autonomous and don't run on the self-driving flow, that's where it would appear first, and that's the moment to add the AV-vehicle registry as a third signal.

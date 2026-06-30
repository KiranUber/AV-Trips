# Identifying AV Trips — What We Found & Validated

_Last updated: 2026-06-30 • Comparison month: April 2026 • Dialect: Presto/Trino (Spark versions also exist)_

## The short version

- The best place to count AV trips is the **AV-native trip table `amd.fact_amd_trip`**, where every row is already an AV job. We keep a trip if **the car that served it is a known autonomous vehicle, or the trip ran on the self-driving flow**.
- We proved this is **complete**. When we checked it against an independent list of every registered AV vehicle, it missed about **4 trips out of 2.38 million**, and **none of those 4 had actually completed on an AV**.
- It is **better than the current approach** in every way that matters. For April 2026 it found **25,603 more real AV trips (+6.0%)** than the current approach. The only **253** trips the current approach had that ours didn't were **trips that never completed and were not AV trips at all**.
- Two clean-ups were tested and applied: we **remove Polestar cars** (they are wrongly flagged as autonomous) and we **stopped using the offer table for partner names** (it made no difference).
- The final query lives in **`av_trips_final_monthly_presto.sql`**.

---

## 1. The question we set out to answer

> For any given month, are we capturing **every** AV trip that actually happened — and how does that differ from the way AV trips have been counted so far?

## 2. Two ways to count AV trips

There are two fundamentally different ways to decide whether a trip is an AV trip.

**The current approach (counting by the driver).**
It starts from the general trips table (`dwh.fact_trip`) and keeps a trip only if the driver on it is currently an **active** AV gig driver. In other words, it asks *"was this trip done by someone who is on the AV driver roster right now?"*

**Our approach (counting by the trip itself).**
It starts from the AV-native trip table (`amd.fact_amd_trip`), where every row is already an AV job, and keeps a trip if **the vehicle that served it is a known autonomous car, or the trip ran on the self-driving flow**. In other words, it asks *"was this trip actually served by an AV?"*

Both approaches label trips with the same trip ID, so we can line them up one-to-one and compare them exactly.

## 3. What our approach actually does

In plain terms:

1. Start from the AV-native trip table.
2. Keep a trip if **the serving car is a known autonomous vehicle** **or** **the trip ran on the self-driving flow**.
3. **Drop Polestar cars** — they are incorrectly tagged as autonomous and are not real AVs.
4. **Collapse to one row per trip** (the source table occasionally repeats a trip).

We then report two numbers:
- **AV trips requested** — every trip that passes the rule above.
- **AV trips completed** — the subset that actually finished on the AV.

## 4. Why the current approach undercounts

Because the current approach decides "is this an AV trip?" based on **the driver's current roster status**, it leaks trips in three ways:

1. **Drivers fall off the roster.** It only keeps trips for drivers marked *active right now*. As soon as a driver becomes inactive, all of their past AV trips silently disappear from the count.
2. **The driver-to-trip link is imperfect.** It matches trips to drivers on an ID that doesn't always line up, so genuine AV jobs get missed.
3. **It's reading the wrong table.** The general trips table also contains non-AV trips by AV drivers and trips that never actually became AV jobs — so it both misses real AV trips and lets in things that aren't AV trips.

Our approach avoids all three because it judges the **trip**, not the **driver**, and reads the table where every row is already an AV job.

## 5. The evidence

### 5a. We checked it against an independent list of AV vehicles

We took an independent registry that lists **every registered AV vehicle** and used it to see whether our rule was quietly dropping any real AV trips. Over a recent window of **2.38 million trips**:

- Only **4 trips** that were genuinely served by a registered AV were dropped by our rule (about **2 in a million**).
- **None** of those 4 had actually completed on an AV.
- A large group (~**853,000**) was correctly left out: these were trips where an AV was **offered or matched but a human car actually did the trip** (a re-dispatch). Those are not AV trips.

Adding the AV-vehicle registry as a third rule recovered **zero** completed trips — so the two-signal rule is already complete.

### 5b. The "WeRide Abu Dhabi" question — answered

People worried we might be missing WeRide trips in Abu Dhabi. We split every trip our rule drops by partner and by what role the AV played:

- For **WeRide**, **20,119** trips were dropped — and **every single one** was a case where the AV was matched but **a human car actually served the trip**. **Zero** were served by a WeRide AV.
- Across all partners combined, only **4** dropped trips were truly served by an AV (2 Waymo, 2 Avride), and none completed.

So the dropped WeRide volume is not missed AV trips — it's human-served re-dispatches, correctly excluded.

### 5c. Head-to-head for April 2026

We lined up the two approaches trip-by-trip for April 2026:

| | trips | what it means |
|---|---|---|
| In both | 419,464 | the two approaches agree |
| **Only in the current approach** | **253** | **every one was a trip that never completed and wasn't in the AV trip table at all — i.e., not a real AV trip** |
| **Only in our approach** | **25,603** | **real AV trips the current approach misses** (mostly because the driver was no longer "active") |
| Current approach total | 419,717 | |
| Our total | 445,067 | **about 6% more trips** |

Nearly everything the current approach finds (99.94%) is already inside our result. The handful it had on top — 253 trips — turned out not to be AV trips.

### 5d. The Polestar clean-up

Polestar cars are mistakenly tagged as autonomous in the vehicle table, but they are not real AVs (they only ever got in through that wrong tag, never through the self-driving flow). We now exclude them everywhere. For April this removed just **5 trips** (445,067 → 445,062).

### 5e. We don't need the offer table for partner names

We tested whether pulling partner names from the offer table gave better labels. It didn't: **100% of trips already get a correct partner name from the vehicle make** (445,062 of 445,062). Using the offer table changed **no** labels and rescued **no** "Unknown" partners, so we removed it — it was extra cost for no gain.

## 6. A few definitions worth keeping straight

- **Served by an AV is not the same as matched to an AV.** A trip can be offered to or matched with an AV but then completed by a human (a re-dispatch). Those are **not** AV trips.
- **Requested vs completed.** "Requested" means the trip passed our AV rule; "completed" means it actually finished on the AV. Always report them separately.

## 7. Where everything lives

| File | What it's for |
|---|---|
| `av_trips_final_monthly_presto.sql` | **The final query** — all AV trips for a given month (trip list + monthly summary) |
| `investigate_av_trip_capture_completeness.sql` (and `_presto`) | The completeness checks against the AV-vehicle registry |
| `compare_trips_ds_pm_vs_ours_apr2026_presto.sql` | The April 2026 trip-by-trip comparison of the two approaches |
| `test_offer_partner_vs_original_presto.sql` | The test showing the offer table adds nothing to partner names |
| `compare_trip_spine_legacy_vs_kb.sql`, `waymo_incident_rate_investigation_spark.sql` | The Spark versions of our method (Polestar already excluded) |

## 8. Recommendation

Use `av_trips_final_monthly_presto.sql` as the single source of truth for AV trip counts. As a cheap safety check, re-run the per-partner split (S6 in the completeness file) every so often: if a brand-new partner ever shows up with cars that aren't tagged autonomous and don't run on the self-driving flow, that's where it would appear first — and that's the moment to add the AV-vehicle registry as a third signal.

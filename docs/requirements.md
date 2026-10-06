# Strava Clone: Requirements & Capacity Estimate

A practice project for systems design at real scale. The system is designed on
paper for the target scale below and validated at reduced scale on constrained
resources.

## Functional requirements

- Users upload ride files. **`.fit` only for now** (GPX out of scope).
- The system parses each file into:
  - a ride summary (distance, duration, start time)
  - time-series performance data (heart rate, power, and other fields present in the file)
- Users view their ride's stats and analyze its data.
- Users see friends' rides and can comment on them.
- Users get weekly/daily progress dashboards built from their parsed ride data.
- Uploaded files may contain bad or malformed data points; the system must handle them. *(Approach not yet designed.)*

## Assumptions

Each assumption is listed separately so a change can be traced to the numbers that depend on it.

| # | Assumption | Value |
|---|---|---|
| A1 | Users (held flat for the year) | 2,000,000 |
| A2 | Page loads per user per week | 140 (≈ 20/day) |
| A3 | API calls per page load | 5 |
| A4 | Uploads per user per week | 3 |
| A5 | Share of weekly traffic on Saturday | 50% |
| A6 | Share of Saturday traffic in the 8am–12pm window | 75% |
| A7 | Average `.fit` file size | 150 KB (sample of 4 real rides averaged ~138 KB, rounded up for headroom) |
| A8 | Parsed data size per ride | ~300 KB (placeholder assuming columnar JSON; storage model not decided) |
| A9 | Copies of raw `.fit` | 2 (working copy + backup) |
| A10 | Copies of parsed data | 2 (primary + synced replica) |

Race spikes (e.g. ~500 riders uploading within 2–3 hours) are small relative to
the normal Saturday-morning peak and don't drive sizing.

## Derived estimates

### Reads (API calls)

| Quantity | Derivation | Value |
|---|---|---|
| Per week | A1 × A2 × A3 | 1.4B |
| Average rate | 1.4B ÷ 604,800 s | ~2,300/s |
| Peak window total | 1.4B × A5 × A6 | 525M in 4 h |
| **Peak rate** | 525M ÷ 14,400 s | **~36,500/s** |

### Writes (uploads)

| Quantity | Derivation | Value |
|---|---|---|
| Per week | A1 × A4 | 6M |
| Average rate | 6M ÷ 604,800 s | ~10/s |
| Peak window total | 6M × A5 × A6 | 2.25M in 4 h |
| **Peak rate** | 2.25M ÷ 14,400 s | **~156/s** |

**Read:write ratio at peak ≈ 230:1** — a heavily read-dominated system.

### Storage

| Quantity | Derivation | Value |
|---|---|---|
| Per ride, single copy | A7 + A8 | 450 KB |
| Per ride, all copies | A7 × A9 + A8 × A10 | 900 KB |
| Per week | 900 KB × 6M | ~5.4 TB |
| **Per year** | 5.4 TB × 52 | **~280–300 TB** |

## Data protection

- **Raw `.fit` files are the source of truth** → protected by backups. Parsed data can be regenerated from them (parser fixes, new metrics).
- **Parsed data serves the dashboards** → protected by a synced replica for availability. Rebuildable from raw if lost.

## Open questions

- Storage model for parsed data (columnar JSON is only a sizing placeholder).
- Synchronous vs. asynchronous parsing, and the partial-failure story for an upload.
- Bad-data handling strategy.
- How dashboard aggregates are computed (on read vs. precomputed).
- Read path at ~36.5k/s and how many replicas read load requires.
- Where backups live (separate failure domain).
- Tech stack.
- How to validate at reduced scale (load level, resource limits).

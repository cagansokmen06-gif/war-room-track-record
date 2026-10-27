# Track record: preregistration

**Status: DRAFT.** Not approved yet. Genesis (the first record, which holds this file's SHA-256) is written only
after Talentino and Cagan approve this text. After genesis this file never changes: an edit makes `verify` fail.
Later changes are amendments (see "Changes after genesis").
**Owner:** Talentino (asked for). Reviewer: Cagan.

## Purpose
A forward out-of-sample test of the Setup Score and of the COT extremes it is built on. Issue #95's edge
decomposition found no out-of-sample edge for the setup in the historical data; this record tests it on weeks
that had not happened when the rules below were fixed. It is not marketing: the result is published whatever it
is, including a null or a negative one. Nothing is filled in backwards: weeks before the first snapshot are not in
the record and are never added.

## Unit
One snapshot per COT report: every market in `tracker.MARKETS`, with the values `tracker.py` wrote to
`dashboard_data.json` in the first run that showed the report (26w and 52w commercial index, 26w spec index, weeks
at the extreme, score, watch level, bias, flags, alignment sides). Snapshots are written at most 7 days after the
report's release (`cot_schedule.release_time`) and only for a report newer than the last snapshot.

**The score is measured as first recorded.** During the week the score, watch level and alignment move with prices
and the seasonal day; those moves are not recorded. Only the COT fields (`cot_date`, `comm_idx_26`, `comm_idx_52`,
`spec_idx_26`, `weeks_at_extreme`) are compared in later runs; a difference is a revision record (below).

A market whose `cot_date` is not the snapshot's `report_date` (its report came late) is left out of that week.

**Publishing:** every record is pushed to the public repo within 7 days of the release it belongs to. A later push
gets a note saying why; the record itself stays as written.

## Events
Two event types, each with a direction:

1. **26w 90/10 entry.** `comm_idx_26` >= 90 (BULLISH) or <= 10 (BEARISH) (`cot.UPPER` / `cot.LOWER`, direction
   rule `commercials_follow` from `backtest_config.json`), after at least `MIN_GAP_WEEKS` (13, `outcomes.py`)
   consecutive reports outside that side. For the reports before the first snapshot, the gap is read from the first
   snapshot's `gap_history`: the 26w index of the 13 reports before it, computed the way the tracker computes it
   (`cot.compute` on the tracker's `cot.db` rows: the `tracker.LEGACY_CODES` contracts, SPX as S&P 500 Consolidated,
   `KNOWN_BREAKS` through `cot.index_nets`, `cot.cot_index`), never with a second series or function; from the first
   snapshot on, the gap is read from the recorded values.
2. **WATCH level 3 entry.** `watch_level` becomes 3 (grade A, score >= 70) with bias BULLISH or BEARISH, after at
   least 13 consecutive recorded snapshots below level 3 for that market. The level depends on prices at the time,
   so it cannot be rebuilt for weeks that were not recorded: a missing week restarts the 13-week count. The first
   possible WATCH events are therefore 13 weeks after the first snapshot.

The gap makes one market's events independent: 13 weeks is the longest hold, so two events' forward windows never
overlap.

**Missing weeks (both event types).** A week missing from the record for a market (the tracker was off, a shutdown
report refused as backfill, or the market's `cot_date` came late that week) restarts that market's gap count. An
entry that falls in a missing week is not an event, and the next recorded week is not an entry if the market is
still at the extreme then. `verify` lists the missing weeks and the late markets.

## Primary hypothesis
- **Primary test:** the 26w 90/10 entry with the 8-week hold (`PRIMARY_HOLD`). Measure: the mean signed return of
  the events minus the benchmark return (same market, same weeks; see "Benchmark"). Ranked second for the same
  test: the hit rate minus the base rate.
- **Everything else is secondary or descriptive:** the WATCH level 3 entry, and the 4- and 13-week holds of both
  event types. With 2 event types x 3 holds x 2 measures there are 12 comparisons, one of them primary. Every report
  gives that total, and secondary results carry the note "no correction for multiple tests".

## Decision rule
- Every quarterly report is labelled **"interim, descriptive"**. None of them says whether there is an edge.
- There is **one confirmatory evaluation**: the December 2028 quarterly report, with the events whose hold has ended
  by then.
- The rule: if the lower end of the 90% bootstrap interval (see "Dependence") of the primary measure (mean signed
  return minus benchmark; 26w 90/10 entry; 8-week hold) is above 0, the verdict is **"edge supported"**; otherwise
  **"edge not supported"**. The hit rate minus the base rate is reported next to it as secondary and does not change
  the verdict.
- The confirmatory evaluation uses every `config_hash` segment together as long as `primary_hash` (see "Code and
  market changes") has not changed.
- The record goes on after that date, but a new verdict needs a new preregistration or an amendment.

## Dependence
- Events in the same release week (any markets) form one cluster. Clusters are for counting: every report gives
  both the number of events and the number of clusters (weeks), and the `MIN_EVENTS_ANY` (10) threshold applies to
  the number of clusters.
- Confidence intervals: a moving block bootstrap over consecutive release weeks, with the block length equal to the
  hold in weeks (8 for the primary test; 4 and 13 for the other holds): blocks of consecutive recorded weeks are
  drawn with replacement until the record's length is reached, and the measure is computed on the events of the
  drawn weeks. `BOOTSTRAP_N` (2000) draws with `SEED` (7) from `backtest_config.json`; the 90% percentile interval.
  Why blocks and not single weeks: events in different weeks have overlapping holds and measure the same macro move.

## Entry, holds and outcome (stage 2)
- **Entry:** the close of the first trading day after the release's calendar date in New York
  (`cot_schedule.release_time`). For a normal Friday release this is Monday's close, the same as `backtest.py` and
  `outcomes.py` (`ENTRY_LAG_TRADING_DAYS` 1). **Deviation from the backtest:** when a federal holiday moves the
  release (`cot_schedule.DELAYED_RELEASES`), the entry follows the real release; `backtest.release_date` always
  uses the Friday, which would enter before the data was public.
- **Holds:** `HOLDS_WEEKS` (4, 8, 13); the primary hold is `PRIMARY_HOLD` (8). The exit is the last close at or
  before entry + h weeks.
- **Outcome:** the return signed by the event's direction (BULLISH: price up is a hit; BEARISH: price down is a
  hit). Outcomes are written by stage 2 as their own records; this file does not compute them.
- **Price source:** still open. It must be a source whose terms allow publishing in a commercial product (issue
  #87); Yahoo is not one. The source and, for every market, the instrument (spot or futures, the symbol, the roll
  rule if there is one) are fixed in a note record before the first outcome record and never change after it; a
  change is an amendment.

## Benchmark
The same market over the same weeks: for each event, the market's h-week return over all recorded snapshots
(same hold), signed by the event's direction. The benchmark return is that mean; the base rate is the share of
those weeks that went the event's way (up for BULLISH, down for BEARISH). The historical base rate (2006 on, as in
`outcomes.py`) is shown next to it for context, not as the benchmark.

## Reporting
- **Calendar:** one report per quarter, published in the week after the last Friday of March, June, September
  and December, whatever the result. It covers every snapshot up to that Friday; events whose hold has not ended
  are counted as open. No interim result is published outside this calendar.
- Pooled over all markets for each event type and hold; per group only as a description.
- With fewer than `MIN_EVENTS_ANY` (10) clusters with a finished hold, no hit rate or mean is shown: only the
  counts and the events.
- Every report gives n (events and clusters), the hit rate, the mean signed return and the benchmark, the number of
  events still open, and the total number of comparisons.
- No event is dropped because of its outcome. The only exclusions are fixed here:
  - a market with a late `cot_date` that week;
  - a shutdown-delayed week: `shutdown_backlog` is true when `release_date` − `report_date` > 10 days. It is
    written in the snapshot when the snapshot is recorded and never changed (`verify` checks it against the two
    dates). `release_date` is `cot_schedule.release_time`'s; a report released later than that schedule says is
    first seen more than 7 days after its scheduled release and is refused as backfill (see "Unit"), so it is not in
    the record either. `backtest_config.json`'s `DELAYED_RELEASES` list is only the reference for the weeks before
    the record (`gap_history`).
- Snapshots written with uncommitted code changes (`code.dirty`) are reported both with and without them.

## Corrections and revisions
- A record is never edited or removed. A wrong or questionable record gets a `note` record saying why.
- A COT revision (CFTC or a data fix) is a `revision` record that names the snapshot's hash and lists the changed
  fields. Events are classified on the snapshot as first recorded; a second line repeats the result with the
  revised values. Only the newest report can be revised (the tracker keeps only the newest week).
- A missed week (the tracker was off) stays missing. It gets a note; it is not filled in later.

## Code and market changes
- Every snapshot carries the git commit, whether tracked files differed from it (`code.dirty`), and `config_hash`
  (`setup.py`'s constants, `cot`'s thresholds and windows, `outcomes.MIN_GAP_WEEKS`, `backtest_config.json`, and the
  market list: `tracker.MARKETS` with each market's CFTC code from `tracker.LEGACY_CODES`).
- Every snapshot also carries `primary_hash`, which covers only what defines the primary event: `cot.UPPER` /
  `cot.LOWER`, the 26w window, `outcomes.MIN_GAP_WEEKS`, `backtest_config.json`'s direction rule
  (`commercials_follow`), `PRIMARY_HOLD` and `HOLDS_WEEKS`. The market list is not in it.
- Every market in a snapshot carries its own `market_def_hash`: its CFTC code (`tracker.LEGACY_CODES`), its CFTC
  names (`tracker.MARKETS`) and its `KNOWN_BREAKS` entries.
- A change of `config_hash` starts a new segment, announced in a note before or in the week of the change. Results
  are reported per segment and pooled; nothing is recomputed under the new settings for weeks recorded under the
  old ones. The primary test is split only by a change of `primary_hash`: a change to `setup.py` alone splits the
  WATCH test into segments, never the primary test, whose confirmatory evaluation uses every `config_hash`
  segment with the same `primary_hash` together.
- Markets are handled one at a time. A market added later, or a market whose `market_def_hash` changes (e.g. a new
  CFTC code), behaves as a new market: its own segment from its first snapshot under that hash, joining the pool
  only once it has 13 weeks of recorded history under it (no `gap_history` is taken for it). Adding or removing a
  market does not affect the other markets, and a removed market's events up to its removal stay in the pool.

## Changes after genesis
This file is frozen. An amendment is a new file (`AMENDMENT-<n>.md`) in the public repo, announced by a note record
that gives its SHA-256. It applies only to snapshots recorded after that note, never to earlier ones.

## Verification
Anyone can check the chain with `python track_record.py verify --path <clone>` or with the stand-alone steps in
`docs/track-record.md`: every line is canonical JSON, every hash is the SHA-256 of the line without its hash, every
line names the previous hash, and genesis names this file's SHA-256.

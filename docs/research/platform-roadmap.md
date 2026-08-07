# Power Market Research Platform: roadmap

Internal research note. Not part of the published PyPSA-USA
documentation (`docs/source/`) and not wired into the Sphinx build.

**Answers:** *in what order do we build it, and what kills it?*

**Status:** nothing built.
**Last updated:** 2026-08-07
**Companion notes:**
[platform-architecture.md](platform-architecture.md) (structure),
[engine-capabilities.md](engine-capabilities.md) (what the engine
provides), [../known-issues.md](../known-issues.md) (upstream defects)

## How to read this

Every phase ends in a **gate**: a specific, falsifiable check that
decides whether the next phase happens. Phases are not a checklist to
complete. The scope described in the architecture note is roughly two
person-years if built exhaustively; it is tractable in 6–12 months
only if gates are allowed to kill work.

Three rules apply throughout:

1. **Build only what the residual points at.** No structural
   enhancement gets built speculatively. A pattern in the error must
   justify it first.
2. **The dashboard is last.** A polished UI over an unvalidated model
   is the single most common failure mode of projects like this, and
   it is detectable on sight by anyone who evaluates it.
3. **Test the cheapest falsifying question first.** Phases are ordered
   by cost-to-falsify, not by build dependency. If a days-scale test
   can invalidate a year of work, it runs before that work — this is
   why Phase -1 exists and comes before everything.

## Gate discipline

The gates below are only as good as their statistical construction,
and as originally written they were not decidable. Before any gate is
evaluated, four things are fixed **in advance and in writing**:

- **The metric**, and the held-out window, chosen before any model is
  fitted. Held out by *time*, never a random split.
- **A minimum effect size** that would justify continuing. "Improved"
  is not a gate; "improved by more than X" is. Phase -1 supplies X —
  the hurdle is not "beats the baseline" but "beats the market by more
  than costs."
- **A retry budget.** A fixed, small number of looks, decided up
  front.
- **A final holdout looked at exactly once, ever.** Not the same data
  used for iteration.

The reason is that Phase 3 as designed ("re-run the comparison after
each change; revert if it didn't move the error") is iterative
selection against the same held-out period, which converts that
holdout into a training set after enough iterations. With a large
panel and several candidate features, something will eventually look
like improvement. Without pre-registration the "hard kill" gates are
soft, and a project that cannot kill itself is the failure mode this
whole document is arranged to prevent.

## Phase -1 — Auction efficiency pre-gate

**Do this before anything else, including the bug fixes.** It is the
cheapest test that can invalidate the entire plan: roughly one dataset
and a regression.

The rest of this roadmap asks whether physical features beat a *naive
statistical baseline*. For a trading application that is the wrong
bar. CRR auction clearing prices already embed the collective basis
forecast of everyone bidding — including desks with nodal models,
outage intelligence, and unit-level offer data we do not have. It is
entirely possible to pass Phases 2 and 3 cleanly, genuinely adding
skill over a naive baseline, and still have zero tradeable edge
because the auction already priced it.

- Pull ERCOT CRR monthly auction clearing prices and realised
  node-hub basis for the same periods.
- Measure the auction's *own* forecast error: how well does the
  auction clearing price predict realised basis?
- Characterise where it is worst — which nodes, seasons, congestion
  regimes. Segment by whether the node is one a coarse physical model
  could ever resolve.
- Build a first-cut **frictions model**: auction fees, bid-ask,
  capital requirements. This is what converts a statistical result
  into a minimum effect size, and no later gate can be stated without
  it.

**Gate:** the auction leaves an identifiable, economically meaningful
residual at a spatial resolution a reduced physical model could
plausibly resolve, and that residual survives transaction costs. If
auction prices are already near-efficient in the segments a ~75–400
bus model can address, **stop** — the physical-modelling chain is moot
regardless of how well Phase 2 would have gone. This is a better
outcome to discover in a week than in a year.

## Phase 0a — Fix what the model gets wrong

Before the dispatch output is used to decide anything, the known
defects in it get resolved. Phase 2 is the gate everything downstream
depends on, and it is currently specified to run on top of unverified
numbers.

- Work [../known-issues.md](../known-issues.md), starting with the
  suspected MWh/kWh unit error and the coal fuel-price conversion. A
  factor-of-1000 error can move the Phase 2 result in *either*
  direction — a false pass from spurious correlation, or a false kill
  from features that would have worked if costs were right.
- Set `consider_efficiency_classes: true` (Tier 0, config only) to
  stop collapsing every regional gas unit into one blended heat rate.
  See [engine-capabilities.md](engine-capabilities.md).
- Add the invariant assertions described in the architecture note's
  model architecture section to whatever solve driver is used, so this
  class of defect fails loudly rather than propagating into a
  residual.

**Gate:** a solved network passes energy-balance, cost-reconciliation
and plausible-price-range assertions, and the two open unit bugs are
either fixed or positively ruled out.

## Sequencing decisions already made

- **ERCOT first.** Energy-only market, no capacity market, simplest
  settlement structure. CAISO or PJM second, and only after ERCOT's
  loop is closed end to end.
  - Caveat worth budgeting for: PJM's data ergonomics are
    substantially better (Data Miner 2 is a documented REST API with a
    published pnode table), while ERCOT MIS is report scraping. ERCOT
    is still the right first market — just expect the data pipeline to
    cost more than it looks like it should.
- **Separate repository, editable fork.** `power-market-platform`, not
  a branch of the PyPSA-USA fork. Only `network_builder/` depends on
  PyPSA-USA. The fork *is* edited where needed, under the four-tier
  rule in the architecture note: additive changes only, anything that
  rewrites existing behaviour graduates into the platform repo.
- **Prototype outside Snakemake.** Snakemake's value is reproducibly
  rebuilding the dispatch model across config combinations — that is
  build-system overhead for a research question. Load solved `.nc`
  networks directly in scripts. Formalise into rules only once a
  methodology is proven and needs to re-run on a schedule.
- **Day-ahead first; DA/RT spread deferred, likely indefinitely.**
  DART is driven mostly by forecast error that a deterministic
  dispatch model has none of by construction.

## Phase 0 — Extraction and reduction

**Goal:** a standing, fast, reusable ERCOT market network, built by a
reproducible one-command process.

- Run the PyPSA-USA workflow for ERCOT as far as `simplify_network`,
  producing `elec_s.nc` — the seam artifact. Config-only; no fork
  edits expected here.
- In `network_builder/`, compute a busmap targeting ~75 buses. First
  pass may simply be PyPSA-USA's own clustering; the congestion-aware
  version is a Phase 3 refinement, not a prerequisite.
- Feed it back through `cluster_network` via `enable.custom_busmap`.
  Take `busmap.csv` and `linemap.csv` as the source-bus provenance for
  `model_buses.parquet` (source bus counts, region labels, centroids,
  load share).
- Write the build manifest: upstream commit SHA, config hash, busmap
  hash, timestamp.
- Write the dispatch loop in `dispatch/` — plain `n.optimize()`, fixed
  capacity, no extendables, no policy constraints. Read
  `solve_network_validation` in `rules/validate.smk` as a reference
  implementation first, then write your own rather than importing.
- Verify an 8760-hour hourly linear dispatch solves in tolerable wall
  time.

**Gate:** *the calibration loop*, not a single solve, runs fast enough
to permit iteration, and a rebuild from the manifest reproduces the
same network. The whole plan depends on cheap, trusted re-runs.

Benchmark the right thing: the real workload is ~730 daily solves
repeated once per calibration iteration, not one 8760-hour run. Model
construction rather than simplex may dominate, which is why the
architecture note specifies building the LP once and warm-starting.
Measure the loop; a fast single solve inside a slow loop is not a
pass.

## Phase 1 — Commercial layer and crosswalk

**Goal:** the market-node layer exists independently and is correct.

- Build `market_nodes.parquet` from ERCOT's published Settlement
  Points List: resource nodes, hubs, load zones, with coordinates
  where obtainable.
- Pull 2+ years of ERCOT day-ahead settlement point prices. This is
  the prediction target and does not exist in PyPSA-USA in any form.
- Build `node_model_crosswalk.parquet` at **Level 1 (regional) only**.
  Populate `mapping_type` and `confidence` honestly.
- Compute observed basis series: `LMP[node] - LMP[hub]`.
- Pull ERCOT generation and transmission outage reports for the same
  period. Promoted here from a later phase deliberately: PyPSA-USA
  models no forced or maintenance outages at all — the only derates
  present are EIA-860 seasonal capacity ratings, which are an ambient
  ratings adjustment, not availability. Reserve margin computed
  without outage data is a margin against nameplate capacity and will
  systematically miss the tight hours that drive basis. Feeding real
  outages in as a time-varying `p_max_pu` is cheap and does not
  require modelling outages endogenously.

**Gate:** basis series are constructed, sane, and stationary enough to
model. Characterise them before modelling anything — distribution,
autocorrelation, seasonality, share of nodes with meaningful basis at
all. Many nodes will have near-zero basis nearly always; those are not
worth modelling and should be dropped now, not later.

## Phase 2 — The experiment that decides the project

**This is the phase that matters. Everything downstream is contingent
on it.**

The question:

> Do PyPSA-derived physical features add predictive skill for node-hub
> basis, *beyond* a baseline using only lagged basis, load forecast,
> and wind forecast?

Procedure:

1. Fit the naive baseline first. Lagged basis, calendar features,
   ERCOT load and wind forecasts. Record its out-of-sample error.
2. Run the reduced dispatch over the same period. Extract physical
   features: interface flows, corridor shadow prices, regional net
   load, marginal carrier, reserve margin, curtailment, dispatch by
   fuel.
3. Refit with physical features added. Compare out-of-sample, on a
   held-out period, not a random split.
4. Attribute: which features carry the incremental skill, and does
   the attribution make physical sense?

**Gate — hard kill.** If physical features do not improve on the
baseline out of sample *by more than the Phase -1 minimum effect
size*, the physical-modelling thesis is decorative. Stop and report
the negative result honestly — a well-documented negative result here
is a respectable outcome and a far better portfolio artifact than a
dashboard built on a thesis that didn't hold.

Two conditions on evaluating this gate, both load-bearing:

- **Do not fire the kill at ~75 buses.** With a Level-1 regional
  crosswalk, every settlement point in a region receives identical
  physical features, so a failure cannot distinguish "physics does not
  explain basis" from "we compressed the physics into twelve numbers."
  Raising resolution to ~250–400 buses is therefore a *precondition of
  the kill*, not a retry after it. Close the loop at 75 for speed;
  evaluate the gate at the finest resolution that is computationally
  tolerable.
- **Features must be information-set aligned.** Every feature satisfies
  `knowledge_time <= decision_time` per the architecture note's
  bitemporality rule. ERA5 reanalysis is realised weather, and the DA
  market cleared on forecasts — using it unguarded gives the model
  information the market did not have, inflating apparent skill
  exactly in the tight hours that matter most. A gate passed on leaked
  features is worse than a failed gate, because it licenses a year of
  downstream work.

## Phase 3 — Empirical exposures and targeted enhancement

Only if Phase 2 passed.

- Estimate the Level-3 exposure matrix `beta[i, k]` per node against
  physical factors. This becomes the platform's distinctive output:
  each node's sensitivity to specific, named physical drivers.
- Address only the structural gaps the Phase 2 residual actually
  points at. Likely candidates, in probable order of impact:
  - congestion-aware clustering — replace the Phase 0 busmap with one
    that groups buses to preserve known ERCOT interfaces rather than
    by geography or load. Still Tier 1: a different busmap, not a
    different clusterer.
  - raising bus count for finer congestion resolution
  - an ORDC-style scarcity adder as post-processing on the shadow
    price, before any full multi-product reserve co-optimisation.
    ERCOT prices RRS, ECRS, non-spin and regulation as separate
    co-optimised products; PyPSA-USA has only a single aggregate
    spinning-reserve constraint, off by default, and its `erm`
    constraint is a planning reserve margin rather than an operating
    product. Treat the simple adder as the first cut.
  - regional gas basis (Waha for ERCOT) rather than a flat fuel price
- Re-run the Phase 2 comparison after each change. If it didn't move
  the error, revert it and move on.

**Gate:** at least one structural enhancement measurably improved
out-of-sample error, and the exposure matrix is physically
interpretable.

## Phase 4 — Instrument-level strategy research

Only if Phase 3 passed.

A predicted basis with good R² is not a strategy, and this is the
first thing a desk will probe. Pick a concrete instrument before
writing any backtest code:

- ERCOT CRR monthly auctions (auction clearing price is the entry
  cost; the basis realisation is the payoff), or
- DA/RT virtuals, if and only if the DART work deferred above is
  revisited.

The backtest must model auction clearing, transaction costs, and
capital constraints — extending the frictions model built in Phase -1
rather than starting one here. Report drawdown and hit rate alongside
returns, and attribute P&L to signals.

Note that Phase -1 already established whether an exploitable residual
exists at all. This phase asks the narrower question of whether *our*
signal captures it after costs.

**Gate:** a backtest whose assumptions are explicit and whose result
survives being stated plainly, including if the result is that the
edge doesn't clear costs.

## Phase 5 — Dashboard

Only now.

Read-only over the parquet artifacts, never solving anything. Pages,
in build order:

1. **Market overview** — load, wind, solar, reserve margin, gas price,
   scarcity indicator.
2. **Network** — the two visually distinct spatial views described in
   the architecture note.
3. **Dispatch** — generation by fuel, marginal generators,
   curtailment.
4. **Congestion** — binding constraints, congestion frequency,
   simulated vs. historical.
5. **Nodal analytics** — node-hub basis, exposure matrix, widening and
   narrowing spreads.
6. **Strategy** — signals, backtest, attribution.

Pages 1–3 are cheap and can be built earlier as debugging tools for
your own use. Pages 4–6 are the actual product and should not exist
until the phases behind them passed their gates.

## Phase 6 — Data platform refactor

**Scope:** multi-ISO ingestion, ancillary services, unified store.

Not sequential with Phases 1–5 in the usual sense — its trigger is an
event, not "the previous phase finished." **Do not start this phase
speculatively.** It exists to name what a *second* ISO and a
*residual-justified* ancillary-service model actually require, so
that when the trigger fires, the work happens as one deliberate
design pass instead of the ad hoc script sprawl that would otherwise
accumulate one ISO/data-source at a time.

**Trigger (both required):**

1. Phase 2's gate has passed for ERCOT — the physical-features thesis
   is validated, so a second ISO or a heavier reserve model is buying
   something real, not decorating an unproven core.
2. A concrete forcing function exists: either CAISO/PJM onboarding
   begins (per the "CAISO or PJM second" sequencing decision above),
   or the Phase 2/3 calibration residual specifically implicates
   missing ancillary-service co-optimization (not just "it would be
   nice to have," per the ORDC-adder-first discipline in
   [engine-capabilities.md](engine-capabilities.md)).

Until both hold, keep doing what Phase 0–1 already do: one-off,
ISO-scoped fetch scripts, no shared abstraction. A schema designed
against one ISO's data is a guess; a schema designed against two is a
generalisation with evidence behind it.

**Goal:** a single, well-designed store for everything that today is
scattered across ad hoc `retrieve_*.py` scripts, a pinned PUDL
snapshot, Zenodo bundles of inconsistent vintage, and config-file year
pins — historical ISO settlement prices, fleet/generator data, fuel
prices, outages, and weather — so that adding ISO #3 is a config entry
against an existing adapter interface, not a new script.

- **Ingestion layer:** one adapter per (source, not per ISO-and-source
  combination where avoidable) — e.g. a `LMPSource` interface
  implemented by `ERCOTMisSource`, `CAISOOasisSource`,
  `PJMDataMiner2Source`; a `FleetSource` wrapping PUDL with an
  explicit refresh cadence instead of a hand-bumped path string; an
  `OutageSource` for NERC GADS / EIA-860M; a `FuelPriceSource` per
  region/hub. Each adapter's job is narrow: fetch, validate against a
  declared schema, write to the raw layer. No cross-ISO logic here —
  that was the mistake `retrieve_caiso_data.py`-style one-offs made
  by accreting ISO-specific assumptions into shared code paths.
- **Storage design — evaluate before committing:**
  - *Partitioned parquet lake* (`raw/source=ercot_mis/year=2024/…`,
    mirroring the existing `artifacts/` parquet convention in
    [platform-architecture.md](platform-architecture.md)) — cheapest,
    consistent with the rest of the platform, but weak for
    point-in-time joins across sources (e.g. "LMP at hour H joined to
    outage state at hour H").
  - *Embedded analytical DB* (DuckDB over the same parquet, or a
    file-based table store) — SQL joins across sources without
    standing up infrastructure; still just files, still git/backup
    friendly.
  - *Managed time-series/relational DB* (TimescaleDB/Postgres) —
    justified only if query patterns actually need it (e.g. a live
    dashboard hitting it directly, or write-heavy frequent-refresh
    operational data like outages). Don't reach for this by default;
    it's infrastructure the parquet-lake decision above should
    disqualify first.
  - Decide with real query patterns from Phase 1–3's actual usage in
    hand, not upfront speculation — the point of gating this phase is
    to have that evidence before picking.
- **Schema contract, not just storage:** every source adapter declares
  its output schema (columns, units, timezone, granularity) up front
  and the ingestion layer validates against it on write. This is
  where the coal-price and MWh/kWh-style unit bugs tracked in
  [../known-issues.md](../known-issues.md) stop being possible to
  introduce silently — a declared unit is checked, not assumed.
- **Refresh cadence as a first-class property**, not a manually
  re-run script: each source declares how often it should be
  re-pulled (PUDL snapshot: per-release; ISO LMPs: daily/monthly
  backfill; outages: daily; fuel prices: daily/weekly). Fixes the
  "fleet data is stale until someone remembers to bump `pudl_path`"
  problem structurally instead of via a checklist item.
- **Ancillary services, built properly this time:** once triggered by
  a real residual (not spec'd speculatively), replace the ORDC-adder
  post-processing hack from Phase 3 with actual multi-product
  co-optimization — separate cleared products and prices for
  regulation, spinning/non-spin reserve, and (ERCOT-specifically)
  ECRS/RRS, each with its own supply curve and clearing constraint in
  the LP. This is a real model-structure change in the PyPSA-USA fork
  layer (Tier 2+ under the four-tier rule), not platform-repo-only
  work, and should be scoped as its own design pass once triggered —
  not bolted on inside this same refactor.

**Gate:** a second ISO's full pipeline (ingestion → market nodes →
crosswalk → calibration) runs through the shared adapters with no
ISO-specific code outside the adapter layer, and — if the
ancillary-service trigger fired — the multi-product reserve model
measurably tightens the calibration residual versus the Phase 3 ORDC
adder on a held-out period.

## Deferred indefinitely

Revisit only if a specific finding demands it, not as a default next
step: DA/RT settlement split, sub-hourly resolution, full
market-clearing mechanics (bid stacks, uplift, make-whole payments),
integer unit commitment, and forward-looking weather forecast
ingestion (GFS/ECMWF rather than ERA5 reanalysis). Multi-ISO
generalisation before the first ISO is finished is deferred the same
way, but see Phase 6 above for what it looks like once its trigger
fires — "deferred" here means gated, not ruled out.

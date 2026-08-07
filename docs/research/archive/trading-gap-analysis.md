# Gap Analysis: PyPSA-USA as a Power Trading Engine (ARCHIVED)

Internal research note. Not part of the published PyPSA-USA
documentation (`docs/source/`) and not wired into the Sphinx build.

**Scope reviewed:** `workflow/`, `docs/source/`, `workflow/repo_data/`
on `master`.
**Written:** 2026-08-02. **Archived:** 2026-08-07.

> **Status: fully archived. Do not use as a reference.** Retained only
> for provenance — it is the original analysis the current design grew
> out of, and nothing links to it for current guidance.
>
> Its content has been split into two live documents:
>
> - Capability inventory (what the engine does and does not provide)
>   moved to
>   [../engine-capabilities.md](../engine-capabilities.md), updated
>   with findings this note predates.
> - Project framing, sequencing and phasing moved to
>   [../platform-architecture.md](../platform-architecture.md) and
>   [../platform-roadmap.md](../platform-roadmap.md).
>
> Two specific reasons not to read this for current guidance: it is
> loose about the distinction between PyPSA buses and ISO pricing
> nodes (the architecture note now treats this as a hard separation),
> and its calibration framing measures success against a naive
> statistical baseline rather than against market-implied forecasts
> (see the roadmap's auction efficiency pre-gate).

## Bottom line

PyPSA-USA is a good foundation for a fundamentals/scenario engine,
not a drop-in trading engine. It already produces nodal marginal
prices (`n.buses_t.marginal_price`) from a DC-OPF with real line
impedances, optional linearized unit commitment, and a
reserve-margin constraint. That's the right physics core.

The revised strategy (below) is deliberately narrower than a full
market-microstructure rebuild: use the dispatch model as a
**fundamentals/feature engine**, and put a statistical calibration
layer on top to bridge the gap to actual settlement prices. This is
the standard pattern used by production-cost tools (PROMOD, PLEXOS,
Aurora) in real trading/forecasting shops - nobody trades directly
off raw physical-optimization output.

## What already works in your favor

- **Nodal marginal prices — have.** True dual values from the
  LP/MILP solve, congestion-inclusive, exposed as
  `n.buses_t.marginal_price` and plotted (`plot_network_maps.py`,
  `plot_statistics.py`).
- **Configurable nodal resolution — have.** Node count is a
  wildcard, not fixed - up to ~41k nodes nationally via the TAMU
  synthetic network; ReEDS-zone topology also available.
- **Linearized DC-OPF with real impedances — have.** Lines carry
  r/x/b/g from voltage-class line types, not a transport model.
  Optional piecewise-linear loss approximation.
- **Unit commitment scaffolding — partial.** Min up/down time,
  start-up cost, ramp rates modeled and sourced from WECC ADS. Off
  by default; solved as relaxed (linearized) MILP, not full integer
  commitment.
- **Operating reserve margin — partial.** Generic load/VRE-based
  spinning-reserve constraint exists (`opts/reserves.py`). Not
  ISO-product-accurate (no regulation, non-spin, frequency response
  markets).
- **Dynamic fuel cost hooks — partial.** PUDL-sourced per-plant
  fuel costs and a wholesale-price option exist
  (`add_electricity.py:apply_pudl_fuel_costs`), off by default.
- **Production/demand validation — partial.** `validate.smk`
  re-solves a fixed-capacity network and compares
  dispatch-by-carrier and demand against EIA/GridEmissions actuals.
  Useful pattern, but not aimed at prices.
- **Price/LMP backtesting — missing.** No code anywhere compares
  modeled marginal prices to actual ISO settlement prices
  (CAISO/ERCOT/ISO-NE/MISO/SPP/PJM).
- **Day-ahead / real-time market structure — missing.**
  Single-settlement cost optimization. No bid/offer stacks, no
  DA-RT spread, no virtual bidding, no CRTs.
- **Sub-hourly resolution — missing.** Everything (demand, weather,
  dispatch) is hourly. Real-time markets clear every 5 minutes.

## Recommended strategy (revised after discussion)

Four decisions changed the shape of the plan from a straight
"close every gap" list to something narrower and sequenced:

1. **Dispatch output is a feature set, not a price.** Don't try to
   make the physical optimization itself replicate ISO
   market-clearing (bid stacks, uplift, strategic offers). Instead,
   extract dispatch-derived signals (shadow price, net load,
   marginal fuel, reserve margin, congestion flags) and fit a
   statistical/ML calibration layer against actual historical
   prices. The residual between modeled and calibrated price is
   where you look to decide which structural enhancement (below) is
   actually worth building - don't build ancillary-service or
   outage modeling speculatively before the residual shows you need
   it.
2. **Day-ahead price levels first; day-ahead/real-time spread is
   deferred, likely indefinitely.** DA-RT spread is driven mostly by
   forecast error (load, wind, solar, outages) that a deterministic
   dispatch model has none of by construction. Approximating it
   would mean injecting synthetic forecast-error distributions - a
   separate project. Get single-settlement (or DA) price calibration
   working and validated before spending any effort here.
3. **One ISO at a time, starting with ERCOT.** Energy-only market,
   simplest settlement structure, no capacity market, well-documented
   public data (ERCOT MIS). Prove the full loop - dispatch, actuals,
   calibration, backtest - end to end on ERCOT before touching a
   second ISO (CAISO is the natural second, since the repo already
   has some CAISO gas-price plumbing). Don't parallelize across ISOs
   before the first backtest is good; the feature set and data
   pipeline are ISO-specific enough that parallelizing early just
   multiplies debugging surface.
4. **Prototype outside Snakemake; formalize later.** Snakemake's job
   is reproducibly rebuilding the dispatch model itself across
   config combinations (interconnect, clustering, scenario) - that's
   build-system overhead for the planning use case, not the
   calibration research question. The `solve_network_validation`
   rule already produces a solved `.nc` network with
   `n.buses_t.marginal_price` in it. For the prototype, load that
   network directly in a standalone script/notebook and compare
   against a historical price CSV pulled separately - no new rules,
   no wildcards. Only write a `calibrate.smk`-style rule once the
   calibration approach is proven and you want it to re-run
   automatically (e.g. monthly, as new actuals arrive).

## Detailed steps

### 1. Data collection

- **Historical ISO LMPs (day-ahead first).** Start with ERCOT MIS
  (`http://www.ercot.com/mktinfo/prices`) day-ahead settlement point
  prices for the hubs/nodes that map onto your clustered buses.
  Pull at least 1-2 years of hourly history to get seasonal
  coverage. This is the calibration target and does not exist
  anywhere in the repo today.
- **Regional gas price basis.** The repo only fetches a CAISO-region
  gas series (`retrieve_caiso_data.py`), pinned to 2019 by default.
  For ERCOT, pull Waha hub gas prices (or another appropriate
  regional index); for a later CAISO pass, extend the existing
  CAISO fetch range past 2023.
- **Generator outage / derate data.** NERC GADS or EIA-860M planned
  outage schedules, if available; at minimum a forced-outage-rate
  by technology to derate thermal availability, since the model
  currently appears to assume full nameplate availability.
- **Weather forecast data (not just reanalysis).** See the Weather
  forecasts section below - this is its own line item because it's
  a different data type (forecast vs. historical actual) from what
  the model uses today.
- Keep all of this ISO-scoped to ERCOT first (rule 3 above); don't
  build a multi-ISO data-fetch abstraction until there's a second
  ISO to generalize from.

### 2. Model input data updates

These are config/data-pin bumps, not structural changes - do them
early since they're cheap and directly affect calibration quality:

- Bump `pudl_path` in `config.common.yaml` to a current PUDL
  snapshot (controls EIA-860/923/AEO/ATB vintage together - the
  single highest-leverage pin in the repo).
- Bump `costs.ng_fuel_year` in `config.default.yaml` off its 2019
  default, and extend `retrieve_caiso_data.py`'s usable range past
  2023.
- Bump `renewable_weather_years` off its 2019 default. For a
  backtest against a specific historical period, this should match
  the years you're calibrating against, not an arbitrary fixed year.
- Re-pull the EIA state-to-state gas capacity files bundled in the
  sector Zenodo bundle (currently dated Jan 2023 / Feb 2024) and
  republish, if the natural-gas sector coupling is in scope for the
  trading use case at all (it likely isn't for an ERCOT
  electricity-price-only first pass - skip if so).
- Reconcile the `data-costs.md` claim of "2024 ATB" against whatever
  ATB edition is actually bundled in the PUDL snapshot you pin to.

### 3. Model structure enhancements

Build these only where the calibration residual (step 4) actually
points at them - listed here in the order they're likely to matter,
but treat this as a candidate list, not a checklist to complete
up front.

- **Ancillary services / operating reserves.** The current
  `operational_reserve` constraint (`opts/reserves.py`) is a single
  aggregate load/VRE-based margin, off by default, and not
  ISO-product-accurate. ERCOT specifically prices reserves
  (RRS, ECRS, non-spin, regulation) as separate co-optimized
  products with their own clearing prices, and ERCOT's ORDC folds
  reserve scarcity directly into the energy price - this is likely
  a real contributor to under-predicted price spikes. A reasonable
  first cut: model a single ERCOT-style scarcity adder (an ORDC-like
  price curve as a function of available reserves) as a
  post-processing step on top of the existing shadow price, before
  building full multi-product co-optimized reserves. Only build the
  full multi-product version if the calibration residual shows the
  simple adder isn't enough.
- **Outage/derate modeling.** Apply forced-outage-rate derating to
  thermal capacity (from step 1's data) rather than assuming full
  availability - this directly affects how often the model produces
  a scarcity/tight-margin hour at all.
- **Full (non-linearized) unit commitment for backtest runs.**
  Linearized UC understates price spikes from genuinely discrete
  start/stop decisions near tight margins. Worth enabling true MILP
  UC specifically for the backtest/calibration runs, even if
  planning runs elsewhere stay linearized for tractability.
- **Price decomposition (energy / congestion / loss).** Once you
  have a reference (hub) bus per zone:
  `marginal_price[node] - marginal_price[hub] = congestion + loss`.
  Useful once you're validating congestion behavior specifically,
  not required for an initial hub-level price calibration.
- **Explicitly out of scope for now:** full market-clearing
  mechanics (bid/offer stacks, uplift/make-whole payments), DA/RT
  settlement split, sub-hourly resolution. These are the items most
  likely to be *unnecessary* if the feature-based calibration layer
  (step 4) already closes most of the gap - revisit only if the
  calibration residual analysis specifically points at market
  microstructure rather than a fundamentals-level gap.

### 4. Price calibration

The core of the new approach - treat the dispatch model as a
feature generator and calibrate against actuals statistically,
rather than trying to make the physics itself market-accurate.

1. **Baseline comparison.** Run the existing `_operations`
   fixed-capacity re-solve for the ERCOT test period, extract
   `n.buses_t.marginal_price`, and diff against the historical DA
   settlement point prices from step 1 - no model yet, just
   modeled-vs-actual. This tells you the size of the gap before you
   build anything further.

   *Corrected framing:* the original wording here said "at nodes
   matching your ERCOT hubs", which implies an identity between model
   buses and ISO settlement points that does not exist. A model bus is
   not an ERCOT settlement point. Any such comparison is between a
   *model-bus shadow price* and a *commercial settlement price*
   associated with it by a documented, confidence-scored crosswalk.
   See [platform-architecture.md](../platform-architecture.md).
2. **Feature extraction.** Per node/hour, pull from the solved
   network: shadow price, net load (load minus wind/solar), binding
   marginal fuel/carrier, system-wide and local reserve margin,
   congested-line flags, and the gas price basis used that hour.
3. **Calibration model.** Join features to actual historical LMPs
   and fit a regression against the residual (actual − modeled).
   Start with the simplest thing that could work - an hour-of-day /
   season bias correction, or a linear model with fuel-price
   interactions - before reaching for gradient boosting or anything
   more complex. Only add model complexity if the simple version's
   residual still shows structure.
4. **Backtest metrics.** Track MAE and bias by hour-of-day and
   season, and overlay modeled vs. actual price duration curves.
   Systematic bias patterns (e.g. under-prediction concentrated in
   summer peak hours) are the signal for which structural
   enhancement in step 3 to prioritize next - e.g. persistent
   peak-hour under-prediction points at missing scarcity pricing
   (ORDC adder) before it points at needing full ancillary-service
   co-optimization.
5. **Iterate against the residual, not the roadmap.** Each
   structural enhancement in step 3 should be justified by a
   specific pattern in this residual, not added speculatively.

### 5. Weather forecasts

Distinct from the historical-weather-year staleness issue (the
`renewable_weather_years: [2019]` default, addressed in step 2) -
this is about whether the model can use forward-looking weather at
all, which matters once calibration moves from pure backtesting
toward anything forward-looking:

- **Current state:** the model uses a single deterministic
  historical weather year (via Atlite/ERA5 reanalysis or the
  GoDEEEP dataset) to generate renewable capacity factor profiles.
  This is appropriate for backtesting against a matching historical
  period, but there is no forecast or ensemble weather input path.
- **For backtesting (near-term focus):** no change needed beyond
  matching the weather year to the period you're calibrating
  against (step 2) - historical reanalysis is the right input here.
- **For anything forward-looking (later, only if in scope):** would
  need either (a) a weather-forecast feed (e.g. GFS/ECMWF forecast
  data rather than ERA5 reanalysis) for short-horizon forward runs,
  or (b) a multi-year historical ensemble (multiple
  `renewable_weather_years`, not just one) to characterize the
  distribution of outcomes rather than a single deterministic
  trace, which matters for anything resembling risk/hedging
  analysis. Treat this as a later-phase item - it isn't needed to
  validate whether the fundamentals-plus-calibration approach works
  at all.

### 6. Tooling / workflow approach

- Keep the existing Snakemake pipeline exactly as-is for building
  and solving the dispatch model itself.
- Prototype the calibration loop (steps 1-4 above) entirely outside
  Snakemake: a plain script/notebook that loads the already-solved
  `.nc` network via `pypsa.Network(...)` and a separately-fetched
  historical price CSV.
- Only formalize into a Snakemake rule (e.g. `rules/calibrate.smk`,
  mirroring the existing `validate.smk` pattern) once the
  calibration methodology is validated on ERCOT and you want it to
  run repeatably - for example, re-run monthly as new actuals
  arrive, or extended to a second ISO.

## Revised phased roadmap (superseded)

Superseded by [platform-roadmap.md](../platform-roadmap.md), which keeps
the ERCOT-first and residual-driven sequencing below but reorganises
it around the two-spatial-layer architecture and adds explicit
kill-gates. Retained here for provenance.

### Phase 0: prove the gap is measurable (ERCOT, no calibration yet)

- Pull ~1-2 years of ERCOT DA hub LMPs.
- Run the existing `_operations` re-solve for the matching period.
- Diff modeled vs. actual directly, no model, no Snakemake changes.
- Decision point: this tells you the actual size of the gap before
  investing further.

### Phase 1: feature-based calibration (ERCOT, DA prices)

- Build the feature set and calibration model described in step 4.
- Backtest and characterize the residual by hour/season.
- Bump the cheap stale-data pins from step 2 in parallel.

### Phase 2: targeted structural enhancements (ERCOT)

- Address only the structural gaps the Phase 1 residual actually
  points at - most likely candidates, in probable order of impact:
  outage/derate modeling, an ORDC-style scarcity adder, gas basis
  refinement, full MILP UC for backtest runs.
- Re-run the Phase 1 backtest after each change to confirm it
  actually moved the residual before adding the next one.

### Phase 3: replicate to a second ISO (CAISO)

- Only after Phase 1-2 produce a backtest you trust for ERCOT.
- Reuse the calibration methodology; expect to rebuild the feature
  set and data pipeline for CAISO's market structure rather than
  assuming ERCOT's generalizes directly.

### Phase 4 (deferred, revisit only if justified): market microstructure and sub-hourly

- DA/RT settlement split, sub-hourly resolution, full
  market-clearing mechanics (bid stacks, uplift).
- Only pursue any of this if Phases 1-3 show the
  fundamentals-plus-calibration approach has a ceiling that these
  specifically would break through - not as a default next step.

---

Findings drawn from `docs/source/data-*.md`, `workflow/rules/*.smk`,
`workflow/scripts/*.py`, and `workflow/repo_data/` as of this
repository's `master` branch, plus discussion on 2026-07-19/20
covering calibration strategy, DA/RT scope, ISO sequencing, and
Snakemake decoupling.

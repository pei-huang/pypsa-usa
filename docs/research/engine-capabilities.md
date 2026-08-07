# PyPSA-USA as a dispatch engine: capabilities and limits

Internal research note. Not part of the published PyPSA-USA
documentation (`docs/source/`) and not wired into the Sphinx build.

**Answers:** *what does PyPSA-USA actually give us as a dispatch
engine, and where does it fall short?*

**Last updated:** 2026-08-07
**Companion notes:** [platform-architecture.md](platform-architecture.md),
[platform-roadmap.md](platform-roadmap.md),
[../known-issues.md](../known-issues.md)

This is a capability inventory of the upstream engine, kept separate
from the platform design so that "what the tool does" never has to be
disentangled from "what we intend to build." It supersedes the
capability sections of
[archive/trading-gap-analysis.md](archive/trading-gap-analysis.md).

Note the division of labour with [../known-issues.md](../known-issues.md):
this document covers what the engine is *designed* to do or not do;
that one covers where it is *defective* relative to its own design.
A missing feature belongs here; a wrong number belongs there.

## What exists

- **Nodal marginal prices — have.** True LP/MILP duals,
  congestion-inclusive, exposed as `n.buses_t.marginal_price` and
  already plotted (`plot_network_maps.py`, `plot_statistics.py`).
- **Configurable nodal resolution — have.** Node count is a wildcard,
  not fixed — up to ~41k nodes nationally via the TAMU synthetic
  network; ReEDS-zone topology also available.
- **Linearised DC-OPF with real impedances — have.** Lines carry
  r/x/b/g from voltage-class line types, not a transport model.
  Optional piecewise-linear loss approximation.
- **Rolling-horizon solve — have.** `solve_network.py:232-237` already
  wires `n.optimize.optimize_with_rolling_horizon()`. Do not
  reimplement this; see the model architecture section of the
  architecture note for why boundary conditions matter.
- **A constraint library, not a monolith — have.**
  `workflow/scripts/opts/` exposes importable constraint builders:
  `reserves.py` (`add_operational_reserve_margin`,
  `add_ERM_constraints`, `store_ERM_duals`), `interchange.py`
  (`add_interchange_constraints`), `policy.py` (RPS/CES/CO2, out of
  scope for trading). This matters for reuse strategy — see the
  architecture note's Tier 1.5.
- **Unit commitment scaffolding — partial.** Min up/down time,
  start-up cost and ramp rates modelled, sourced from WECC ADS. Off by
  default; solved relaxed (linearised), not full integer.
- **Operating reserve margin — partial.** Generic load/VRE-based
  spinning-reserve constraint exists (`opts/reserves.py`), plus an
  `erm` (energy reserve margin) constraint. Neither is
  ISO-product-accurate: no regulation, non-spin, or frequency-response
  products with their own clearing prices.
- **Dynamic fuel cost hooks — partial.** PUDL-sourced per-plant fuel
  costs and a wholesale-price option exist
  (`add_electricity.py:apply_pudl_fuel_costs`), off by default. See
  [../known-issues.md](../known-issues.md) for a suspected coal-price
  unit defect in this path.
- **Production/demand validation — partial.** `validate.smk` re-solves
  a fixed-capacity network and compares dispatch-by-carrier and demand
  against EIA/GridEmissions actuals. Aggregate-level only, and not
  aimed at prices.
- **Price/LMP backtesting — missing.** No code anywhere compares
  modelled marginal prices to actual ISO settlement prices.
- **Day-ahead / real-time market structure — missing.** Single
  settlement, cost-minimising. No bid/offer stacks, no DA-RT spread,
  no virtual bidding, no CRRs.
- **Sub-hourly resolution — missing.** Everything is hourly; real-time
  markets clear every five minutes.
- **Forced-outage modelling — missing.** The only derates present are
  EIA-860 seasonal capacity ratings, which are an ambient-ratings
  adjustment, not availability. Reserve margin computed this way is a
  margin against nameplate.

## `solve_network_validation` is not a lean operations solve

Worth correcting explicitly, because the platform design depends on
it. [validate.smk:1-38](../../workflow/rules/validate.smk) invokes the
*same* `solve_network.py` (440 lines of capacity-expansion and policy
machinery) as the planning path — it is differentiated by config
(`foresight`, `planning_horizons`, sector params), not by being a
separate, smaller solve.

Two consequences:

1. A fixed-capacity operations mode already works today *via config*.
   Try that before reimplementing a dispatch driver from scratch.
2. Anything reimplemented from scratch risks silently dropping
   behaviour this path already handles (ramp constraints, reserve
   margin interaction, `erm`). See the architecture note's Tier 1.5
   for the selective-reuse position this leads to.

## Where aggregation biases the output

Not merely a loss of resolution — a directional bias, and one the
default config makes worse.

`cluster_network` aggregates generators by carrier, and
`config.default.yaml:204` sets `consider_efficiency_classes: false`.
Every gas unit in a region therefore collapses into a single
carrier-level composite with a blended heat rate. Three consequences,
all pointing the same way:

- The **marginal-fuel / marginal-unit** signal becomes an artifact of
  the blend rather than an actual marginal unit — and it is one of the
  more informative features the dispatch solve produces.
- The **supply curve is smoothed**, systematically compressing price
  spikes.
- **Intra-regional congestion is removed by construction**,
  compressing dispersion further.

So modelled shadow prices are biased low in the tails — precisely the
regime where basis is economically interesting, and precisely what any
scarcity-pricing (ORDC) work is trying to reproduce. Because empirical
exposure regressions use these as factors, the bias propagates into
the exposure estimates.

Setting `consider_efficiency_classes: true` is a config-only (Tier 0)
mitigation and belongs in the first build, not a later refinement.

## Structural enhancements, ranked but not scheduled

Build only where the calibration residual actually points — this is a
candidate list, not a checklist. Ordering reflects likely impact.

- **Scarcity pricing / ancillary services.** ERCOT prices RRS, ECRS,
  non-spin and regulation as separate co-optimised products, and its
  ORDC folds reserve scarcity directly into the energy price. This is
  a likely contributor to under-predicted spikes. First cut: an
  ORDC-style adder as post-processing on the existing shadow price.
  Full multi-product co-optimisation only if the adder proves
  insufficient.
- **Outage / derate modelling.** Apply forced-outage rates to thermal
  capacity as a time-varying `p_max_pu` rather than assuming
  availability. Cheap, does not require modelling outages
  endogenously, and directly affects how often a tight-margin hour
  occurs at all.
- **Full (integer) unit commitment for backtest runs.** Linearised UC
  understates spikes from genuinely discrete start/stop decisions near
  tight margins. Plausibly worth enabling for backtest runs even if
  planning runs stay relaxed.
- **Regional gas basis.** Waha for ERCOT rather than a flat or
  CAISO-region fuel price.
- **Price decomposition** (energy / congestion / loss) once a
  reference hub bus per zone exists.

## Explicitly out of scope

Full market-clearing mechanics (bid/offer stacks, uplift, make-whole
payments), DA/RT settlement split, and sub-hourly resolution. These
are the items most likely to prove *unnecessary* if a statistical
calibration layer already closes most of the gap. Revisit only if
residual analysis points specifically at market microstructure rather
than a fundamentals-level gap.

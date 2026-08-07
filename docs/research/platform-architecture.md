# Power Market Research Platform: architecture

Internal research note. Not part of the published PyPSA-USA
documentation (`docs/source/`) and not wired into the Sphinx build.

**Answers:** *what are we building, structurally?*

**Status:** design, nothing built.
**Last updated:** 2026-08-07
**Companion notes:** [platform-roadmap.md](platform-roadmap.md) (build
order and gates), [engine-capabilities.md](engine-capabilities.md)
(what the upstream engine provides),
[../known-issues.md](../known-issues.md) (upstream defects)

## What this is

An institutional-style power market research platform. It answers one
question:

> What is the physical state of the market, and what does that state
> imply about observed nodal basis?

It is deliberately **not** branded as a trading platform. A research
platform that explains basis is a claim that can be defended with
evidence. A trading platform implies validated P&L on instruments that
have not been modelled.

The platform lives in its own repository (`power-market-platform`).
PyPSA-USA is an upstream *build-time* input, not a runtime dependency
of the daily pipeline. See "The PyPSA-USA boundary" below for how that
dependency is governed — including where the platform legitimately
modifies the network-generation process.

## The claim, stated honestly

The platform says:

> The physical network is derived from and reduced from PyPSA-USA.
> Actual ISO pricing nodes form a separate commercial layer. PyPSA
> outputs are used as structural features to explain and forecast
> observed nodal basis.

It does **not** say:

> We replicate ERCOT or PJM nodal prices on the PyPSA-USA network.

The second claim would require a validated mapping to the ISO
operational topology, plus actual transmission constraints, outage
schedules, offer curves, and unit characteristics that the open model
does not contain.

## Four layers

1. **Digital twin.** A reduced, PyPSA-derived network that simulates
   economic dispatch and DC power flows for a target ISO.
2. **Market intelligence.** Physical indicators derived from the
   dispatch solve: reserve margin, interface loading, marginal
   generator and marginal fuel, renewable curtailment, dispatch by
   carrier, binding-constraint shadow prices.
3. **Quantitative models.** Statistical models combining simulated
   physical features with observed ISO market data to forecast hub
   and nodal basis.
4. **Strategy research.** Backtests against a specific tradeable
   instrument, signal attribution, and performance analysis.

Each layer is only worth building if the layer below it produced
something that survived validation. See the kill-gates in the roadmap.

## The two spatial layers

This is the central design constraint and the easiest thing to get
wrong. There are four distinct node concepts in play:

**PyPSA-USA bus** — a modelled electrical connection point in the
open-source transmission network. *Role: source topology.*

**Reduced model bus** — several PyPSA-USA buses aggregated into one.
*Role: fast simulation. Our own analytical construct.*

**ISO electrical bus** — a bus in ERCOT's or PJM's operational network
model. *Role: actual grid operation. Not available to us.*

**ISO pricing node** — a commercial location at which a price is
calculated and settled. *Role: market data and prediction target.*

PJM defines a pricing node (pnode) as either a single pricing location
*or a subset of pricing nodes* at which injection or withdrawal is
modelled and an LMP is calculated — so a pnode is not necessarily one
physical bus. ERCOT separates electrical-bus LMPs from Settlement Point
Prices at resource nodes, hubs, and load zones, and publishes both a
Settlement Points List and an Electrical Buses Mapping precisely
because the commercial and electrical layers differ.

The platform therefore maintains two parallel spatial layers that are
never conflated:

```text
PHYSICAL MODEL LAYER                 COMMERCIAL MARKET LAYER

PyPSA-USA buses                      ISO pricing nodes, hubs, zones
        |                                        |
        v                                        v
Reduced model buses                  Observed DA / RT LMPs
        |                                        |
        v                                        v
Flows, dispatch, shadow prices       Observed node-to-hub basis
        |                                        |
        +----------- statistical model ----------+
```

The connection is **not** an assertion that model bus `j` equals ISO
node `i`. It is:

```text
basis[i, t] = f( pypsa_system_state[t],
                 iso_market_features[t],
                 weather[t],
                 node_attributes[i] )
```

PyPSA supplies system-state features. Actual ISO data supplies the
commercial target.

### Data contracts

Three tables encode the separation explicitly.

`market_nodes.parquet` — real ISO commercial definitions:

| market_node_id | market | node_type | node_name | zone | hub | lat | lon |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ERCOT_001 | ERCOT | resource_node | ... | West | HB_WEST | ... | ... |
| ERCOT_002 | ERCOT | hub | HB_NORTH | North | HB_NORTH | ... | ... |
| PJM_001 | PJM | pnode | ... | APS | WESTERN_HUB | ... | ... |

`model_buses.parquet` — reduced PyPSA buses, our own construct:

| model_bus_id | source_bus_count | region | lat | lon | load_share |
| --- | --- | --- | --- | --- | --- |
| R01 | 32 | West Texas | ... | ... | ... |
| R02 | 18 | North Texas | ... | ... | ... |

`node_model_crosswalk.parquet` — association metadata, **not** a claim
of electrical identity:

| market_node_id | model_region | model_bus_id | mapping_type | confidence |
| --- | --- | --- | --- | --- |
| ERCOT_001 | West Texas | R01 | regional | 0.80 |
| PJM_001 | Western PJM | R14 | statistical | 0.65 |

The `mapping_type` and `confidence` columns are load-bearing. Anything
consuming the crosswalk must be able to see how the association was
made and how much to trust it.

### Three mapping levels

**Level 1 — regional.** Map each market node to a broad physical
region, and expose regional aggregates: net load, wind output, gas
generation, curtailment, imports/exports, corridor loading. Safest
first version and the only one to build initially.

**Level 2 — nearest bus.** Where reliable coordinates exist, associate
an ISO location with the closest suitable model bus. Proximity is a
*candidate* mapping, never proof: geographically close substations can
be electrically separated, voltage levels differ, the open topology
omits and simplifies lines, and a pnode may be an aggregate.

**Level 3 — empirical.** Regress each node's observed basis on
simulated physical factors:

```text
basis[i, t] = alpha[i] + sum_k beta[i, k] * factor[k, t] + eps[i, t]
```

where factors are things like West-to-North flow, Houston import
requirement, western curtailment, eastern gas dispatch, or the shadow
price on a modelled corridor. The resulting exposure matrix
`beta[i, k]` — node `i`'s sensitivity to physical factor `k` — is
likely more useful for research than an uncertain one-to-one
electrical mapping ever would be.

## Spread definitions

Spreads are defined **only** on actual ISO prices:

```text
node-to-hub basis   B[i, t]     = LMP[i, t] - LMP[hub, t]
day-ahead vs real   DART[i, t]  = LMP_DA[i, t] - LMP_RT[i, t]
node-to-node        S[i, j, t]  = LMP[i, t] - LMP[j, t]
```

The platform does not define an "ISO nodal spread" as the difference
between two PyPSA bus prices. That quantity is a *simulated model-bus
spread*: usable as a feature or scenario diagnostic, never presented
as an observed or tradeable price.

## Role of PyPSA shadow prices

Modelled shadow prices are **features, not targets**. This is a
demotion, not a deletion. The dual on a binding corridor is the
densest single output of the dispatch solve — it encodes why flows
are what they are — and dropping it would discard the main thing the
optimisation buys us. What changes is that it is never compared
directly to a settlement price as if it were one.

## Resolution and what it can resolve

Reduced-network size is a trade-off against the congestion thesis, and
the honest position must be stated up front:

- A ~75-bus ERCOT model resolves roughly a dozen meaningful
  interfaces. That supports **zonal interface** congestion factors and
  nothing finer.
- Node-level congestion is therefore captured **statistically**, via
  Level-3 beta exposures, not physically.
- Start at ~75 buses to close the end-to-end loop quickly, then raise
  it. A 250–400 bus DC-OPF over 8760 hours is comfortable on a laptop
  with HiGHS; the LP solve is not the bottleneck, data plumbing is.
  Freezing permanently at 75 would be a mistake.

Two consequences that are easy to understate:

**Aggregation biases the duals, it does not merely coarsen them.**
Clustering aggregates generators by carrier, and
`config.default.yaml:204` defaults `consider_efficiency_classes` to
`false`, collapsing every gas unit in a region into one composite with
a blended heat rate. The marginal-fuel signal becomes an artifact of
the blend, the supply curve is smoothed (compressing spikes), and
intra-regional congestion vanishes by construction. Modelled shadow
prices are therefore biased *low in the tails* — exactly where basis is
economically interesting — and the bias propagates into any exposure
regression using them as factors. Setting
`consider_efficiency_classes: true` is a Tier-0 mitigation and belongs
in the first build. See
[engine-capabilities.md](engine-capabilities.md).

**A regional crosswalk makes the feature set rank-deficient.** With
Level-1 regional mapping over ~75 buses, every settlement point inside
a region receives *identical* physical features, so the physical block
can only explain the region-common component of basis. Meanwhile the
naive baseline — lagged basis — already captures node-specific
persistent structure, and basis is highly persistent. The incremental
information channel is roughly a dozen interfaces wide against a
target with thousands of node-dimensions. This is why the roadmap's
kill-gate must not be evaluated at 75 buses: a failure there cannot
distinguish "physics does not explain basis" from "we compressed the
physics into twelve numbers."

## Offline / online split

The dashboard never solves an optimisation.

```text
offline:  run dispatch -> extract features -> write parquet artifacts
online:   dashboard loads artifacts
```

Artifacts are the interface between the two. The dashboard opens
instantly because it does nothing but read files.

## The PyPSA-USA boundary

The platform is a separate repository, but *not* a frozen vendor drop.
Modifying the network-generation process is expected and legitimate.
What matters is not whether the fork is edited, but how the dependency
surface is governed.

### Why separate repositories at all

- The runtime dependency is one-directional and build-time only. Once
  `market_network.nc` exists, the daily pipeline never touches
  PyPSA-USA again.
- This is a fork of an active upstream (`PyPSA/pypsa-usa`). Trading
  code committed here becomes permanent merge-conflict surface and
  forecloses contributing anything back.
- The Snakemake wildcard axis is `interconnect / simpl / clusters /
  opts / sector` — scenario combinations. The platform's axis is
  *date*. Fighting that is fighting the build system.
- Different lifecycles: PyPSA-USA builds a scenario and solves once;
  the platform holds one fixed network and re-solves on new data.

### Your policy, their mechanism

Clustering is precisely where planning and trading intent diverge.
PyPSA-USA clusters to shrink an investment model; the platform
clusters to preserve congestion interfaces. Same object, different
objective.

But clustering has two separable halves:

- the **decision** — which buses group together. Encodes intent.
  This is ours.
- the **mechanics** — aggregating generators by carrier, summing line
  capacities, recomputing equivalent impedances, dissolving
  geographic regions, carrying load shares. Fiddly, correct, already
  written. No reason to reimplement.

`cluster_network.py` already splits along exactly this line. If
`enable.custom_busmap` is set and a busmap CSV is supplied, it skips
kmeans/HAC/greedy-modularity entirely and clusters to the given
grouping. The platform therefore computes a congestion-aware busmap in
its own repository and hands it back — owning the decision, inheriting
the mechanics, with zero upstream edits for what would otherwise be
the single largest modification.

`cluster_network` also already emits `busmap.csv` and `linemap.csv`,
which supply the source-bus provenance that `model_buses.parquet`
requires. No upstream change needed there either.

### Four tiers of change

Every intended modification is classified before it is written. The
tier determines where it lives.

**Tier 0 — config values.** Cluster count, weather years,
interconnect. *Home: platform config.*

**Tier 1 — injected input.** Custom busmap, substituted data files.
*Home: platform repo, via existing hooks.*

**Tier 1.5 — selective library reuse.** Importing upstream constraint
builders without inheriting the driver around them.
`workflow/scripts/opts/` is a constraint *library*, not a monolith:
`reserves.py` exposes `add_operational_reserve_margin`,
`add_ERM_constraints` and `store_ERM_duals` (the last is already the
pattern for extracting reserve shadow prices as features);
`interchange.py` exposes `add_interchange_constraints`. The platform
wants these and does not want `policy.py` (RPS/CES/CO2). *Home:
platform repo — a thin solve driver that imports selectively.*

This tier exists because the reuse question is not the binary the
original design implied. "Reimplement thirty lines" discards working
reserve logic; "reuse `solve_network.py`" inherits capacity-expansion
and policy machinery that is out of scope. Selective import is
neither, and is where most of the real reuse lives.

**Tier 2 — additive upstream change.** A new output, a new optional
parameter. *Home: fork branch. Contributable upstream.*

**Tier 3 — algorithmic divergence.** Different logic inside an
existing path. *Home: platform repo, as its own module.*

The rule that governs the fork:

> **The fork may be edited, but only additively.** The moment a change
> rewrites existing behaviour instead of adding beside it, it
> graduates out of the fork and into the platform repository.

This is what makes rebases survivable. Additive changes rebase cleanly
almost indefinitely; in-place rewrites conflict on every upstream touch
of the same file. Tier-2 changes have a further advantage: they can be
contributed upstream, which retires them from maintenance entirely.

Practical test: if a rebase onto upstream gets ugly, that change was
secretly Tier 3. Move it rather than fighting it.

### The seam

The handoff artifact is **`elec_s{simpl}.nc`** — simplified, fully
data-populated, *un-clustered* — not the clustered network.

```text
PYPSA-USA FORK  (additive edits only)

  build_shapes -> build_base_network -> build_powerplants
    -> build_renewable_profiles -> add_electricity
    -> simplify_network
                    |
              elec_s.nc                     <- the seam
                    |
PLATFORM REPO
                    v
  network_builder/  congestion-aware busmap ---+
                                               |  back through
                                               |  cluster_network via
                                               |  custom_busmap
                                               v
                             market_network.nc + busmap + linemap
                                               |
  dispatch/            own n.optimize(), fixed capacity
  feature_engineering/ models/ strategies/ dashboard/
```

Nothing below `market_network.nc` imports PyPSA-USA.

### What is reused, and what is not

Reused, once, at build time:

| Stage | PyPSA-USA script |
| --- | --- |
| Topology and shapes | `build_base_network`, `build_shapes` |
| Bus regions | `build_bus_regions` |
| Plants | `build_powerplants` |
| Renewable profiles | `build_renewable_profiles` |
| Attach components | `add_electricity` |
| Simplify | `simplify_network` |
| Cluster mechanics | `cluster_network` (with custom busmap) |

Not reused:

- `prepare_network` — adds expansion options, CO2 limits, policy
  constraints. All of it out of scope.
- `solve_network` — capacity-expansion objective plus policy
  constraint machinery. Not what the platform solves.

The platform's solve is fixed capacity, no extendable components, no
policy constraints, minimising dispatch cost over a rolling horizon.

**Correction to an earlier framing.** This was previously described as
"roughly thirty lines of PyPSA — read `solve_network_validation` as a
reference, then reimplement." That understates what already exists.
`validate.smk` invokes the *same* `solve_network.py`, differentiated by
config rather than by being a smaller solve — so a fixed-capacity
operations mode already works today via configuration. See
[engine-capabilities.md](engine-capabilities.md) for detail.

The resulting position, in order of preference:

1. Try operations-mode-via-config first. It exists and is exercised.
2. Where a thin driver is genuinely needed, import constraints
   selectively per Tier 1.5 rather than rewriting them.
3. Reimplement only what is actually policy-specific and unwanted.

Reimplementing wholesale risks silently dropping behaviour the
existing path handles (ramp constraints, reserve-margin interaction,
`erm`) — the kind of subtle correctness gap the platform can least
afford in its one load-bearing solve.

### Fork hygiene

- `master` tracks upstream, untouched.
- A `platform` branch carries Tier-2 changes only, rebased on upstream
  periodically.
- Pin by commit SHA. Submodule or sibling checkout — the pin matters,
  the mechanism does not.
- **Emit a build manifest** alongside `market_network.nc`: upstream
  commit SHA, config hash, busmap hash, build timestamp. This is what
  makes an edited upstream safe rather than frightening — any artifact
  traces to exactly the code and inputs that produced it, and staleness
  is visible at a glance.

## Data pipeline design

`data_pipeline/` is the component every other layer depends on and the
one most likely to rot into one-off scripts if left undesigned. This
section specifies its shape now so that when
[Phase 6](platform-roadmap.md#phase-6--data-platform-refactor) actually
triggers, it's an implementation of a design, not an improvisation.
Nothing here is built yet — same status as the rest of this document.

### Source categories

Five distinct kinds of input feed the platform, and they differ enough
in access pattern and refresh needs that no single fetch abstraction
should try to unify all of them:

| Category | Examples | Cadence | Access pattern |
| --- | --- | --- | --- |
| Settlement prices | ERCOT MIS, CAISO OASIS, PJM DM2 | daily | bulk append |
| Fleet data | PUDL (EIA-860/923), ATB | per release | full replace |
| Fuel prices | EIA gas, Waha basis | daily/weekly | bulk, append-only |
| Outages/derates | NERC GADS, EIA-860M | daily | bulk, append-only |
| Weather | ERA5 (Atlite), GODEEEP | per weather-year | one-time, large file |

Weather is intentionally not redesigned here — the existing
Atlite/ERA5 and GODEEEP paths PyPSA-USA already uses are adequate for
the platform's offline dispatch runs (see the gap analysis's weather
section) and are out of scope for this refactor unless forward-looking
forecast ingestion is later added.

### Adapter interface

One adapter per **source**, not per (ISO, source) pair. The interface
is the same shape for every category above; only the implementation
differs:

```text
class DataSource(Protocol):
    name: str                    # e.g. "ercot_mis_da_spp"
    schema: SourceSchema         # declared columns, dtypes, units, tz
    refresh_cadence: Cadence     # e.g. Daily, PerRelease, OneTime

    def fetch(self, since: Timestamp | None) -> RawBatch: ...
    def validate(self, batch: RawBatch) -> RawBatch: ...   # raises on schema violation
    def write(self, batch: RawBatch) -> None: ...           # to the raw layer
```

`ERCOTMisSource`, `CAISOOasisSource`, and `PJMDataMiner2Source` all
implement `DataSource` for the "historical settlement prices"
category; nothing ISO-specific leaks outside the adapter itself. This
is the concrete fix for how `retrieve_caiso_data.py`-style scripts
accreted assumptions over time — a new ISO means a new adapter class,
not a new branch inside shared code.

### Schema contracts

Every adapter declares `schema` up front: column names, dtypes,
**units**, and timezone, and `validate()` enforces it on every write,
not just at read time. This is the direct, structural answer to the
unit-conversion bugs tracked in
[../known-issues.md](../known-issues.md) (coal fuel price units, the
suspected MWh/kWh mixup) — a declared unit that's checked on ingest
can't silently drift the way an implicit one can.

### Bitemporality is not optional

Every record carries **two** timestamps, and this is the single most
important property of the store:

- `event_time` — when the thing happened or applied to.
- `knowledge_time` — when it became *knowable* to us (published,
  posted, released).

The reason is a lookahead hazard that would otherwise silently
invalidate every backtest. ERA5 is *reanalysis*: realised weather,
published with lag. The day-ahead market cleared on *forecast*
weather. Dispatch features derived from ERA5 therefore contain
information that did not exist at DA clearing time, and the leakage
concentrates exactly where it matters most — high-wind and
tight-margin hours, where forecast error is largest and prices most
extreme. The same applies throughout: outage records (NERC GADS,
EIA-860M) describe what *happened*, not what was posted and known at
clearing; PUDL is an annual snapshot, so mid-year retirements and
additions mean the modelled fleet is not the fleet that operated on a
given historical date.

The rule that follows: **feature extraction joins with
`knowledge_time <= decision_time`**, where `decision_time` is the
market decision being modelled (DA clearing, CRR auction close). A
feature that cannot satisfy this constraint is not a feature; it is
leakage.

This is far cheaper to design in now than to retrofit after two years
of history have been ingested with a single timestamp column. Example
schema for a settlement-price source:

| field | dtype | unit | notes |
| --- | --- | --- | --- |
| `event_time` | datetime64 | — | UTC, converted on ingest |
| `knowledge_time` | datetime64 | — | UTC, publication time |
| `node_id` | string | — | source-native id |
| `price` | float64 | `$/MWh` | declared, validated |
| `source` | string (const) | — | adapter name |

Where a source genuinely has no publication timestamp, the adapter
must record its best conservative estimate and flag it — never
silently default `knowledge_time` to `event_time`, which is precisely
the leak this design exists to prevent.

### Storage layers

Two layers, not one — raw and normalized are kept separate so a schema
fix or a new normalization rule never requires re-fetching:

```text
raw/        source=ercot_mis_da_spp/year=2024/*.parquet
              (as fetched, validated, immutable)
normalized/ prices/year=2024/*.parquet
              (unioned across sources, common schema)
```

Physical format: a partitioned parquet lake, consistent with the
`artifacts/` convention already used for `market_nodes.parquet` etc.
An embedded analytical engine (DuckDB) queries directly over the
parquet — no server, still just files, git/backup-friendly, but SQL
joins across sources (e.g. price joined to concurrent outage state)
without hand-rolled pandas merges. A managed time-series database
(TimescaleDB/Postgres) is deliberately not the default: justified only
if a query pattern actually needs it (a live dashboard querying
directly, or write-heavy frequently-refreshed operational data) —
evaluate against real usage from the calibration work, not upfront.

### What this replaces

Today: `retrieve_caiso_data.py` and siblings, a hand-pinned
`pudl_path` string in `config.common.yaml`, Zenodo bundles of
inconsistent vintage, no declared schema anywhere, no unified query
layer. This design doesn't change any of that yet — see Phase 6's
trigger condition in the roadmap for when it does.

## Model architecture

How the dispatch engine is actually structured and driven. The
platform's axis is **date**; PyPSA-USA's is **scenario**. Most of what
follows falls out of taking that difference seriously.

### `market_network.nc` carries static components only

PyPSA-USA bakes 8760 hours of load and VRE profiles into its `.nc`
outputs. If `market_network.nc` inherits that, then changing the date
range, updating a weather year, or injecting a corrected outage series
requires re-running the PyPSA-USA build chain — which is exactly the
dependency this design claims to have severed.

So the artifact splits along the axis that varies:

```text
market_network.nc     topology, buses, lines, generators, static attrs
                      (rebuilt rarely; the only PyPSA-USA dependency)
timeseries store      load, VRE, fuel price, outage-derated p_max_pu
                      (from data_pipeline; injected at solve time)
```

Nothing time-varying is baked into the network artifact. This is what
makes the date axis genuinely independent of the build axis, and it is
a precondition for both warm-starting and ensembling below.

### Build the LP once, not once per date

`n.optimize()` rebuilds the entire linopy model on every call. For a
workload of ~730 daily solves across a two-year backtest — repeated
for every iteration of the calibration loop — model *construction*, not
simplex, is plausibly the dominant cost.

Construct once via `n.optimize.create_model()`, then update only the
parameters that change between dates and re-solve reusing the prior
basis. Consecutive days are nearly identical LPs, which is close to
the ideal case for warm-starting.

This also corrects what the Phase 0 gate measures. "One 8760-hour
solve in under 30 minutes" is the wrong benchmark for a workload that
is actually many small repeated solves; benchmark the loop, not the
single run, because it determines how many calibration iterations are
affordable — and therefore whether the later phases are practical at
all.

### Rolling horizon with state carryover — already implemented

`solve_network.py:232-237` already wires
`n.optimize.optimize_with_rolling_horizon()`. Do not reimplement it.

It matters more than it appears. The DA market clears 24 hours at a
time with lookahead, carrying storage state of charge and commitment
status across the boundary. Solving "one day of snapshots" without
specifying boundary conditions gets storage arbitrage wrong at every
day boundary — and in ERCOT batteries are now a major setter of
peak-hour prices. Distorted peak prices corrupt the physical features
precisely in the hours that carry the most information about basis.

### The solve is a pure function

```text
solve(network, timeseries, config) -> outputs
```

No file-path side effects. With this, ensemble runs — N perturbed
input draws yielding a *distribution* of prices rather than a point
estimate — are trivial parallelism. With a netCDF-file-per-run design
they require an orchestration layer.

Ensembles are not needed initially, but distributional output is
necessary for anything resembling risk or hedging analysis, which a
trading application eventually is. Purity is free if designed in from
the start and expensive to retrofit.

### Invariant assertions in the solve wrapper

Given the class of defect tracked in
[../known-issues.md](../known-issues.md), the solve driver asserts on
every run rather than trusting outputs: energy balance (served load
equals demand), total system cost reconciles against dispatch times
marginal cost, marginal prices within a plausible `$/MWh` band,
capacity factors within `[0, 1]`, **and `n.buses_t.marginal_price` is
non-empty for every bus and snapshot**. That last one is not
hypothetical: with the repo's current default solver options
(`ipm`, `run_crossover: off`), HiGHS returns `status=unknown` on a
majority of sampled weeks on the `western`/33-cluster network,
and PyPSA correctly refuses to populate duals when that happens —
`n.buses_t.marginal_price` comes back with zero columns, silently,
concentrated in exactly the high-load weeks a trading platform cares
about most. See the corresponding entry in
[../known-issues.md](../known-issues.md) for the reproduction and the
`run_crossover: on` fix, which resolved every tested case at
comparable wall time.

Each is a few lines, and together they catch the entire *class* of
unit bug — a factor-of-1000 error fails a plausible-range assertion
immediately and loudly, instead of propagating silently into a
calibration residual that then costs weeks to interpret. This is the
runtime counterpart to schema validation on ingest: validate on write,
assert on solve.

## Repository layout

```text
power-market-platform/
├── data_pipeline/       ISO market data, weather, fuel prices
├── network_builder/     busmap construction, extraction, manifest
├── dispatch/            daily economic dispatch runs
├── feature_engineering/ physical features from solved networks
├── models/              basis forecasting
├── strategies/          instrument-level backtests
├── dashboard/           read-only artifact viewer
└── artifacts/           parquet outputs + build manifest
```

`network_builder/` is the only component that imports PyPSA-USA.

## Dashboard: two views, never merged

**Physical simulation view** — reduced model buses, simulated lines,
flows, utilisation, dispatch, modelled shadow prices. Every label
reads "model bus" or "simulated corridor".

**Market node view** — actual ISO pricing nodes, hubs, historical or
predicted basis, congestion clusters, signals.

The two may be overlaid geographically but must be visually distinct:

```text
○   reduced model bus
●   ISO pricing node
──  simulated transmission corridor
··  empirical node-to-model-region association
```

## What is explicitly out of scope

Carried over from the PyPSA-USA feature set and deliberately dropped:
capacity expansion, investment optimisation, carbon scenarios,
hydrogen, sector coupling, storage expansion planning, multi-decade
simulation, future technology assumptions. The platform models daily
market operations only.

Also out of scope until evidence demands otherwise: full
market-clearing mechanics (bid/offer stacks, uplift, make-whole
payments), sub-hourly resolution, and integer unit commitment.

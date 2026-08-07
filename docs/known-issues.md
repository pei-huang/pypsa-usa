# PyPSA-USA: known issues and technical debt

Working backlog of defects and technical debt in **this repository** —
the PyPSA-USA modelling engine itself. Not part of the published
Sphinx docs (`docs/source/`) and not wired into the doc build.

**Answers:** *what is broken in PyPSA-USA right now?*

**How this differs from `docs/research/`:** those notes plan a
*separate downstream project* that consumes PyPSA-USA as an input —
see [research/README.md](research/README.md). This document is the
opposite: it tracks things that are wrong *inside* PyPSA-USA — bugs,
unit errors, stale carry-over from the PyPSA-Eur fork, sharp edges.

The dividing line is **defect vs. limitation**. A wrong number belongs
here. A feature the engine was never designed to have belongs in
[research/engine-capabilities.md](research/engine-capabilities.md).
"Coal prices convert wrong" is the first; "no ancillary-services
market model" is the second.

Fixing these is now sequenced as **Phase 0a** in
[research/platform-roadmap.md](research/platform-roadmap.md) — the
downstream Phase 2 gate is specified to run on this model's output, so
unverified unit errors can produce either a false pass or a false kill
on the decision the whole plan turns on.

**Last updated:** 2026-08-07

## How to use this doc

- Add an entry as soon as you notice something wrong, even without
  time to root-cause it yet — `Status: reported` with as much detail
  as you have is more useful than not writing it down.
- One entry per distinct bug. Don't batch unrelated issues.
- When an entry is fixed, move it to **Resolved** with the commit/PR
  reference instead of deleting it — it's a record of what was wrong
  and why, useful for anyone who hits the same symptom later.
- Severity is about impact on model correctness/output, not effort to
  fix: **high** = silently wrong numbers or a crash on common configs,
  **medium** = wrong in specific/uncommon configs or workarounds
  exist, **low** = cosmetic, dead code, or minor inefficiency.

## Open issues

### Default solver settings silently drop all marginal prices in most weeks

- **Area:** solver config (`config.default.yaml` `solving.solver_options.highs-default`),
  affects every rule that reads `n.buses_t.marginal_price`
- **Severity:** high — confirmed, not suspected
- **Status:** partially fixed, and the fix is confirmed **not
  sufficient on its own**. `config.default.yaml`'s `run_crossover`
  flipped to `"on"` 2026-08-07, verified on 56-snapshot windows and on
  a real full-scale (365-snapshot, 12-cluster, `status=optimal`,
  crossover-optimal) production solve — see the separate "Empty duals
  on an optimal, crossover-on solve" entry below for that case, which
  is NOT explained by the mechanism described in this entry (that one
  is `status=unknown`; this one is a clean `status=optimal` with a
  verified-correct crossover run, and still zero duals). Treat
  `run_crossover: on` as necessary but not proven sufficient.
  `config.tutorial.yaml`'s matching `highs-default` block left as
  `"off"`: that config's active solver is `gurobi`, so the block is
  dead config there, unexercised by HiGHS.
- **Description:** The repo's default HiGHS options
  (`solver: ipm`, `run_crossover: off`) return HiGHS status
  `unknown` (not `optimal`) often enough that PyPSA correctly refuses
  to populate duals at all — `n.buses_t.marginal_price` comes back as
  an empty DataFrame (0 columns), not merely imprecise. Tested on the
  `western`/33-cluster/2019 network (`elec_s75_c33_ec_lv1.0_REM-3h_E.nc`)
  by solving nine independent 56-snapshot (one-week) windows spread
  across the year with the exact repo default options: **6 of 9
  windows — including the highest-load week of the year (centered
  2019-07-26) — returned `status=unknown` and zero duals.** Only
  low-demand winter weeks reliably returned `status=optimal`.
  Switching `run_crossover` to `on` solved every tested window
  cleanly (`status=optimal`, full duals) in about the same wall time
  (3.1–3.8s per window in this test, i.e. crossover was not
  meaningfully slower here). HiGHS also repeatedly warns
  `Problem has some excessively large column bounds` /
  `Consider scaling the bounds by 1e-4` on this network.
  **Investigated and mostly ruled out as a separate bug:** this
  network has zero `Store` components, so it isn't the `e_nom=1e9`
  sentinels in `add_extra_components.py` (see the next entry below —
  those are real but inactive here). The likely explanation is the
  205 of 387 generators with `p_nom_max = inf` (correctly unbounded,
  extendable capacity) — solvers can't write literal infinity into an
  LP/MPS file, so PyPSA/linopy substitutes a large finite value when
  serializing, which is standard and not itself a bug. Not chasing
  this further; it's plausibly just what triggers HiGHS's scaling
  heuristic on any network with many unbounded extendable generators.
- **Impact:** `n.buses_t.marginal_price` is the single most
  load-bearing output for anything price-related — see
  [research/platform-architecture.md](research/platform-architecture.md)
  and [research/platform-roadmap.md](research/platform-roadmap.md).
  Any downstream code that doesn't explicitly check the shape of this
  DataFrame will silently get no data (or propagate `NaN`s) during
  exactly the tight/peak-demand hours that matter most for price
  analysis, with no error raised — the solve itself reports success.
  This directly undermines the Phase 2 gate in the roadmap: features
  extracted from a run with empty duals are not "no signal", they're
  missing data that could be silently treated as zero.
- **Next steps:** verify wall time at full network scale (2,920
  snapshots, not 56) — crossover cost is superlinear in problem size
  (37,411 dual pushes were needed just for one 56-snapshot window), so
  the "comparable wall time" finding above does not necessarily hold
  at full size. The original `western`/33-cluster, full-year
  crossover-off benchmark attempt was killed after ~17 minutes without
  finishing (per-iteration IPM time had grown ~20x mid-solve), so
  there is no completed full-scale timing in either configuration yet
  — this is the actual gap, not just a crossover-on confirmation.
  Separately, investigate the large-column-bounds warning — find and
  fix whatever attribute is set to a `~1e9` sentinel, since it's
  plausibly the root cause of the conditioning problem rather than
  just the crossover setting.

### Empty duals on an optimal, crossover-on solve (unreproduced)

- **Area:** `n.buses_t.marginal_price`, discovered via
  `plot_statistics.py`'s `plot_region_lmps`, but the underlying issue
  is in the solve/dual-extraction step, not the plotting code.
- **Severity:** high — silent, no error raised, on a solve that
  otherwise reports full success.
- **Status:** reported, root cause not found despite substantial
  investigation; worked around in `plot_statistics.py` (skip the LMP
  plot with a warning instead of crashing) but the actual defect is
  still live.
- **Description:** a real production solve
  (`western`, `topological_boundaries='state'`, 12 clusters, sector
  `E`, full year at `REM-24h` = 365 daily snapshots,
  `run_crossover: on`) completed with `Model status: Optimal`,
  `Status crossover: optimal`, and `basic solution dual infeasibility:
  2.78e-17` (numerically clean) — yet
  `n.buses_t.marginal_price.shape == (365, 0)`: **zero columns**, not
  imprecise values, no values at all. `n.buses_t.p` (power injections)
  populated correctly `(365, 12)` in the same solve, so this is
  specifically a dual-extraction gap, not a failed solve.

  This is a **different mechanism** from the "silently drop all
  marginal prices" entry above — that one is `status=unknown` on a
  degenerate/ill-conditioned solve; this one is a clean `status=optimal`
  with verified-correct crossover, and duals are still missing.

  Extensive attempts to reproduce by re-solving the saved (already
  solved) network directly all **failed to reproduce it** — every one
  of these gave correctly populated `(N, 12)` marginal prices instead:
  - Fresh `n.optimize()` on the exact network structure (0 lines, 48
    links, 12 buses) at 3, 30, and 60 snapshots.
  - With `multi_investment_periods=True` and `assign_all_duals=False`
    (matching `solve_network.py`'s actual kwargs).
  - With the real `extra_functionality` from `solve_network.py`
    applied (`add_regional_co2limit` via the `REM` opt, using the
    actual saved run's config), at up to 60 snapshots.
  - With the exact real solver options including `threads=4` (I had
    been testing with `threads=2`, copied from an unrelated earlier
    fix, before catching the mismatch).

  A network with **zero `n.lines`** (pure `Link`-based transport
  model) was specifically tested in isolation and did NOT reproduce
  emptiness on its own — a minimal 3-bus/2-link/no-lines network
  solved with correct per-bus duals. So "no lines" alone isn't
  sufficient to trigger this either.
- **Leading hypotheses, none confirmed:**
  1. Genuinely scale-dependent — only manifests at the full
     365-snapshot problem; I only tested reproduction up to 60
     snapshots (compute cost of going further wasn't spent here).
  2. Depends on the network's state *before* its first solve in a way
     that can't be replicated by re-solving the already-solved output
     — e.g. extendable generators starting from `p_nom_min` on the
     original solve vs. already at `p_nom_opt` on a re-solve, changing
     degeneracy characteristics enough to hit a different code path in
     linopy's/PyPSA's dual-assignment loop
     (`pypsa/optimization/optimize.py`'s `assign_duals`, which walks
     `m.constraints` looking for names ending in `nodal_balance` — see
     that file for the exact logic if picking this up).
- **Impact:** any run matching this network's shape (transport model,
  no lines, `topological_boundaries` other than the default
  `reeds_zone`) may silently lose all price output while reporting
  full success. Anything consuming `n.buses_t.marginal_price` from
  such a run — including the trading-platform work in
  [research/platform-architecture.md](research/platform-architecture.md) —
  must check `.shape[1] > 0` before trusting it, not just check solve
  status.
- **Next steps:** reproduce starting from the *pre-solve* network
  (`solve_network`'s actual input, not its output) at full scale
  (365 snapshots) rather than a re-solved copy, since that's the one
  variable not yet isolated. If reproducible, bisect by
  `topological_boundaries` (`reeds_zone` vs `state`) to see if this is
  specific to the coarser aggregation. Add the invariant assertion
  from [research/platform-architecture.md](research/platform-architecture.md)'s
  "Invariant assertions in the solve wrapper" section
  (non-empty `n.buses_t.marginal_price`) to `solve_network.py` itself
  so this fails loudly at solve time instead of silently at plot time.

### Coal fuel price unit/conversion bug

- **Area:** fuel prices (`build_fuel_prices.py`, `eia.py`)
- **Severity:** high (suspected — unverified)
- **Status:** reported, not yet root-caused
- **Description:** Coal fuel price values coming out of the pipeline
  look wrong, suspected to be a unit conversion bug (e.g. $/short ton
  vs $/MMBtu, or a missing heat-rate conversion) somewhere in
  `get_state_coal_power_prices` (`build_fuel_prices.py:75`) or the EIA
  `_CoalCosts`/`_FutureCosts` extractors in `eia.py`.
- **Impact:** downstream generator marginal costs for coal plants are
  potentially wrong wherever this data is used.
- **Next steps:** trace the unit through `eia.py`'s coal cost
  extractors end to end and compare against a known-good EIA
  $/MMBtu or $/ton reference value for a specific state/month.

### MWh/kWh unit bug carried over from PyPSA-Eur

- **Area:** likely `solve_network.py` or nearby cost/constraint code
- **Severity:** high (suspected — unverified)
- **Status:** reported, location not yet pinned down
- **Description:** A hard-coded unit mismatch (MWh vs kWh) believed to
  be stale/inherited from the original PyPSA-Eur model, not something
  introduced in this fork. Exact location unknown.
- **Impact:** unknown until located — potentially affects cost
  reporting or a constraint coefficient by a factor of 1000.
- **Next steps:** grep `solve_network.py` and cost-assembly code for
  literal `1000`/`1e3`/`0.001` unit-conversion factors and check each
  against the units of its inputs.

### Leap years drop Feb 29 / undercounts snapshots

- **Area:** snapshot generation (`_helpers.py`, `build_renewable_profiles.py`,
  `build_demand.py`, `add_demand.py`, `build_fuel_prices.py`, `add_sectors.py`)
- **Severity:** medium
- **Status:** partially fixed
- **Description:** `get_snapshots()` in `_helpers.py` defaults to
  `drop_leap_day=True`, so leap years silently produce 8760 hours
  instead of 8784 wherever that default isn't overridden. Two
  sub-issues:
  1. `get_renewable_snapshots()` in `build_renewable_profiles.py` — **fixed**
     2026-08-06, now passes `drop_leap_day=False`. (Note: this
     function and the `renewable_snapshots` config block are not
     currently wired into any Snakemake rule — `build_renewable_profiles`
     still consumes the flat `config["snapshots"]` param at
     `build_renewable_profiles.py:108`, which does not go through this
     function.)
  2. `get_multiindex_snapshots()` (`_helpers.py:821`), used by
     `build_demand.py` and `add_demand.py` for multi-horizon
     snapshots, has a deeper structural issue: it generates one
     template date range from a fixed non-leap template year (config
     default 2019) and reuses it for every planning horizon via
     `.replace(year=X)`. Since the template year has no Feb 29 to
     begin with, no planning-horizon year — leap or not — can ever
     get Feb 29 this way, regardless of `drop_leap_day`. Needs a
     per-year date range (same approach as the renewable fix) rather
     than a templated-and-shifted one.
- **Impact:** leap years are modelled with 24 fewer hours than they
  should have; demand-side multi-horizon snapshots can never include
  Feb 29 at all.
- **Next steps:** decide whether to flip the `get_snapshots()` default
  outright (touches fuel prices, sector snapshots, plotting too — and
  needs the underlying weather/demand sources checked for Feb 29 data
  first, or `.sel(time=...)` calls will start raising `KeyError`s
  instead of silently truncating), then rework
  `get_multiindex_snapshots()` to generate per-year ranges.

### linopy 0.3.14 misdetects highspy >= 1.10 (latent, not currently hit)

- **Area:** solver stack (`linopy` 0.3.14, pinned in `pyproject.toml`)
- **Severity:** low as configured — but high if `io_api: "mps"` is ever set
- **Status:** open upstream (linopy), documented here as a trap
- **Description:** `linopy/solvers.py:63` compares versions as
  *strings*: `if version("highspy") < "1.7.1"`. Lexicographically
  `"1.15.1" < "1.7.1"` is `True` (since `"1" < "7"` at the third
  character), so every highspy from 1.10.0 onward is misdetected as
  pre-1.7.1 and `_new_highspy_mps_layout` is wrongly set to `False`.
- **Impact:** at `solvers.py:150`, that flag guards
  `solution.objective *= -1` — a silent **objective sign flip** — but
  only when `io_api == "mps"`. PyPSA-USA never sets `io_api`, so it
  defaults to `None`, which writes an LP file, not MPS. The bug is
  therefore not triggered today. Verified empirically after the
  highspy 1.15.1 upgrade: objective and constraint duals are identical
  to highspy 1.9.0 on the repo's exact solver options.
- **Next steps:** do not set `io_api: "mps"` while `linopy` is pinned
  at 0.3.14. Fixed upstream by upgrading linopy (a larger change,
  since `linopy==0.3.14` and `pypsa==0.30.2` are exact pins that move
  together).

### DuckDB `spatial` extension bypasses `s3_url_style` for geometry columns

- **Area:** `build_demand.py`'s `ReadFERC714._read_census_data` (the
  only caller of the census query in the codebase)
- **Severity:** medium — blocks FERC714-based demand reading against
  PUDL's dotted-name S3 bucket; does not affect the default `ReadEia`
  demand path
- **Status:** root cause not found; workaround not found either
- **Description:** the fix below (`s3_url_style='path'`) resolves
  every plain (non-geometry) `read_parquet('s3://pudl.catalyst.coop/...')`
  call verified against real `v2026.7.2` data. It does **not** resolve
  the one query that selects a `geometry` column
  (`out_censusdp1tract__states.parquet` via `gpd.read_postgis`) — that
  one still fails with the same vhost-style SSL error even with
  `spatial` explicitly pre-installed/loaded and with an explicit
  `duckdb.connect()` connection object shared consistently across
  `INSTALL`/`LOAD`/`SET`/read calls (ruled out: extension-autoload
  timing, and connection-object mismatch). Most likely explanation,
  not yet confirmed: the `spatial` extension's geometry decoding
  routes S3 access through GDAL's own virtual filesystem layer
  internally, which has separate S3 configuration from DuckDB's
  `httpfs` extension and doesn't inherit `s3_url_style`.
- **Impact:** `ReadFERC714` (an alternate demand-reading strategy, not
  the default) cannot read census population-weighting data from this
  PUDL bucket at all right now.
- **Next steps:** try GDAL-level env vars
  (`AWS_S3_ENDPOINT`/`CPL_VSIL_CURL_USE_HEAD`/`VSI_CACHE`) rather than
  DuckDB settings; or avoid the geometry column entirely by reading
  non-geometry columns via plain `read_parquet` (confirmed working)
  and joining state boundaries from a different, non-PUDL source.

### PUDL data is fetched over the network on every run — no real cache

- **Area:** every `duckdb read_parquet('s3://pudl.catalyst.coop/...')`
  call — `build_powerplants.py` (`initialize_duckdb`,
  `load_eia_operable_data`, `load_heat_rates_data`),
  `build_cost_data.py`, `build_demand.py` (2 sites, one of them the
  still-broken geometry query above)
- **Severity:** medium — not a correctness bug, but a reliability and
  performance one, and it's what turned this session's debugging into
  a multi-hour saga: the TLS/SSL dotted-bucket-name error, the DuckDB
  OOM, and general network flakiness were all encountered *while
  fetching PUDL data live*, not from anything in the model logic.
- **Status:** reported. A manual, out-of-band workaround exists (a
  scratch shell script that `aws s3 cp`s the ~9 needed files to
  `~/pudl-mirror/{version}/` and points the user's local, gitignored
  `pudl_path` at it) but nothing in the repo or pipeline does this
  automatically, and the *tracked* config template still points
  `pudl_path` straight at `s3://pudl.catalyst.coop/...`.
- **Description:** `pudl_path` is just a string — every script that
  reads from it does a live `read_parquet('s3://...')`, once per rule
  invocation, with no Snakemake-managed local artifact in between.
  Concretely this means: (1) the same PUDL files get re-fetched
  independently across scenarios (`Default` and `Historical-2019`)
  that both need them, (2) every re-run of a downstream rule (e.g.
  after a code change, per this session's `--rerun-triggers mtime`
  workflow) re-fetches from the network rather than reading a local
  file, and (3) the pipeline has zero resilience to PUDL's S3 being
  slow, rate-limiting, or transiently unavailable — it just fails.
  This is also the concrete, present-day version of the "input data is
  scattered, no unified store" architecture gap already named in
  [research/platform-architecture.md](research/platform-architecture.md)'s
  data-pipeline design and [research/platform-roadmap.md](research/platform-roadmap.md)'s
  Phase 6 — this entry is that problem showing up as an actual bug,
  not just a design concern.
- **Impact:** slower, more fragile builds; redundant network transfer;
  and — per this session — a live S3 dependency turns an ordinary code
  change into a debugging session about TLS certificates and memory
  limits, none of which have anything to do with the actual pipeline
  logic being changed.
- **Next steps:** turn PUDL retrieval into a real Snakemake rule (e.g.
  in `retrieve.smk`) that downloads each needed parquet file once per
  `pudl_path` version into a local `data/pudl/{version}/` directory as
  a proper rule output, and have `build_powerplants.py`/
  `build_cost_data.py`/`build_demand.py` take that local path as a
  `snakemake.input` rather than constructing an `s3://` URL from a
  config string directly. This gets caching, provenance tracking, and
  cross-scenario sharing (via the existing `shared_resources` config
  option) for free, instead of requiring every user to separately
  discover and run a manual mirror script.

## Resolved

### `consider_efficiency_classes` crashes on storage-only carriers

- **Area:** `cluster_network.py`, the `consider_efficiency_classes`
  code path (`config.default.yaml`'s
  `clustering.cluster_network.consider_efficiency_classes`)
- **Fixed:** 2026-08-07.
  [cluster_network.py:941-960](../workflow/scripts/cluster_network.py)
- **Description:** `aggregate_carriers` is built from the union of
  `n.generators.carrier` and `n.storage_units.carrier`, but this loop
  only ever queries `n.generators` for each carrier. For a
  storage-only carrier (`battery` — exists only in `storage_units`,
  never `generators`), `n.generators.query("carrier == @c")` returns
  an empty frame, so `.quantile()` returns `NaN` for both `low` and
  `high`. The existing skip-guard (`low >= high or
  low.round(2)==high.round(2)`) silently fails to catch this — `NaN`
  comparisons are always `False` in Python/pandas, so the guard
  doesn't fire, and `pd.cut(..., bins=[0, NaN, NaN, 1])` raises `bins
  must increase monotonically`.
- **Why this was never caught before:** `consider_efficiency_classes`
  defaulted to `false`, so this code path had never actually run in
  practice — flipping it to `true` (as part of the aggregation-bias
  fix — see [research/engine-capabilities.md](research/engine-capabilities.md))
  was the first time it was exercised against a real network, which is
  exactly when the bug surfaced.
- **Fix:** check `gens.empty` explicitly before computing quantiles,
  rather than relying on the `NaN`-comparison guard to catch it
  implicitly.
- **Verified:** ran the exact carrier-splitting loop against the real
  `western`/`Historical-2019` network — completes without error,
  `battery` correctly passes through unmodified, and the five thermal
  carriers that should split (`CCGT`, `OCGT`, `coal`, `biomass`,
  `waste`) correctly produce `low`/`medium`/`high` efficiency
  sub-carriers.

### `e_nom=1e9` used as an "unbounded" sentinel instead of `np.inf`

- **Area:** `add_extra_components.py` (demand-response and
  import/export `Store` components)
- **Fixed:** 2026-08-07. [add_extra_components.py:755,769,1020,1036](../workflow/scripts/add_extra_components.py)
  now use `np.inf`, matching the correct pattern already used two
  lines above one of these sites (`p_nom=np.inf` on the
  demand-response link). Verified: `ruff` clean, and a real
  `n.optimize()` solve still produces the correct objective/duals/flows
  after the change.
- **Note:** was inactive in the tested network (zero `Store`
  components in the electricity-only sector configuration) — fixed
  preemptively since it's the same class of bound-scaling risk as the
  solver-settings issue above, on networks that do use these
  components (demand response or interregional import/export
  enabled).

### PUDL S3 access fails with a TLS/SSL error (dotted bucket name)

- **Area:** all `duckdb` `read_parquet('s3://pudl.catalyst.coop/...')`
  calls — `build_powerplants.py`, `build_cost_data.py`,
  `build_demand.py` (2 sites)
- **Fixed:** 2026-08-07. Symptom: `_duckdb.IOException: IO Error: SSL
  peer certificate or SSH remote key was not OK error for HTTP HEAD to
  'https://pudl.catalyst.coop.s3.amazonaws.com/...'`.
- **Root cause:** the PUDL bucket name (`pudl.catalyst.coop`) itself
  contains dots. DuckDB's default S3 addressing is virtual-hosted-style
  (`s3_url_style='vhost'`), which folds the bucket name into the
  hostname — producing `pudl.catalyst.coop.s3.amazonaws.com`. AWS's
  wildcard TLS cert for S3 (`*.s3.amazonaws.com`) is single-level and
  does not match a hostname with the bucket's own dots folded in, so
  TLS verification fails. This is a known, general S3 gotcha for any
  dotted bucket name, not specific to PUDL or to this repo.
- **Fix:** `SET s3_region='us-west-2'; SET s3_url_style='path';`
  before any `read_parquet` call against this bucket, switching to
  `https://s3.us-west-2.amazonaws.com/pudl.catalyst.coop/...`
  (bucket as a path segment, not folded into the hostname). Applied
  immediately after `INSTALL httpfs; LOAD httpfs;` at all four call
  sites.
- **Verified:** against real `v2026.7.2` data, not just synthetic
  files — both `core_eia860__scd_plants.parquet` (plain columns) and
  the AEO projection query in `build_demand.py` succeeded after the
  fix. One related query does **not** work even after this fix — see
  the open "DuckDB `spatial` extension bypasses `s3_url_style`" entry
  above.

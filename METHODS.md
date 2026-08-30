# Methods & Metrics — guide_assignment

Condensed reference. Full algorithmic detail:
`/data/yunzliu/assignment_benchmark_starter/METHODOLOGY_ASSIGNMENT_ALGORITHMS.md`.

## Assignment methods (module → lineages)

| Module | Lineages (`parameters`) | Vendored script | Sort key | Notes |
|---|---|---|---|---|
| `pgmm` | `variant: mle` / `map_e2` | `run_pgmm_em.py` / `run_variant_e2.py --e2-prior` | UMI DESC | 2-comp Poisson-Gaussian EM; UMI≥1 + P(Gauss)≥0.75 gate. Seeded ⇒ deterministic. E2 = MAP prior; base 6-col output (LPO layer dropped — falsified). |
| `umi` | `threshold: 3/5/10` | `run_umi_threshold.py` | UMI DESC | Parameter-free baseline; precision–recall frontier. Deterministic. |
| `crispat` | `crispat_method: pgmm/2beta` | `run_crispat_pgmm.py` / `run_crispat_2beta.py` (+`mex_to_h5ad.py`) | UMI / percent_counts DESC | Pyro SVI. MEX→h5ad done inside the module. **Stochastic** — parity via pinned env + seed (see CONSISTENCY_VALIDATION). |
| `fishash` | fixed | `run_fishash.R` | log_pval ASC | One-sided Fisher exact + GS-FDR 5% + Simpson refit×10 (R). Deterministic. |

All modules consume the MEX trio (`data.matrix/barcodes/features`), present it to the
vendored script as `merged_{matrix,barcodes,features}`, and emit `assignments.csv`
with each method's native columns (so the archived md5 pins hold).

## Metrics (metric selection via `--metric`)

Idiomatic Omnibenchmark: each metric value is its own node/output, **not** a
monolithic scorer (matches the official clustering example). Modules split only
where (env, input-shape) differ.

| `--metric` | Vendored logic | Inputs | State |
|---|---|---|---|
| `tier1` | `benchmark_assignments.py` (`compute_metrics`, dual) / `benchmark_papalexi_tier1.py` (single) | assignments + GT + guide_map + spec | ✅ dual verified; single = Cut 3 |
| `construct_set` | `construct_level_eval.py` (dual only) | assignments + guide_map + spec | ⏳ Cut 2 |
| `kd` / `discovery` / `mismatch` | `benchmark_kd_efficiency.py` / `phase2_discovery.py` / `analyze_mismatch.py` | + `data.gex` | ⏳ Cut 2 (GEX env) |
| `strat_tier1` / `strat_mismatch_loc` / `capacity` | difficulty Phase 1A/1C/3 | + `difficulty.table` | ⏳ Cut 3 |

The Tier-1 loader **auto-detects the CSV schema** (percent_counts→2beta,
log_pval→fishash, prob_gaussian→pgmm, else UMI) to pick the exact per-method sort
the original used — so the metric needs no cross-stage knowledge of the method, and
reproduces archived values field-for-field.

### Dataset semantics (`data.spec`)

The importer emits `{dataset}_spec.json` = `{guide_design, match_regime,
collapse_lanes, has_gex}`. Metrics read it — dual vs single Tier-1, and the barcode
matching regime: Replogle `(lane,16mer)` over 48 real lanes (never collapse),
Papalexi bare-16mer (8 pools folded to L01, ceiling 20,441). Methods stay
dataset-agnostic.

## Cross-lineage aggregation (collectors)

The `collectors` entrypoint of `guide_assignment_metrics`: `jaccard` (all lineages; per-cell
aggregation — the fixed 2026-08-05 behaviour), `strat_jaccard` (Phase 1B),
`extraction_shift` (Phase 1E, joins the two extractions' difficulty tables).

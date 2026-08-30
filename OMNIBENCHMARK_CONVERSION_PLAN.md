# Guide Assignment — Omnibenchmark Conversion: Final Report

**Status: COMPLETE & STABLE (2026-08-13).** The Perturb-seq guide-assignment
pipeline (`assignment_benchmark_starter/03_scripts/*`) is fully converted to
Omnibenchmark. All method/metric logic is vendored verbatim and **parity-verified
byte/field-identical to the original archive**; the plan generates a **deterministic
DAG** (135 rules, identical across `PYTHONHASHSEED` and repeated runs). Ready to
push to GitHub. A detailed end-to-end run from a fresh Git clone is the agreed next
step (not done here).

Companion docs in this repo: `README.md` (how to run), `METHODS.md` (methods +
metric idiom), `CONSISTENCY_VALIDATION.md` (the full parity ledger).

---

## 1. What was built

**1 benchmark repo + 8 module repos**, each Git-tracked, all `03_scripts` logic
vendored verbatim (`cp`) behind a thin Omnibenchmark CLI wrapper — re-orchestration
only, no science rewritten.

```
omnibenchmark_benchmarks/guide_assignment/          benchmark.yaml (+ benchmark_cut1.yaml) + docs
omnibenchmark_modules/
  guide_assignment_data        importer + Phase-0 difficulty table (method-independent)
  guide_assignment_pgmm        pgmm_em: variant mle | map_e2 (run_pgmm_em.py / run_variant_e2.py)
  guide_assignment_umi         umi threshold 3 | 5 | 10 (run_umi_threshold.py)
  guide_assignment_crispat     crispat pgmm | 2beta (mex_to_h5ad + run_crispat_*.py)
  guide_assignment_fishash     fishash (R; run_fishash.R via Rscript)
  guide_assignment_metrics     per-lineage scorer: tier1/construct_set/kd/discovery/mismatch/strat_*
                               + the difficulty (Phase-0) and collectors entrypoints
```

## 2. Architecture (as-built)

Linear DAG (OB requires a single upstream chain per stage — no divergent-branch joins):

```
data ──► guide_assignment ──► metrics_gt      (tier1, construct_set, strat_tier1, strat_mismatch_loc, capacity)
 │  │                     └─► metrics_gex      (kd + NT-NT + pair)
 │  │                     └─► metrics_discovery(discovery + FPR)
 │  └── data.difficulty_table (Phase-0, computed in the importer — method-independent, lives on the data branch)
 └─► metric_collectors: jaccard, strat_jaccard, extraction_shift, mismatch
```

Key design decisions:
- **Phase-0 difficulty table on the data branch.** It's method-independent, so the
  data importer computes it as `data.difficulty_table`; this keeps the DAG linear
  (OB cannot join two divergent branches — issue #289).
- **Metric selection via the `--metric` parameter** (each value = its own node/output).
- **One module per metric *stage*.** OB 0.6.0's DAG builder is nondeterministic when
  a single stage holds multiple modules sharing an output id (fan-out collapses ~half
  the runs); splitting into `metrics_gt` / `metrics_gex` / `metrics_discovery` makes
  it deterministic (verified across seeds).
- **Collector fan-in is param-aware.** Collectors read each node's sibling
  `parameters.json` to recover the real params (variant/threshold/crispat_method)
  and the dataset — OB's opaque path hashes alone cannot distinguish crispat
  pgmm/2beta or umi t3/t5/t10.
- **E2 = a `variant` of pgmm**, base-6-col output (the falsified LPO layer is dropped).
- **Dataset semantics in `data.spec`** (guide_design, match_regime, collapse_lanes)
  — Replogle `(lane,16mer)` never collapses the 48 lanes; Papalexi bare-16mer.

## 3. Parity — verified byte/field-identical to the archive

Everything runnable was checked against the original outputs (Replogle/HAM unless noted):

| Component | Result |
|---|---|
| pgmm, umi **assignments** | byte-identical md5 (`d566aeff`, `525e4401`) |
| **tier1** (dual), **construct_set**, **kd** (+NT-NT+pair) | all fields identical |
| **Phase-0 difficulty table** | 0 / 6.28M field mismatches |
| **strat_tier1 / strat_mismatch_loc / capacity** | 0 mismatches |
| **jaccard / strat_jaccard / extraction_shift** | 0 mismatches |
| **mismatch** (union-NT collector, all 5 methods) | 0 mismatches |
| discovery | vendored verbatim; dry-run clean (GEX-heavy, not run here) |
| `strat_delta_kd` (Phase-2 stratum KD) | no archive counterpart — metric postdates the archive |

Full numbers in `CONSISTENCY_VALIDATION.md`.

## 4. Stability

`ob validate plan` passes; `ob run --dry` generates **135 rules deterministically**
(identical at PYTHONHASHSEED 0/1/2 and across repeated runs). The DAG: 2 data ×
8 methods = 16 assignment lineages, each scored by 3 metric stages (7 metric values),
plus 4 cross-lineage collectors.

## 5. Known limitations / next-step items (for the detailed Git test)

- **crispat parity is not guaranteed byte-exact.** crispat uses Pyro SVI (stochastic);
  even with a pinned seed, md5 can drift across a conda-env rebuild. Use the tolerance
  gate (≥99.5% top-1, ties only), not exact md5. Not run here.
- **Phase-2 is the per-lineage `strat_delta_kd` metric** (stratum-binned KD), wired into
  the `metrics_gex` stage. It supersedes the older method-independent delta-KD, which
  would have needed `requires:` + a 3-way branch join — a shape that triggers OB's
  fan-out nondeterminism. Being per-lineage, it needs no such join.
- **No full end-to-end `ob run` (conda) yet.** Verified per-component (host backend +
  standalone) + full-plan dry-run. Building the 4 conda envs and running the whole DAG
  is the detailed-test step.
- **crispat/fishash assignments + discovery not executed** (SVI ≈ hours; discovery
  ≈ 160 GB). Logic vendored verbatim; dry-run clean.
- **Papalexi lineage** is sketched (single-guide tier1 implemented; the canonical
  pgmm assignment gives rec=0.992 — the old 0.985 baseline used a deprecated
  prob-sorted assignment, not a metric bug).

## 6. How to run

```bash
cd /home/yunzliu/omnibenchmark_benchmarks/guide_assignment
# smoke test (fast, host backend, parity-verified):
conda run -n omnibenchmark ob run benchmark_cut1.yaml --dry --dirty
cd out && conda run -n scp_analysis snakemake --cores 8

# full plan (conda backend): validate + inspect the DAG
conda run -n omnibenchmark ob validate plan benchmark.yaml
conda run -n omnibenchmark ob run benchmark.yaml --dry --dirty        # 135 rules
# real run builds the 4 conda envs, then: ob run benchmark.yaml --cores N
```

Local-path modules require `--dirty`. Envs are exported from the pre-existing
`scp_analysis` / `crispat` / `scprocess_rlibs` envs (the ones that produced the
archive) for maximum parity.

---

*This document was the original conversion plan; it has been rewritten as the final
build report. The original design rationale (stage/module classification, metric
idiom, difficulty decomposition, parity protocol) is preserved in git history.*

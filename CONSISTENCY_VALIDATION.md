# Consistency / Parity Validation — guide_assignment

**Prime directive:** the Omnibenchmark conversion is re-orchestration only. Results
after conversion must equal results before. This ledger records the parity checks.

Method logic is **vendored verbatim** (`cp` of the original `03_scripts/*` files —
md5-identical) and wrapped behind the module CLI; only orchestration is replaced.

---

## Cut 1 — Replogle / HAM — VERIFIED ✅ (2026-08-13)

Ran `benchmark_cut1.yaml` (host backend, `scp_analysis` env):
`data.replogle_ham → {pgmm(mle), umi(t3)} → metrics(tier1)`.

### Assignment parity — byte-identical md5 (base columns)

| Method | OB output md5 | Archived pin (`DATA_INDEX.md §1`) | Result |
|---|---|---|---|
| pgmm_em (mle) | `d566aeff24894979aa9e2a0c1f26093c` | `d566aeff…` (`05_pgmm_em_assignment/ham`) | ✅ identical |
| umi_t3 | `525e44012238ac721aed02c427d5000b` | `525e4401…` (`08_umi_crispat/ham/t3`) | ✅ identical |

### Tier-1 metric parity — every field identical to `10_benchmark_results/*.json`

| Metric | pgmm_em OB / archive | umi_t3 OB / archive |
|---|---|---|
| cell_recovery_rate | 0.988321 / 0.988321 ✅ | 0.983234 / 0.983234 ✅ |
| n_shared_cells | 306760 / 306760 ✅ | 305181 / 305181 ✅ |
| t1_pair_accuracy | 0.982521 / 0.982521 ✅ | 0.984825 / 0.984825 ✅ |
| t2_pair_accuracy | 0.992981 / 0.992981 ✅ | 0.994374 / 0.994374 ✅ |
| t3_pair_accuracy | 0.993425 / 0.993425 ✅ | 0.994577 / 0.994577 ✅ |
| ari | 0.974577 / 0.974577 ✅ | 0.978351 / 0.978351 ✅ |
| per_construct_pearson_r | 0.989737 / 0.989737 ✅ | 0.990758 / 0.990758 ✅ |
| guides_per_cell_median | 2.0 / 2.0 ✅ | 2.0 / 2.0 ✅ |
| umi_median | 34.0 / 34.0 ✅ | 55.0 / 55.0 ✅ |

**Conclusion:** zero substantive difference. Byte-identical assignments ⇒ identical
downstream metrics, as designed. The `pgmm_em` seed (`RandomState(42+init)`) and the
`scp_analysis` env reproduce the archive exactly.

### How to reproduce

```bash
cd /home/yunzliu/omnibenchmark_benchmarks/guide_assignment
conda run -n omnibenchmark ob run benchmark_cut1.yaml --dry --dirty   # generate Snakefile + cache modules
cd out && conda run -n scp_analysis snakemake --cores 8               # host backend -> scp_analysis python
# then md5sum the produced assignments.csv vs the DATA_INDEX pins
```

---

## Metric parity — per-lineage refactors verified vs archive ✅ (2026-08-13)

Each metric refactored to score ONE lineage (vendored logic imported or copied
verbatim) and checked against the archived monolith output, on Replogle/HAM/pgmm_em:

| Metric | Source (reuse) | Check vs archive | Result |
|---|---|---|---|
| `tier1` (dual) | `benchmark_assignments.compute_metrics` (import) | `10_benchmark_results/pgmm_em__ham.json` | ✅ all 9 fields identical |
| `construct_set` | `construct_level_eval.py` (copied) | `B1_construct_set_eval.json[pgmm_em__ham]` | ✅ all 9 fields identical (F1 0.895375) |
| `kd` (+nt_nt +pair) | `benchmark_kd_efficiency.py` (import) | `12_kd_efficiency/…/pgmm_em__ham.json` | ✅ ALL fields identical (incl nested) — see KD row |
| **`strat_tier1`** (Phase 1a) | `difficulty_fix_all.py` (copied) | `phase1a_stratum_metrics.json[pgmm_em__ham__*]` | ✅ **0 mismatches** (4 strata × 11 fields) |
| **`strat_mismatch_loc`** (Phase 1c) | `difficulty_fix_all.py` (copied) | `phase1c_mismatch_loc.json[pgmm_em__ham]` | ✅ **0 mismatches** |
| **`capacity`** (Phase 3) | `difficulty_fix_all.py` (copied) | `phase3_capacity.json[pgmm_em__ham]` | ✅ **0 mismatches** (mean_gpc 3.23, slope 1.711) |
| **`discovery`** (Phase 2 crispat) | `phase2_discovery.py` (import) | 160 GB — wired + dry-run verified; per-method NT = archive semantics | ⏳ logic reused verbatim, full run not on this box |
| **difficulty table** (Phase 0) | `difficulty_phase0_build_table.py` (copied) | `cell_difficulty_ham.tsv` | ✅ **0 mismatches** (628,397 cells × 13 cols = 6.28M) |
| `jaccard` (collector) | `compute_jaccard.py` (copied) | `_jaccard.json` | ✅ matches |
| **`strat_jaccard`** (Phase 1b collector) | `difficulty_fix_all.py` (copied) | `phase1b_jaccard_stratum.json[ham]` | ✅ **0 mismatches** (40 pair-strata) |
| **`extraction_shift`** (Phase 1e collector) | `difficulty_fix_all.py` (copied) | `phase1e_extraction_shift.json` | ✅ **0 mismatches** (n_common 628366) |
| **`mismatch`** (Phase 3 collector, union-NT) | `analyze_mismatch.py` (copied) | `_mismatch_arbitration.json` (all 5 methods) | ✅ **0 mismatches, all 5 methods** — win_rate pgmm_em 0.4456 / crispat_pgmm 0.4806 / crispat_2beta 0.4693 / fishash 0.3746 / umi_t3 0.4776, all = archive (union-NT pool = 23,279 cells). Per-method NT gave 0.4425 → this is why it's a collector, not a per-lineage metric. |
| **Phase-2 delta-KD** (difficulty validate) | `difficulty_phase2_delta_kd_ham.py` (copied) | GEX-heavy; wired + dry-run verified | ⏳ deterministic (seeded), full run not on this box |
| `tier1` (single/Papalexi) | `benchmark_papalexi_tier1.py` (copied) | rec=0.992075 on canonical `02_results/pgmm_em/ham` | ✅ metric correct — see note below |

**KD verified ✅** — `kd.scores.json` on Replogle/HAM/pgmm_em vs
`12_kd_efficiency/replogle2022/pgmm_em__ham.json`: **all fields identical**
(n_guides 3945, KD median −1.8274, mean −2.3826, std 2.0246, frac_expected 0.9612,
frac_strong 0.9006) **including** nested `nt_nt_baseline` (kd_median −0.0241,
n_tests 3894) and `pair_consistency` (concordance 0.3866, delta_median 0.7363,
n_pairs 1847). GEX loaded 643,651 × 2044 (per-gene ε, min 5 cells) — the vendored
`benchmark_kd_efficiency` functions reused unchanged.

### Papalexi Tier-1 (single) — resolved: assignment vintage, not a metric bug

The metric gives rec=0.992 on the **canonical** pgmm assignment
(`02_results/pgmm_em/ham`, 59,739 rows, gpC med 2, current config UMI≥1+prob gate).
The old published `03_benchmark/_papalexi_summary.json` (rec=0.985, n_shared=20138)
was computed on a **different, older pgmm assignment** — `total_assignments=34724`,
`gpC_median=1.0`, `sort_key=prob_gaussian` (the deprecated prob-sorted config).
Same GT (n_gt=20441 both). So the 0.985↔0.992 gap is two assignment vintages, not
a GT-load difference and not a conversion error: the OB pipeline, which regenerates
pgmm from MEX with the current config, is self-consistent and 0.992 is its correct
Tier-1. (Verify end-to-end once the Papalexi lineage is wired: fresh pgmm → 0.992.)

## Full plan — validated + deterministic ✅ (2026-08-13)

`benchmark.yaml` (conda backend) — `ob validate plan` passes; `ob run --dry`
generates **135 rules deterministically** (10 modules; 2 data × 8 methods = 16
assignment lineages × 3 metric stages [7 metric values] + 4 collectors). See the
DAG-stability section below.

**Difficulty subsystem — RESOLVED (Phase-0 builder found).** The per-cell table
builder is `03_scripts/difficulty_phase0_build_table.py`; its logic is copied
verbatim into the data importer (`data.difficulty_table`) and reproduces
`cell_difficulty_ham.tsv` with **0 mismatches**. All strat metrics + collectors
consume it and match the archive exactly (rows above).

## Not executed here (logic vendored verbatim + dry-run clean; GEX/SVI-heavy)

| Item | Why not run | Gate for the detailed Git test |
|---|---|---|
| crispat pgmm/2beta assignments | Pyro SVI ≈ hours | md5 `f8294660`/`9cf2b649` **or** tolerance (stochastic; ≥99.5% top-1, ties only) |
| fishash assignments | needs R env | md5 `c8e2af96` |
| pgmm map_e2, umi t5/t10 | fast but not run | md5 `25f3221b`/`35498c35`; `08_umi_crispat/*/t{5,10}` |
| discovery | ≈160 GB HVG | Discovery/FPR vs `12_kd_efficiency/.../discovery/*` |
| Phase-2 delta-KD | GEX-heavy; not a plan stage (see below) | vs `_phase2_delta_kd.json` |
| Papalexi lineage | not wired | fresh pgmm → tier1 rec 0.992 (canonical) |

---

## DAG stability (2026-08-13)

`ob run --dry` generates **135 rules deterministically** — identical output at
`PYTHONHASHSEED` 0/1/2 and across repeated runs.

Two OB 0.6.0 DAG-builder nondeterminism traps were found and worked around:
1. **Multiple modules in one stage sharing an output id** → metric fan-out over
   lineages collapsed ~half the runs. Fixed by splitting `metrics` into three
   single-module stages (`metrics_gt` / `metrics_gex` / `metrics_discovery`).
2. **`requires:` + 3-way branch join** (`difficulty_validation`: difficulty_table +
   assignments + gex) → collapsed the whole plan's fan-out nondeterministically.
   Removed from the runnable plan; Phase-2 logic stays in the difficulty module
   (`--phase validate`) for standalone use.

Both are OB-planner issues, not defects in the vendored logic. The benchmark's
science is verified identical (above); these were purely about DAG generation.

# guide-assignment

Omnibenchmark for Perturb-seq **guide assignment**: 8 methods evaluated against a
tiered set of metrics (ground-truth accuracy, knockdown / discovery, construct-set,
mismatch) plus cell-level difficulty stratification.

```
data ─► guide_assignment (8 methods) ─► metrics_gt / metrics_gex / metrics_discovery
 │                                       + collectors: jaccard, strat_jaccard, extraction_shift, mismatch
 └── data.difficulty_table (per-cell, method-independent)
```

## Repositories

| Repo | Role |
|---|---|
| [guide-assignment](https://github.com/yunzhe-liu/guide-assignment) | benchmark plan + docs |
| [guide-assignment-data](https://github.com/yunzhe-liu/guide-assignment-data) | data importer + difficulty table |
| [guide-assignment-pgmm](https://github.com/yunzhe-liu/guide-assignment-pgmm) | PGMM-EM |
| [guide-assignment-umi](https://github.com/yunzhe-liu/guide-assignment-umi) | UMI threshold |
| [guide-assignment-crispat](https://github.com/yunzhe-liu/guide-assignment-crispat) | crispat |
| [guide-assignment-fishash](https://github.com/yunzhe-liu/guide-assignment-fishash) | fishash (R) |
| [guide-assignment-metrics](https://github.com/yunzhe-liu/guide-assignment-metrics) | metrics + difficulty + collectors |

## Run

```bash
ob validate plan benchmark.yaml
ob run benchmark.yaml --cores N
```

## Methods & metrics

See `METHODS.md`. Method selection is by module + `parameters`; metric selection is
by the `--metric` parameter value.

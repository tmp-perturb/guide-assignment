# Spec Alignment — relative to the `omni-perturb` reference architecture

**Purpose.** This benchmark is **standalone** — it is not merged with, and does not
depend on, the group's `omni-perturb` benchmark. This document (a) positions our
benchmark against `omni-perturb` as the *format reference*, and (b) lays out how we
conform to its conventions (data-container flexibility, standardized assignment
schema, tidy metric output) **without changing our methods or metrics**. The goal is
to demonstrate that we can produce an Omnibenchmark that meets the group's spec —
"deeper science **and** spec-compliant."

> Not a fusion plan. Our methods, metrics, difficulty stratification, and parity
> guarantees stay exactly as they are; only the outer I/O *format* is adapted.

---

## 1. The reference: `omni-perturb` (observed 2026-08)

Org `github.com/omni-perturb` — same Omnibenchmark skeleton as ours:

| Repo | Role |
|---|---|
| `plan` | the benchmark.yaml |
| `count-data` | data importer (fetches pre-aligned scPerturb-seq counts) |
| `crispat`, `kaichi` | method modules (kaichi bundles umi/max/ratio/poisson_gauss/beta2/beta3/neg_binomial) |
| `metrics` | metrics module |

Its plan (conda backend, one env per module):

```
data (count-data)            guide_assignment                 metrics
 dataset_name:               crispat: method=pgmm             → metrics.parquet
  [K562_essential, rpe1]     kaichi:  models=[umi,max,ratio,     + metrics.tsv
 → rawcounts.h5mu                     poisson_gauss,beta2,beta3,neg_binomial]
                             → assignments.h5ad
```

Format conventions worth adopting:
- **Data container:** `rawcounts.h5mu` (MuData, multimodal) → assignment output
  `assignments.h5ad`.
- **Standardized assignment schema (h5ad):** `obs["is_unassigned"]`,
  `obs["is_multi_infected"]`, `obs["target_gene"]`; `layers["assigned"]`; `X` = UMI.
- **Tidy long-format metrics:** `metrics.parquet` + `metrics.tsv`, columns
  `dataset, metric, submetric, value, n`. Metrics: coverage, umi,
  perturbation_coverage, rna_knockdown (largely ground-truth-free).
- **Dataset provenance:** a `dataset_name` parameter + a fetch step.

---

## 2. Connections & differences

**Connections (same paradigm — interoperable in principle):** identical skeleton
`data → guide_assignment → metrics`, conda backend, one env per module, GitHub
module repos pinned by commit, method-as-module + parameter sweep, both wrap
crispat, both use Replogle K562.

**Differences:**

| Axis | `omni-perturb` (reference) | Ours |
|---|---|---|
| Data container | h5mu (raw) + h5ad (assignment) | MEX trio + assignment CSV |
| Assignment schema | h5ad: `is_unassigned/is_multi_infected/target_gene` + `layers[assigned]` | CSV: `cell,gRNA,UMI_counts[,prob/pct/log_pval]` (method-native) |
| Data source | fetch pre-aligned scPerturb (K562_essential, rpe1) | our own extraction MEX (ham/simpleaf) + GT/GEX |
| Methods | crispat(pgmm) + kaichi(7 models) | pgmm_em, umi t3/5/10, crispat pgmm/2beta, fishash |
| Metrics | small, mostly GT-free: coverage/umi/pert-coverage/rna-KD | tiered: **GT Tier-1 accuracy** + Tier-2 (KD/discovery/FPR/construct-set) + Tier-3 mismatch + **difficulty (7 phases)** + jaccard |
| Metric output | tidy parquet/tsv (dashboard-ready) | per-metric JSON |
| Scale | minimal (3 stages, 2 method modules) | 8 methods, difficulty, collectors, 151-rule deterministic DAG |

---

## 3. Where ours stands

Two different maturity axes — and they are complementary:

- **Scientific depth & rigor (ours is stronger):** ground-truth Tier-1 accuracy,
  Tier-2/3, difficulty stratification, cross-method collectors, and a **line-by-line
  parity ledger** (assignment md5 byte-identical; every metric 0-mismatch vs the
  original) with a **deterministic** DAG. See `CONSISTENCY_VALIDATION.md`.
- **Format standardization & ecosystem fit (the reference is ahead):** multimodal
  h5mu/h5ad containers, tidy parquet/tsv output, dataset provenance, dashboard
  integration.

**Adopting the reference's format conventions makes ours both deep and
spec-compliant** — that is the whole of the plan below.

---

## 4. Spec-alignment checklist (format only — science untouched)

| # | Reference convention | Our current | Alignment (format-only) |
|---|---|---|---|
| A | input `h5mu` (MuData) | MEX trio | a loader that ingests **h5mu / h5ad / MEX** interchangeably (data-format inclusiveness) |
| B | `assignments.h5ad` with `is_unassigned/is_multi_infected/target_gene` + `layers[assigned]` | method-native CSV | a **normalization export** CSV→standard h5ad (keep CSV to preserve md5 parity) |
| C | tidy `metrics.parquet` + `.tsv` (`dataset,metric,submetric,value,n`) | per-metric JSON | a **tidy exporter** (numbers already computed; just re-packaged); keep JSON |
| D | `dataset_name` param + fetch | local-path import | wrap our extraction source behind the same `dataset_name` convention |
| E | S3 storage + `ob dashboard --format bettr` | local `out/` | add a `storage:` block + dashboard export |

---

## 5. Plan (phased, not executed)

- **P1 — outer shell, lowest risk (do first):** C (tidy parquet/tsv exporter) + A
  (multi-format loader). **Does not touch the verified method/metric logic** — only
  input adaptation and output packaging.
- **P2:** B (standard `assignments.h5ad` normalization); build a CSV↔h5ad schema map;
  CSV still drives the parity gate.
- **P3:** D (dataset convention) + E (storage + dashboard) → a shareable, bettr-ready
  product.

Throughout, our 8 methods / tiered metrics / difficulty / collectors / parity remain
unchanged.

---

## 6. One-line summary (for the mentor)

> We have an independent guide-assignment Omnibenchmark that is **more complete than
> the reference implementation** (ground-truth tiered accuracy, difficulty
> stratification, cross-method collectors), **line-by-line parity-verified** against
> the original pipeline, with a **deterministic DAG** — and it can be emitted in the
> `omni-perturb` format (h5mu/h5ad containers, standardized assignment schema, tidy
> parquet/tsv metrics). In short: **deeper content + spec-compliant format.**

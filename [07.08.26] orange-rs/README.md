# Orange-RS

A visual data analysis tool built in Rust — place widgets on a canvas, connect them with wires, and the graph becomes an analysis pipeline.

**The pipeline is treated as a single query, not a chain of materialized tables.** This is the project's reason to exist.

> No affiliation with the University of Ljubljana's [Orange](https://orangedatamining.com/) or its authors — not endorsed, not sponsored. Zero lines of Orange code are used. Orange is GPL-3.0 Python; this is MIT Rust. The only shared idea is "connect widgets with wires."

---

## Why

Most node-based tools (Orange included) materialize an entire DataFrame at every widget. A 10-node chain creates 10 DataFrames in memory.

Orange-RS folds non-barrier widgets into a single `LazyFrame`. A `File → Select Columns (2/5) → Filter Rows` chain compiles to:

```
Csv SCAN [customers.csv]
PROJECT 2/5 COLUMNS
SELECTION: [(col("a")) > (0)]
```

Column selection and row filtering are **attached to the file scan node.** No intermediate steps. Inspect the compiled plan via the inspector's `Compiled query` tab; tests verify the plan text.

### Benchmarks

1,000,000 rows × 8 columns, release build:

| Pipeline | Time | vs eager |
|---|---|---|
| eager — collect at every node | 84.9 ms | — |
| lazy chain, CSV | 82.5 ms | 1.03× |
| lazy chain, Parquet | 6.2 ms | **13.6×** |

**CSV shows no time gain** — row-based formats must parse the entire file regardless of which columns are needed. The real wins are:

- **No intermediate materialization** — eager allocated 10 million cells; lazy allocated 0. Format-agnostic, and what makes eager pipelines run out of memory.
- **Columnar formats** — Parquet skips unneeded columns entirely.

Exports also avoid materialization. `Save Data` uses Polars' file sink to write rows as they arrive, cutting peak heap from 395 MB to 197 MB for a 6-million-row export.

---

## Widgets

| Category | Widgets |
|---|---|
| **Data** | File (CSV/Parquet/NDJSON), SQLite, Select Columns, Filter Rows, Merge, Append, Unique, Head, Save Data |
| **Transform** | Sort, Group By, Impute, Normalize, Formula, Discretize, Pivot, Train/Test Split, SQL |
| **Visualize** | Data Table, Scatter Plot, Distribution, Correlation Heatmap |
| **Model** | k-Means, PCA, Linear Regression, Logistic Regression, Decision Tree, Apply Model, Score, Confusion Matrix, Cross-Validate |

**Only Pivot, models, and brushing viewers are barriers.** Everything else folds into the lazy chain — Group By included. No matter how long the pipeline, it's still one query until a barrier is hit.

`SQL` is an escape hatch that remains lazy — Polars lowers SQL to the same `LazyFrame`, so column lists still propagate down to the file scan.

### Execution & selection propagation

The engine distinguishes two kinds of changes with different costs:
- **Parameter/connection edits** recompile the node and everything downstream.
- **Brushing** keeps the plan and only re-filters downstream.
- Moving a node triggers no re-execution.

Selection propagation uses **explicit edges** (the `Selected` output port on `Data Table` / `Scatter Plot` / `Distribution`, drawn in orange). Models also use typed edges (sky blue). Wrong port types are rejected on connection.

Execution and export run on a worker thread. Long pipelines populate node-by-node rather than all at once.

### Supervised learning

- `Learn from` restricts training rows; prediction runs on **all rows.** Without it, downstream `Score` gives in-sample scores (90.3% vs 79.9% in bundled examples).
- Rows with a null target are excluded from training and **predicted as null.**
- Models are passed via the `Model` port or saved to file with `Save model...`. Files bundle the model object, standardization stats, and feature names — re-standardizing with new data or matching columns by position yields confidently wrong answers.
- `Cross-Validate` reports per-fold scores with **mean and standard deviation** — 0.8 accuracy at ±0.01 means something very different from ±0.15.

---

## Running

```bash
cargo run --release
```

Menu → Open to load example workflows from `sample/`.

| Example | Nodes | What it shows |
|---|---|---|
| `customer-segmentation.orsw` | 8 | Shortest path: impute → scale → k-Means → scatter, brushing propagates downstream |
| `regional-benchmark.orsw` | 9 | **Rejoining** aggregates to original rows (`Group By` → `Merge`). Builds a spend index vs. regional average |
| `holdout-regression.orsw` | 19 | **Honest holdout.** Train on `Fit on` subset, score on test, compare with train score side-by-side. Top/bottom 20 residuals appended to see where the model breaks |
| `segment-classifier.orsw` | 13 | 3-class classification, two-model comparison + confusion matrix + **stratified 5-fold** (mean ± std). Trained tree crosses via `Model` port to `Apply Model` for full-population labeling |
| `cluster-profile.orsw` | 11 | Unsupervised. z-score → k-Means (k=4) → PCA 2D projection. Cluster means table alongside, because clusters mean nothing without profiles |
| `data-quality-audit.orsw` | 13 | Dedup → `SQL` missing flags → per-cell missing aggregates / age×region pivot / cleaned export |

All six examples use **every widget in the palette at least once** (verified by `every_shipped_example_runs_end_to_end` and `the_shipped_examples_cover_the_whole_palette`).

`sample/customers.csv` is **synthetic data** (540 rows) generated for this repository — no real people, no personally identifiable columns. One property to keep in mind: **`segment` is determined by `region`** (one segment per region, 180 rows each), so grouping by both gives 3 cells, not 9.

### Key operations

| | |
|---|---|
| Add widget | Palette button or right-click canvas |
| Connect / disconnect | Drag output port → input port / click input port |
| View | Double-click node (multiple simultaneously) |
| Pan / zoom | Drag background / scroll |
| File | Ctrl+N / O / S / Shift+S |
| Edit | Ctrl+Z / Y · Ctrl+C / V / D · Delete |
| Select | Ctrl+drag (rect) · Ctrl+click (toggle) · Ctrl+A |
| Layout | Ctrl+L — selected subset when multiple nodes are selected |
| Group | Ctrl+G / Ctrl+Shift+G, double-click group box to collapse |

**Relative paths in workflows are relative to the workflow file's folder**, not the process working directory — so workflows and data can be shared as a folder.

**Groups are cosmetic only.** `compile.rs` and `engine.rs` never look at groups.

Only `recent.json` and crash logs are written to `%APPDATA%\Orange-RS`. No network access.

---

## Architecture

```
graph.rs      Nodes/ports/edges, cycle detection, topological sort — pure IR, no data
widgets.rs    Widget catalog: parameters, port signatures, is_barrier / emits_selection
compile.rs    Graph → LazyFrame lowering. Never collects
exec.rs       Pivot (group_by + conditional aggregation)
ml.rs         linfa integration — non-finite guards, row ordering preserved
engine.rs     2-tier execution, worker thread, per-node state and dirty sets
canvas.rs     Node editor (returns actions, doesn't mutate graph directly)
inspector.rs  Parameter editing
views.rs      Viewers and brushing, derived data caching
workflow.rs   .orsw serialization (version check, load-time validation, relative path rebasing)
app.rs        Panel assembly and action dispatch
icon.rs / recent.rs / crash.rs / heap.rs
```

`.orsw` is human-editable JSON. On load, **UI-enforced invariants are applied identically to the file** (unique node IDs, port range, one edge per input port, acyclic, group integrity). UTF-8 BOM is skipped — failing to parse the 3 bytes Windows editors add when you hand-edit would make the workflow appear broken to the reader.

---

## Verification

```bash
cargo test
cargo clippy --all-targets -- -D warnings
```

391 tests + 13 benchmarks (pipeline timing, export memory, editor benches × 9, example table reports), 0 clippy warnings.

Numbers in this document are verified by `packaging/verify-doc-numbers.ps1` against actual runs — because prose numbers are the only claims in the repository that neither the compiler, tests, nor linter reads.

Additional harnesses:

| Script | What it checks |
|---|---|
| `revert-check.ps1` | Deliberately breaks one invariant at a time and verifies the corresponding test **actually fails** — 62 cases |
| `verify-resources.ps1` | 3 icon sizes and version string embedded **byte-for-byte** in the exe, and nothing else |
| `smoke-gui.ps1` | 3 pixels of the actual window icon |
| `verify-installer.ps1` | Silent install → contents → shortcut → launch → **in-use overwrite denied** → silent uninstall, 0 remnants |
| `verify-advisories.ps1` | `cargo audit`, and `NOTICES.md` still describes the actual deploy graph |

`render_tests` draw one frame without a window or GPU, testing every widget's inspector and viewer against real data (empty frames, single rows, all-null, extreme values, error states).

**How we got here, what was measured and what got overturned, is in [`docs/ENGINEERING.md`](docs/ENGINEERING.md)** — why colors were chosen via color-vision simulation rather than by eye, five rounds of editor performance measurement, and the things this document got wrong about itself along the way.


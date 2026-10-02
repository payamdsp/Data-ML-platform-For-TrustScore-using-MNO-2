# Notebooks

There are two tracks in this directory and they are not alternatives to each
other. Read this page before opening anything, because opening the wrong track
first is the difference between an hour and a day.

| Track | Directory | Imports `trust_score_05`? | Needs a cluster? | Read it when |
|---|---|---|---|---|
| Standalone | `0.0-standalone/` | No | No | You are new, or you want to change the pipeline's logic |
| Package | `1.0-exploration/`, `2.0-features/`, `3.0-modeling/` | Yes | Yes, for the real data paths | You want to run the pipeline on real data |

## If you are new here, start with the standalone track

`0.0-standalone/` is five notebooks that implement the whole pipeline —
synthetic data, exploration, feature selection, preprocessing, training,
evaluation, reporting — in plain `numpy`, `pandas` and `scikit-learn`, with no
import of this package anywhere in them.

That is deliberate. The package is correct and it is also opaque: a working
sweep is one call to `run_sweep(...)`, and reading that call teaches you
nothing about what a sweep *is*. The standalone notebooks spell every step out
longhand so you can see the shape of the thing, break it, and watch what
breaks. They are the documentation that runs.

Run them strictly in order. Each one writes its outputs to a run directory and
the next one asserts that the previous one finished:

```
00-setup-and-data.ipynb        ->  features_*.parquet, labels, manifest.json
01-explore-and-validate.ipynb  ->  schema.json, quality_report.json
02-select-features.ipynb       ->  ranked_features.csv, selected_features.json
03-train-and-score.ipynb       ->  models/*.joblib, scores.parquet, metrics.csv
04-compare-and-report.ipynb    ->  leaderboard.csv, figures/, report.md
```

The run directory defaults to `./_run` and is overridable with the `TS05_RUN_DIR`
environment variable. It is ordinary files on disk, nothing else, which is what
makes the track standalone in the useful sense: you can run `03` tomorrow, on a
different machine, as long as the run directory came with you.

The synthetic data in `00` is not random noise with a label bolted on. Fraud is
planted as a *conjunction* of conditions — a recent device change and a recent
SIM change and an EnStream identity burst and a short tenure — so that no single
threshold separates the classes and feature interaction actually matters. The
frame also carries five deliberate traps: pure noise columns, a 95%-null column,
a zero-variance column, a numeric customer identifier, and a straightforward
label leak. Notebook `02` ends with an assertion that all five were caught. If
you change the selection logic and that assertion fires, you broke something
real; the assertion exists so that you find out then, rather than from a model
that scores 0.99 and means nothing.

## The package track is for real data

`1.0-exploration/` through `3.0-modeling/` call the library. They are the ones
to use when you have cluster access and want answers about production data.

```
1.1-schema-discovery.ipynb      what is actually in the tables
1.2-population-and-labels.ipynb who is in scope, which of them are fraud
2.1-build-feature-matrix.ipynb  run the five feature stages
2.2-feature-selection.ipynb     rank and cut the feature set
3.1-preprocess-and-fit.ipynb    impute, scale, fit one model
3.2-sweep.ipynb                 fit the whole grid
3.3-evaluate-and-compare.ipynb  top-k metrics, scenario coverage, champion
3.4-drift.ipynb                 PSI and score-distribution monitoring
```

They are also ordered, but more loosely: `2.1` needs the paths confirmed in
`1.1`, `2.2` needs the matrix from `2.1`, and everything in `3.0` needs the
selected feature list. Within `3.0` you can go straight to `3.2` if you do not
care to fit a single model by hand first.

Two things about running them:

- **Every cell that writes anything is gated** behind a flag set to `False` at
  the top of that cell (`PUBLISH`, `SAVE`, `RUN`). Flip the flag deliberately.
  Nothing in these notebooks overwrites a table because you pressed
  shift-enter through the file.
- **`2.1` has a synthetic fallback.** If the configured lineage path is not
  reachable it builds a small frame in memory instead, so the notebook is
  readable and runnable without cluster credentials. The fallback is loud about
  being a fallback; do not report numbers from it.

## Two rules that cost people a day each

**Read the right grain.** `build_scope_snapshots` returns a dict. The `"base"`
entry is one row per lineage interval and carries every entity column. Every
other entry is aggregated to one row per entity and only the grouping keys
survive. A frame with the right *columns* is not a frame at the right *grain*,
and Spark will not tell you the difference — you get a join that silently
multiplies rows. This mistake has been made twice in this codebase; see §4.4 of
`trust_score_05/features/docs/feature_library_findings.md`.

**`LABEL_LEAKAGE_COLUMNS` is not optional.** The exclusion list in the feature
selection config is the only thing standing between you and a model that has
learned to read the fraud case file. Dropping it makes the metrics go up. That
is the tell, not the reward.

## Regenerating

These notebooks were written by a generator script rather than by hand, because
notebook JSON is unreviewable in a diff. The generator is not a build step and
nothing depends on it; it was a convenience for the initial write. Editing a
notebook directly is fine and expected — it just means the generator is stale
from that point on.

## Inputs, outputs, and operating boundary

The standalone track creates synthetic feature/label data locally and writes
intermediate Parquet/CSV/JSON, model bundles, score tables, metrics, plots, and
a Markdown report beneath `TS05_RUN_DIR` (default `./_run`). It does not read
production data or import the package. The package track reads configured
lineage/label locations and writes discovery, feature-selection, sweep,
evaluation, champion, scoring, and drift artifacts to configured local or S3
roots. Table and URI contracts come from `sagemaker/conf/ml/`; notebook cells
show the input and output paths used by each stage.

The package functions live under `trust_score_05/features/` (lineage snapshots,
event normalization, feature assembly) and `trust_score_05/ml/` (discovery,
feature ranking, preprocessing, model sweep/evaluation, and drift). Deployable
Processing/Training entry points are under `sagemaker/jobs/`; notebooks are
interactive drivers and exploration material, not scheduled production jobs
by themselves.

## Safe run procedure

For a local learning run, install dependencies described in the SageMaker
[getting-started guide](../docs/docs/getting-started.md), set an isolated
output directory, and execute `00` through `04` in order:

```bash
cd Project_Lotus/project_lotus/sagemaker
export TS05_RUN_DIR=./_run
jupyter lab notebooks/0.0-standalone/
```

For the package track, first complete the configuration and access checks in
the getting-started guide, then follow `1.1`/`1.2`, `2.1`/`2.2`, and `3.1`-
`3.4` in dependency order. Do not enable a cell's `PUBLISH`, `SAVE`, or `RUN`
flag until its destination, run ID, and overwrite behavior have been reviewed.
The synthetic fallback in `2.1` is a development fallback, not production
evidence; do not report its output as real-data results.

## Phase 1/2 relevance

The notebooks support Phase 1 exploration, feature/input validation, model
evaluation, and controlled batch workflow rehearsals, and Phase 2 handover of
repeatable training/scoring stages. They do not on their own provide vendor
scheduling, source-system incremental extraction, production alerting, or proof
of the vendor brief's acceptance criteria. Track operational milestones
outside notebook outputs.

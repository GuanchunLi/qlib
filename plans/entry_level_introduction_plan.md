# Entry-Level Introduction Tutorial Notebook Plan

## Summary

- Replace the earlier draft with a repo-aligned implementation plan.
- Add a new beginner notebook at `examples/tutorial/entry_level_introduction.ipynb`.
- Keep the scope notebook-only in this pass. Do not update `README.md`, docs indexes, or other discovery links.
- Require `pyqlib[analysis]` so the report cells can render Plotly-based analysis in Jupyter.
- Ensure every persisted notebook artifact uses a `demo_` prefix.

## Notebook Content

The notebook should follow the teaching cadence of `examples/tutorial/detailed_workflow.ipynb` and the workflow structure of `examples/workflow_by_code.ipynb`, but keep the story focused on one beginner-friendly happy path:

1. initialize Qlib and resolve a usable CN daily data path
2. inspect the trading calendar, dynamic CSI300 membership, and one stock's raw features
3. explain price adjustment with `$factor`
4. show one expression-engine example
5. introduce `Alpha158` and inspect a small feature and label slice
6. build a `DatasetH` with the standard train/valid/test time split
7. train the standard LightGBM benchmark configuration
8. generate predictions, signal analysis, and backtest analysis
9. load the saved artifacts and visualize the results
10. point to the benchmark YAML and `qrun` as the reproducible next step

## Artifact Naming

Do not use the stock `SignalRecord`, `SigAnaRecord`, and `PortAnaRecord` classes directly for final saved outputs, because they hardcode non-demo filenames.

Instead, define notebook-local demo-prefixed subclasses that preserve Qlib behavior while changing the persisted artifact names:

- `demo_trained_model.pkl`
- `demo_pred.pkl`
- `demo_label.pkl`
- `sig_analysis/demo_ic.pkl`
- `sig_analysis/demo_ric.pkl`
- `portfolio_analysis/demo_report_normal_1day.pkl`
- `portfolio_analysis/demo_positions_normal_1day.pkl`
- `portfolio_analysis/demo_port_analysis_1day.pkl`

Long-short and indicator-analysis artifacts are intentionally out of scope for this first notebook to keep the run simpler and the artifact set small.

## Implementation Notes

- Keep the notebook local-Jupyter oriented:
  - no Colab badge
  - no in-notebook package installation
  - opening prose should state the `pyqlib[analysis]` requirement
- Prefer repo-local `qlib_data/cn_data` when it already exists, then fall back to `~/.qlib/qlib_data/cn_data`, and only download data if neither path is ready.
- Keep the LightGBM task aligned with `examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml`.
- All later analysis cells must load only the demo-prefixed artifacts.
- Plot rendering stays in-memory only. The notebook should not export chart image files.
- Commit the notebook with cleared outputs.

## Test Plan

- Validate the notebook JSON opens cleanly.
- Execute the notebook end to end in the local `qlib` environment with `pyqlib[analysis]`.
- Confirm the bootstrap logic works with the existing repo-local CN dataset.
- Confirm the recorder only contains the expected demo-prefixed saved results for this notebook flow.
- Confirm the downstream analysis cells load the demo-prefixed artifacts without falling back to Qlib's default filenames.
- Confirm the notebook does not rerun the workflow unnecessarily after the training section.

## Assumptions

- Final notebook path is `examples/tutorial/entry_level_introduction.ipynb`.
- Scope remains notebook-only.
- The example remains on CN daily data, CSI300, SH000300, LightGBM, and `Alpha158`.

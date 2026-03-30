# Windows Qlib Local Demo Run Plan

## Summary

This plan sets up a local Windows Qlib development environment for this repo, keeps the demo dataset inside the repo at `e:/Projects/qlib/qlib_data/cn_data`, and validates the setup with:

1. a lightweight data-access sanity script
2. one full LightGBM workflow run

The preferred manual shell flow uses `conda activate qlib`. The equivalent automation-safe flow uses `conda run -n qlib ...` for each command.

## Preflight

Current machine state before execution:

- `conda` is installed
- no dedicated `qlib` conda environment exists yet
- no repo-local `qlib_data/` directory exists yet
- git worktree is clean

## Command Style

Use one of these command styles consistently:

- Manual interactive shell:

```powershell
conda activate qlib
```

Use this when you are working in one terminal session and want shorter commands.

- Non-interactive or automation:

```powershell
conda run -n qlib <command>
```

Use this when commands are run individually, from automation, or from tools that do not preserve shell session state.

## Phase 1: Create And Verify The Environment

### 1.1 Create the environment

```powershell
conda create -n qlib python=3.11 -y
```

### 1.2 Install Qlib from source

Manual interactive shell:

```powershell
conda activate qlib
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -m pip install numba
```

Automation-safe form:

```powershell
conda run -n qlib python -m pip install --upgrade pip
conda run -n qlib python -m pip install -e ".[dev]"
conda run -n qlib python -m pip install numba
```

Why `numba` is explicit here:

- it is not part of the base package dependency list in `pyproject.toml`
- the repo's CI installs it before running the benchmark workflow on source builds

### 1.3 Verify imports

Manual interactive shell:

```powershell
python -c "import qlib, lightgbm; print(qlib.__version__)"
```

Automation-safe form:

```powershell
conda run -n qlib python -c "import qlib, lightgbm; print(qlib.__version__)"
```

If the import fails with a compiled extension error such as `qlib.data._libs.rolling` or `qlib.data._libs.expanding`, rebuild in place and retry:

Manual interactive shell:

```powershell
python setup.py build_ext --inplace
python -c "import qlib, lightgbm; print(qlib.__version__)"
```

Automation-safe form:

```powershell
conda run -n qlib python setup.py build_ext --inplace
conda run -n qlib python -c "import qlib, lightgbm; print(qlib.__version__)"
```

## Phase 2: Download And Check Repo-Local Data

Target dataset path:

```text
e:/Projects/qlib/qlib_data/cn_data
```

### 2.1 Download CN daily data

Primary path: use the built-in downloader because it matches the current repo code and CI flow.

Manual interactive shell:

```powershell
python -m qlib.cli.data qlib_data --target_dir e:/Projects/qlib/qlib_data/cn_data --interval 1d --region cn
```

Automation-safe form:

```powershell
conda run -n qlib python -m qlib.cli.data qlib_data --target_dir e:/Projects/qlib/qlib_data/cn_data --interval 1d --region cn
```

If `qlib_data/cn_data` already exists on a rerun, prefer the skip-safe form instead of re-downloading:

```powershell
python -m qlib.cli.data qlib_data --target_dir e:/Projects/qlib/qlib_data/cn_data --interval 1d --region cn --exists_skip True
```

or:

```powershell
conda run -n qlib python -m qlib.cli.data qlib_data --target_dir e:/Projects/qlib/qlib_data/cn_data --interval 1d --region cn --exists_skip True
```

### 2.2 Fallback if the built-in download fails

If the built-in download returns 404, times out, or otherwise fails, use the community dataset linked in the repo `README.md` and extract it into:

```text
e:/Projects/qlib/qlib_data/cn_data
```

Expected structure after download or extraction:

```text
e:/Projects/qlib/qlib_data/
  cn_data/
    calendars/
    features/
    instruments/
    ...
```

### 2.3 Run data health checks

Manual interactive shell:

```powershell
python scripts/check_data_health.py check_data --qlib_dir e:/Projects/qlib/qlib_data/cn_data
```

Automation-safe form:

```powershell
conda run -n qlib python scripts/check_data_health.py check_data --qlib_dir e:/Projects/qlib/qlib_data/cn_data
```

Success criteria:

- Qlib initializes against the repo-local data path
- the script completes without crashing
- warnings are acceptable findings to review, but path or initialization failures are blocking

## Phase 3: Run Demo Validations

### 3.1 Create and run a temporary sanity script

Create a temporary file at:

```text
plans/_tmp_demo_sanity.py
```

Script content:

```python
import qlib
from qlib.constant import REG_CN
from qlib.data import D


if __name__ == "__main__":
    qlib.init(provider_uri=r"e:/Projects/qlib/qlib_data/cn_data", region=REG_CN)

    calendar = D.calendar(start_time="2020-01-01", end_time="2020-12-31")
    print(f"Trading days in 2020: {len(calendar)}")

    fields = ["$close", "$volume", "Ref($close, 1)", "Mean($close, 5)"]
    df = D.features(["SH600000"], fields, start_time="2020-01-01", end_time="2020-12-31")
    print(df.head(10))
```

Run it:

Manual interactive shell:

```powershell
python plans/_tmp_demo_sanity.py
```

Automation-safe form:

```powershell
conda run -n qlib python plans/_tmp_demo_sanity.py
```

Success criteria:

- prints a non-empty 2020 trading calendar count
- prints a non-empty feature DataFrame for `SH600000`

Delete the temporary script after the check completes.

### 3.2 Create and run a repo-local workflow config

Create a temporary file at:

```text
plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml
```

Source:

```text
examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

Change only this field:

```yaml
qlib_init:
    provider_uri: "e:/Projects/qlib/qlib_data/cn_data"
```

Keep the rest of the config unchanged.

Run the workflow from the repo root:

Manual interactive shell:

```powershell
python -m qlib.cli.run plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml
```

Automation-safe form:

```powershell
conda run -n qlib python -m qlib.cli.run plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml
```

Success criteria:

- the workflow runs to completion without traceback
- training and backtest output is printed
- a run is created under `mlruns/`

Delete the temporary workflow config after the run completes.

## Phase 4: Post-Run State

Keep these outputs after execution:

- `e:/Projects/qlib/qlib_data/cn_data`
- `e:/Projects/qlib/mlruns`

Delete only the temporary helper files:

- `plans/_tmp_demo_sanity.py`
- `plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml`

## Expected Blocking Conditions

These are blocking failures for this plan:

- environment creation fails
- editable install fails
- `import qlib` still fails after the rebuild fallback
- data download and fallback download both fail
- repo-local data cannot initialize with Qlib
- the sanity script crashes or returns empty results
- the LightGBM workflow crashes before producing a run under `mlruns/`

## Notes For This Machine

- Python 3.11 is used because it is already present locally and is covered by the repo's Windows CI.
- This plan intentionally keeps data inside the repo instead of `~/.qlib/...`.
- `workflow_by_code.py`, notebooks, US data, RL flows, and multi-model runs are out of scope for this pass.
- Package installation and dataset download require network-enabled execution outside the current sandbox.

## Execution Results: 2026-03-30

Execution status on this machine:

- `conda create -n qlib python=3.11 -y` succeeded
- editable install succeeded with `conda run -n qlib python -m pip install -e ".[dev]"`
- `numba` was installed successfully
- `conda run -n qlib python -c "import qlib, lightgbm; print(qlib.__version__)"` succeeded and printed `0.9.8.dev28`
- built-in CN daily data download succeeded; no fallback source was needed
- repo-local data now exists under `e:/Projects/qlib/qlib_data/cn_data`
- the downloaded archive `20260330120555_qlib_data_cn_1d_latest.zip` was left in `qlib_data/cn_data`

Observed data health check outcome:

- Qlib initialized successfully against `e:/Projects/qlib/qlib_data/cn_data`
- the script completed without crashing
- the script reported missing-data warnings for many instruments
- the script reported large-step-change warnings in OHLCV data
- required columns, factor presence, and lowercase feature directory checks passed

Observed sanity-script outcome:

- `plans/_tmp_demo_sanity.py` ran successfully
- `Trading days in 2020: 180`
- feature lookup for `SH600000` returned a non-empty DataFrame

Observed workflow outcome:

- `conda run -n qlib python -m qlib.cli.run plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml` completed successfully
- best LightGBM iteration: `18`
- reported metrics:
  - `IC = 0.04680587323833807`
  - `ICIR = 0.3815683918932705`
  - `Rank IC = 0.049049290457489736`
  - `Rank ICIR = 0.406748756941287`
- reported excess return with cost:
  - `annualized_return = 0.080654`
  - `information_ratio = 0.914486`
  - `max_drawdown = -0.086083`
- a workflow run was created under `e:/Projects/qlib/mlruns`

Cleanup completed:

- temporary helper files were removed:
  - `plans/_tmp_demo_sanity.py`
  - `plans/_tmp_workflow_config_lightgbm_Alpha158.local.yaml`
- retained outputs:
  - `e:/Projects/qlib/qlib_data/cn_data`
  - `e:/Projects/qlib/mlruns`

# Plan: QLib Environment Setup & Demo Run (Windows + Conda)

## Goal

Set up a working QLib development environment on a local Windows machine using a fresh conda environment, download the required datasets into this repo's directory structure, and run a few demos to verify everything works.

---

## Phase 1: Create Conda Environment & Install QLib

### Step 1.1 — Create a new conda environment

```bash
conda create -n qlib python=3.10 -y
conda activate qlib
```

> Python 3.10 is recommended for broad compatibility. QLib supports 3.8–3.12.

### Step 1.2 — Install build dependencies

```bash
pip install numpy>=1.24.0 cython setuptools setuptools-scm
```

These are needed to compile QLib's Cython extensions (`rolling.pyx`, `expanding.pyx`).

### Step 1.3 — Install QLib from source (editable mode)

Since we're working from the cloned repo:

```bash
cd e:\Projects\qlib
pip install -e .[dev]
```

This installs QLib in development mode with test dependencies (`pytest`, `statsmodels`).

### Step 1.4 — Verify installation

```bash
python -c "import qlib; print(qlib.__version__)"
```

If this prints the version without errors, installation succeeded. If you see an error about `qlib.data._libs.rolling`, run:

```bash
python setup.py build_ext --inplace
```

---

## Phase 2: Download Datasets

### Step 2.1 — Download CN daily data

Store data inside the repo to keep everything self-contained:

```bash
python -m qlib.tests.data qlib_data --target_dir ./qlib_data/cn_data --region cn
```

If the official source is down, use the community alternative:

```bash
# Download from: https://github.com/chenditc/investment_data/releases
# Extract the qlib_bin.tar.gz into ./qlib_data/cn_data/
```

### Step 2.2 — (Optional) Download US daily data

```bash
python -m qlib.tests.data qlib_data --target_dir ./qlib_data/us_data --region us
```

### Step 2.3 — Verify data health

```bash
python scripts/check_data_health.py check_data --qlib_dir ./qlib_data/cn_data
```

### Expected data directory structure

```
e:\Projects\qlib\qlib_data\
  cn_data\
    calendars\
    features\
    instruments\
    ...
  us_data\        (optional)
    ...
```

> Add `qlib_data/` to `.gitignore` if not already there to avoid committing large binary data.

---

## Phase 3: Run Demo Scripts

All demos require wrapping in `if __name__ == "__main__":` on Windows due to the `spawn` multiprocessing model. The existing example scripts already handle this.

### Demo 1 — Minimal data access (sanity check)

Create and run a quick script to confirm data loading works:

```python
# save as demo_sanity.py in repo root
import qlib
from qlib.constant import REG_CN
from qlib.data import D

if __name__ == "__main__":
    qlib.init(provider_uri="./qlib_data/cn_data", region=REG_CN)

    # Check trading calendar
    calendar = D.calendar(start_time="2020-01-01", end_time="2020-12-31")
    print(f"Trading days in 2020: {len(calendar)}")

    # Fetch stock features
    fields = ["$close", "$volume", "Ref($close, 1)", "Mean($close, 5)"]
    df = D.features(["SH600000"], fields, start_time="2020-01-01", end_time="2020-12-31")
    print(df.head(10))
```

```bash
python demo_sanity.py
```

**Expected result**: prints ~244 trading days and a DataFrame with close/volume/derived features.

### Demo 2 — Full workflow (model training + backtest)

```bash
cd examples
python workflow_by_code.py
```

This runs the complete pipeline: data loading -> Alpha158 feature engineering -> LightGBM training -> prediction -> backtest -> analysis report.

**Expected result**: training logs, IC/ICIR metrics, and backtest return summary printed to console. Takes a few minutes.

### Demo 3 — YAML-based workflow via `qrun`

```bash
cd examples
qrun benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

This is the same workflow as Demo 2 but driven by a YAML config file, which is the standard way to run QLib experiments.

**Expected result**: same kind of output as Demo 2 — model metrics and backtest results.

### Demo 4 — (Optional) Jupyter notebook

```bash
cd examples
jupyter notebook workflow_by_code.ipynb
```

Walk through the notebook cells interactively for a step-by-step understanding of the workflow.

---

## Phase 4: Verify & Clean Up

### Step 4.1 — Confirm all demos ran without errors

Check that:
- [ ] `import qlib` works
- [ ] Data loads without file-not-found errors
- [ ] LightGBM model trains and produces predictions
- [ ] Backtest completes and prints return metrics

### Step 4.2 — Clean up temporary files

```bash
# Remove demo script if created
del demo_sanity.py

# Remove mlflow artifacts if not needed
rmdir /s /q mlruns
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `No module named 'qlib.data._libs.rolling'` | Run `python setup.py build_ext --inplace` |
| Data download hangs or fails | Use the community data source (chenditc/investment_data on GitHub) |
| `RuntimeError: freeze_support()` | Wrap code in `if __name__ == "__main__":` |
| Redis connection errors | Redis is optional — QLib works without it, just no caching |
| `pip install -e .` fails with missing headers | Ensure you're inside a conda env, not system Python |
| Import errors when running from repo root | Run scripts from `examples/` or another directory to avoid shadowing the `qlib` package |

---

## Notes

- **Memory**: 16GB RAM recommended. 8GB minimum.
- **Disk**: ~5GB free for data + models.
- **Redis**: Not required for local demos. Only needed for caching in production setups.
- **GPU**: Not needed for LightGBM demos. Only required for deep learning models (LSTM, Transformer, etc.) — install PyTorch separately if needed.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

**Qlib** is an open-source AI-oriented quantitative investment platform by Microsoft. It provides a complete pipeline for quantitative research: data processing → feature engineering → model training → backtesting → portfolio analysis.

## Build & Development Commands

```bash
# Install the package (editable dev mode) — prerequisite step
make install

# Full dev setup (install + lint tools + docs deps)
make dev

# Compile Cython extensions (rolling/expanding ops) without full install
make prerequisite

# Run all non-slow tests
cd tests && python -m pytest . -m "not slow" --durations=0

# Run a single test file
cd tests && python -m pytest path/to/test_file.py -m "not slow"

# Run a specific test function
cd tests && python -m pytest path/to/test_file.py::test_func -m "not slow"

# Run a workflow by config (integration test)
python qlib/cli/run.py examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml

# Download test data
python scripts/get_data.py qlib_data --name qlib_data_simple --target_dir ~/.qlib/qlib_data/cn_data --interval 1d --region cn

# Lint & type checking
make black        # Black format check (line length 120)
make pylint       # Pylint on qlib/ and scripts/
make flake8       # Flake8 on qlib/
make mypy         # Mypy type checking

# Build & package
python -m build --wheel   # Build wheel via setuptools
```

## Project Structure

### Core Package (`qlib/`)

- **`qlib/__init__.py`** — `qlib.init()` entry point; configures providers, caches, and logging
- **`qlib/config.py`** — Configuration system (`C` singleton, `QSettings` with pydantic-settings). Supports client/server modes with `provider_uri`, `region`, and logging config
- **`qlib/constant.py`** — Region constants (`REG_CN`, `REG_US`, `REG_TW`), time constants (`ONE_DAY`, `ONE_MIN`), `EPS` for numerical stability

### Data Pipeline (`qlib/data/`)

The data layer follows a **provider → cache → dataset** architecture:

- **`data.py`** — Data providers: `CalendarProvider`, `InstrumentProvider`, `FeatureProvider`, `ExpressionProvider`, `DatasetProvider`. Each has `Local*` and `Client*` variants for local files or remote server. `D` is the singleton entry point for expression calculation
- **`cache.py`** — Multi-level cache: `MemoryCalendarCache`, `DiskExpressionCache`, `DiskDatasetCache`, `DatasetURICache`. `H` is the singleton cache handler
- **`ops.py`** — Feature operators (expressions): element-wise (`Ref`, `Mean`, `Sum`, `Std`), pair-wise (`Sub`, `Add`, `Mul`, `Div`, `GT`, `LT`), rolling (`RSquare`, `Resi`, `Slope`, `Corr`, `Cov`), and more. Cython-accelerated rolling/expanding operations in `_libs/`
- **`base.py`** — Expression base classes: `Expression`, `Feature`, `PFeature`, `ExpressionOps`
- **`dataset/`** — Dataset construction:
  - **`handler.py`** — `DataHandlerABC` / `DataHandlerLP` classes: load raw data → process → fetch for training/inference
  - **`processor.py`** — Data processors (z-score, min-max, robust winsorization, etc.), chained in sequence
  - **`loader.py`** — Data loaders from various sources
  - **`__init__.py`** — `DatasetH` (segmented dataset) and `TSDatasetH`/`TSDataSampler` (time-series dataset with efficient indexing)
  - **`weight.py`** — `Reweighter` classes for sample weighting
- **`storage/`** — Storage backends (`FileCalendarStorage`, `FileInstrumentStorage`, `FileFeatureStorage`, etc.)
- **`inst_processor.py`** — Instrument processors for advanced instrument handling
- **`pit.py`** — Point-in-time data support
- **`filter.py`** — Data filtering utilities
- **`client.py`** — Client for remote qlib server

### Models (`qlib/model/`)

- **`base.py`** — `BaseModel` (abstract predict) and `Model` (learnable, with `fit()` + `predict()`)
- **`trainer.py`** — `task_train()` function for model training orchestration
- **`utils.py`** — Model utilities
- **`ens/`** — Ensemble methods
- **`interpret/`** — Model interpretation
- **`meta/`** — Meta learning
- **`riskmodel/`** — Risk model implementations
- **`qlib/contrib/model/`** — 30+ pre-built models:
  - GBDT family: `LightGBM`, `XGBoost`, `CatBoost`
  - Neural networks: LSTM, GRU, Transformer, GATs, TCN, SFM, TabNet, AdaRNN, Logistic, KRNN, HIST, Localformer, Sandwich, IGMTF, TCTS, plus time-series variants
  - `pytorch_nn.py` — Base PyTorch NN model class
  - `pytorch_utils.py` — Shared PyTorch utilities

### Backtesting (`qlib/backtest/`)

- **`backtest.py`** — `backtest_loop()` and `collect_data_loop()`: core simulation loop
- **`decision.py`** — Trade decision types
- **`executor.py`** — Order execution (simulated executor)
- **`exchange.py`** — Exchange abstraction for trading
- **`account.py`** — Account/position tracking
- **`report.py`** — Performance indicator calculation
- **`signal.py`** — Signal generation
- **`strategy.py`** — Strategy base classes (in `qlib/strategy/`)
- **`high_performance_ds.py`** — High-performance decision simulation
- **`profit_attribution.py`** — Profit attribution analysis
- **`position.py`** — Position management

### Workflow & Experiment Tracking (`qlib/workflow/`)

- **`expm.py`** — `ExpManager` (MLflow-based singleton experiment manager)
- **`exp.py`** — `Experiment` / `MLflowExperiment` classes
- **`recorder.py`** — `Recorder` for logging experiment runs
- **`record_temp.py`** — Template recorders: `SignalRecord`, `PortAnaRecord`, `SigAnaRecord`
- **`task/`** — Task management (`collect.py`, `gen.py`, `manage.py`, `utils.py`)
- **`online/`** — Online/real-time experiment updates (`manager.py`, `strategy.py`, `update.py`)

### CLI (`qlib/cli/`)

- **`run.py`** — `qrun` CLI entry point: runs a workflow from a YAML config file (model + dataset + backtest/portfolio analysis)

### Utilities (`qlib/utils/`)

- **`serial.py`** — `Serializable` base class with smart pickle (config-aware attribute serialization)
- **`mod.py`** — Module loading utilities
- **`paral.py`** — Parallel processing (`datetime_groupby_apply`, `ParallelExt`)
- **`time.py`** — Time/frequency utilities (`Freq`)
- **`data.py`** — Data utilities (robust_zscore, etc.)
- **`pickle_utils.py`** — `RestrictedUnpickler` for secure deserialization
- **`file.py`**, **`index_data.py`** — File/IO helpers

### Scripts (`scripts/`)

- **`get_data.py`** — Download qlib data or RL data
- **`dump_bin.py`** — Dump data to binary format
- **`dump_pit.py`** — Dump point-in-time data
- **`data_collector/`** — Data collection scripts
- **`check_data_health.py`** — Health checks for data integrity

### Examples (`examples/`)

- **`workflow_by_code.py`** / **`workflow_by_code.ipynb`** — End-to-end workflow example
- **`benchmarks/`** — Model benchmark configurations (YAML workflows)
- **`tutorial/`** — Tutorial notebooks

### Data Flow Summary

```
Raw Data (CSV/binary) 
  → DataLoader (load into DataHandler) 
    → Processors (z-score, fillna, etc.) 
      → DatasetH (segment into train/valid/test) 
        → TSDataSampler (time-series windows) 
          → Model.fit() / Model.predict()
            → Backtest (strategy → executor → exchange → account)
              → Report (indicators, portfolio metrics)
```

## Key Architecture Patterns

- **Serializable pattern**: Models, datasets, and handlers inherit from `Serializable`, which provides smart pickle behavior — attributes starting with `_` are excluded from serialization by default, config attributes are always excluded
- **Config-driven instantiation**: `init_instance_by_config(config_dict)` creates instances from dict configs with `class` and `kwargs` keys — used throughout the framework
- **Provider pattern**: Data sources follow a `BaseProvider` → `LocalProvider`/`ClientProvider` hierarchy, enabling seamless switching between local and remote data
- **Operator expression system**: Features are composed as expression trees (e.g., `Ref(Mean($close, 5), -1)`), evaluated lazily by the expression provider
- **Singleton pattern**: `D` (data), `H` (cache), `C` (config), `R` (experiment recorder) are global singletons

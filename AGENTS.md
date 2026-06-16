# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is
A single-notebook research project: `DNR_Diagnostic_PreTraining-2.ipynb`. It is a
pre-training diagnostic pipeline for **Distribution Network Reconfiguration (DNR)** on the
**IEEE 33-bus** feeder (physics-guided load sampler + exact BFS power-flow solver +
topology/loss analysis). There is no server, database, or web app — the "application" is
the notebook itself. There is no lint/test/build tooling in the repo.

### Required input data
The notebook reads `Data_33bus.xlsx` from the current working directory (`EXCEL_FILE` in
the config cell), expecting two sheets:
- `Linedata` with columns `Line Number`, `From bus`, `To bus`, `r`, `x` (37 rows: 32
  section + 5 tie lines).
- `Busdata` with columns `Bus No.`, `P_load`, `Q_load`, and a 4th unnamed Type column
  (33 rows, loads in kW/kvar).

This file is committed in the repo root. It was reconstructed from the notebook's own saved
outputs (total base load 3.715 MW), so it reproduces the original author's results.

### Dependency notes (non-obvious)
- Dependencies are installed into the system Python (`pip --break-system-packages`); there
  is no virtualenv. The update script reinstalls them on startup.
- `pandas` must stay **< 3.0**: pandas 3.0 made `DataFrame.to_hdf(...)` arguments
  keyword-only, which breaks the notebook's positional `to_hdf(path, 'key')` calls.
- `numba` 0.63.x requires `numpy` **< 2.4** (we use numpy 2.3.x).
- HDF5 I/O needs `tables` (PyTables); Excel I/O needs `openpyxl`.
- `torch` / `torch_geometric` are **optional** — the relevant cells are wrapped in
  `try/except` and are skipped (with a printed notice) when torch is absent. They are not
  installed by default.

### How to run it
From the repo root (cwd must contain `Data_33bus.xlsx`):

```
# Headless full run (writes dnr_diagnostic.h5 with keys: scenarios, summary, loss_weights)
python3 -m nbconvert --to notebook --execute --ExecutePreprocessor.timeout=1800 \
  --output /tmp/executed.ipynb DNR_Diagnostic_PreTraining-2.ipynb

# Or interactively
jupyter lab    # (binaries live in ~/.local/bin)
```

### Known pre-existing notebook issues (NOT environment problems)
- `compute_loss(...)` (and `distflow_loss_approx(...)`) are **commented out** in the BFS
  cell but `compute_loss` is still called, so a clean top-to-bottom run raises
  `NameError: name 'compute_loss' is not defined`. To run the full pipeline you must
  define/uncomment `compute_loss`. (The author's saved outputs show it was defined in their
  kernel session before being commented out.)
- The default/base-load BFS "sanity check" reports non-convergence (`Converged: False`,
  `solver returned None on base loads`). This matches the committed notebook's own saved
  output, i.e. it is pre-existing behavior, not caused by the environment or the input data.
  The 2,000-scenario diagnostic still runs and produces valid results + `dnr_diagnostic.h5`.

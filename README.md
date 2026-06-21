# DEA Solver (CRS)

A lightweight Data Envelopment Analysis (DEA) solver in Python for measuring the
relative efficiency of Decision Making Units (DMUs). It implements the
**Constant Returns to Scale (CRS)** model in both **input-oriented (IO)** and
**output-oriented (OO)** form, using a standard two-phase procedure, and exports
the full results to a structured Excel workbook.

## What it does

Given a set of DMUs, each consuming several inputs to produce several outputs, the
solver computes for every DMU:

- its **efficiency score** (`theta` for IO, `phi` for OO),
- the **dual values** (multiplier weights on each input and output),
- the **lambda values** (the peer / reference set defining the efficient frontier),
- the **input and output slacks**, and
- the **projected efficient targets** (`Xeff`, `Yeff`) each DMU should reach to become efficient.

Under CRS the two orientations give reciprocal scores (`theta_IO = 1 / phi_OO`),
so the set of efficient DMUs is identical; the slacks, lambdas and projection
targets differ because the projection direction differs.

## Method

For each DMU the solver runs a **two-phase LP**:

1. **Phase I** – optimize the radial efficiency score.
   - IO: minimize `theta` while inputs are scaled down (`sum_j lambda_j x_ij <= theta * x_io`) and outputs are held (`sum_j lambda_j y_rj >= y_ro`).
   - OO: maximize `phi` while outputs are scaled up (`sum_j lambda_j y_rj >= phi * y_ro`) and inputs are held (`sum_j lambda_j x_ij <= x_io`).
   - Dual values are read from the constraint shadow prices.

2. **Phase II** – fix the optimal score and **maximize total slack** to identify
   any remaining non-radial inefficiency and remove weakly-efficient solutions.
   The slacks and final lambda values come from this phase.

The model is built with [PuLP](https://github.com/coin-or/pulp) and solved with
**Gurobi**. Because the code reads dual values (`constraint.pi`), a solver that
returns LP duals is required.

## Requirements

```
pip install -r requirements.txt
```

Gurobi must be installed with a valid license and reachable by PuLP
(`GUROBI()`). To use the bundled open-source CBC solver instead, replace every
`p1.solve(GUROBI())` / `p2.solve(GUROBI())` with `p1.solve(PULP_CBC_CMD(msg=0))`.

## Input data format

Place an Excel file with a `Sheet1` worksheet where each row is one DMU. The
solver detects roles from the **column headers**:

- columns ending in `(I)` are treated as **inputs**,
- columns ending in `(O)` are treated as **outputs**,
- a `Garage DMUs` column holds the DMU names.

Example (the included `Garage_Data.xlsx`):

| No. | Garage DMUs | Staff (I) | Show room space (100 m2) (I) | ... | Alpha sales (1000 s) (O) | Profit (millions) (O) |
|-----|-------------|-----------|------------------------------|-----|--------------------------|-----------------------|
| 1   | Winchester  | 7         | 8                            | ... | 2.0                      | 1.5                   |

## Usage

Set the orientation in the call to `dea_solver`:

```python
orient, eff_rows, lam_rows, slack_rows, n, m, s = dea_solver(X, Y, orient="oo")  # or "io"
```

Run it:

```
python solver.py
```

The console prints each DMU with its `Theta`, `Phi` and an efficiency flag, plus a
count of efficient DMUs. Results are written to `Results/{orient}crs_results.xlsx`.

## Excel export

Each run produces one `.xlsx` workbook (`iocrs_results.xlsx` or
`oocrs_results.xlsx`) with **three sheets**. Every sheet has one row per DMU,
indexed `1..n` in the first `DMU` column. Below, `m` = number of inputs and
`s` = number of outputs.

### Sheet 1 — `EffWeights`

Efficiency scores and dual (multiplier) values.

| Column | Meaning |
|--------|---------|
| `DMU` | DMU index |
| `Theta` | Input-oriented efficiency score (= `1/phi` in OO runs) |
| `Phi` | Output-oriented expansion factor (= `1/theta` in IO runs) |
| `dualX_1 … dualX_m` | Dual value (weight) on each input constraint |
| `dualY_1 … dualY_s` | Dual value (weight) on each output constraint |

A DMU is **efficient** when `Theta = 1` (equivalently `Phi = 1`). Dual values are
reported as absolute magnitudes so they read as positive multiplier weights.

### Sheet 2 — `Lambda`

The reference/peer set for each DMU.

| Column | Meaning                         |
|--------|---------------------------------|
| `DMU` | DMU index                       |
| `lambda_1 … lambda_n` | Referenced weights on every DMU |

For a given row, the DMUs with non-zero `lambda` are its **peers** on the
efficient frontier; an efficient DMU typically references only itself.

### Sheet 3 — `Slacks`

Slacks plus the projected efficient targets, written side by side.

| Column | Meaning |
|--------|---------|
| `DMU` | DMU index |
| `slackX_1 … slackX_m` | Input slack (extra input that can still be removed after radial scaling) |
| `slackY_1 … slackY_s` | Output slack (extra output achievable after radial scaling) |
| `DMU` | DMU index, repeated to separate the two blocks |
| `Xeff_1 … Xeff_m` | Projected efficient input targets |
| `Yeff_1 … Yeff_s` | Projected efficient output targets |

Targets are computed as:

- **IO:** `Xeff = theta * X - slackX`, `Yeff = Y + slackY`
- **OO:** `Xeff = X - slackX`, `Yeff = phi * Y + slackY`

A fully efficient DMU has all slacks equal to 0 and `Xeff`/`Yeff` equal to its
original inputs/outputs.

## Repository layout

```
solver.py                  # main script (data loading, solving, Excel export)
Example_data/
    Garage_Data.xlsx       # sample 28-DMU dataset
Results/
    iocrs_results.xlsx     # input-oriented output
    oocrs_results.xlsx     # output-oriented output
```

> Note: `solver.py` reads from `Example_data/` and writes to `Results/`. Create the
> `Results/` folder (or adjust the paths) before running.

## Notes

- Tiny values such as `1e-14` in lambda or slack columns are numerical noise from
  the LP solver and can be treated as zero.
- For degenerate DMUs, alternative optimal solutions may yield slightly different
  dual or lambda values depending on the solver; efficiency scores are unaffected.
- The current model is CRS only. A VRS variant can be added by appending the
  convexity constraint `sum_j lambda_j = 1` to both phases.

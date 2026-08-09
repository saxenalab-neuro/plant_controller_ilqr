# Plant-controller iLQR: reproducible workflow

This repository contains a consolidated iterative linear-quadratic regulator (iLQR) implementation for the plant-controller model and three standalone notebooks for reproducing the collective trajectory, subspace, and network-energy analyses.

## Repository contents

| File | Purpose |
| --- | --- |
| `consolidated_plant_controller_ilqr.ipynb` | Runs one controller, angle, and recurrent-network condition and saves its trajectories. |
| `generate_plots_collective.ipynb` | Plots collective trajectories and calculates movement-period plant-input range. |
| `gen_plots_orthogonality_collective.ipynb` | Performs preparatory-versus-movement PCA cross-projection analysis. |
| `energy_plot.ipynb` | Calculates and plots neural-state energy. |

The model notebook and analysis notebooks are intentionally separate. Generate all required trajectories with the consolidated model notebook before running the analyses.

## Software requirements

- Python 3
- NumPy
- PyTorch
- Matplotlib
- scikit-learn
- JupyterLab or Jupyter Notebook

One possible CPU environment is:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install jupyterlab numpy torch matplotlib scikit-learn
jupyter lab
```

On Windows PowerShell, activate the environment with:

```powershell
.venv\Scripts\Activate.ps1
```

Install the appropriate PyTorch build separately if the workstation requires a specific CUDA version.

## Workflow overview

The complete workflow is:

1. Select one controller type, angle, and W-network type.
2. Run `consolidated_plant_controller_ilqr.ipynb` to convergence or the iteration limit.
3. Repeat until all nine controller-angle combinations exist for the chosen W type.
4. Run the three analysis notebooks using that same W type.
5. Repeat the nine-condition generation and analyses if results are required for the other W type.

## Step 1: choose the model condition

Open `consolidated_plant_controller_ilqr.ipynb` and edit its selection cell.

### Controller type

```python
controller_types = ("recurrence", "orthogonal", "overlapping")
controller_type = "recurrence"
```

The controller configurations are:

| Controller | Plant-control source | Constraint interpretation |
| --- | --- | --- |
| `recurrence` | Neural state `[x1, x2]` | Recurrence-constrained controller. |
| `orthogonal` | Kinematic state `[y, y_dot]` | Separates preparatory and movement dynamics through the goal constraints. |
| `overlapping` | Kinematic state `[y, y_dot]` | Encourages shared or overlapping neural dynamics. |

### Angle

```python
angle_options = {
    "pi/8": (1, 22.5),
    "pi/4": (2, 45.0),
    "3pi/8": (3, 67.5),
}
angle_selection = "pi/8"
```

The integer is the condition index used in saved filenames.

| Selection | Angle | Filename index |
| --- | ---: | ---: |
| `pi/8` | 22.5° | 1 |
| `pi/4` | 45° | 2 |
| `3pi/8` | 67.5° | 3 |

### Recurrent weight matrix

```python
w_types = ("non_normal", "oscillatory")
w_type = "oscillatory"
```

The available recurrent matrices are:

```python
w_options = {
    "non_normal": ((0.0, 0.0), (1.0, 0.0)),
    "oscillatory": ((0.0, -1.0), (1.0, 0.0)),
}
```

Keep `w_type` fixed while generating the nine conditions used in one collective analysis.

### Convergence threshold

```python
convergence_threshold = 0.5
```

The run stops when:

```python
abs(y - y_target) < convergence_threshold
```

The solver is capped at 500 iterations. Trajectories are saved every tenth iteration and on the iteration where the convergence threshold is reached.

## Step 2: review optimization settings

The model notebook uses these defaults:

| Setting | Default |
| --- | ---: |
| Horizon | 1750 samples |
| Preparatory period | 750 samples |
| Maximum iLQR iterations | 500 |
| Control-update step size | 0.1 |
| Save interval | 10 iterations |
| Initial position | 0.1 |
| Target position | 10.0 |

The following environment variables may override paths or large-run settings before the corresponding configuration cell is executed:

| Environment variable | Purpose |
| --- | --- |
| `ILQR_HORIZON` | Total trajectory horizon. |
| `ILQR_PREP_STEPS` | Number of preparatory samples. |
| `ILQR_MAX_ITERATIONS` | Iteration limit, still capped at 500. |
| `ILQR_OUTPUT_ROOT` | Parent directory for generated data folders. |

For a controlled replication, retain identical optimization settings across all compared conditions.

## Step 3: generate all required inputs

Run every cell through **Run the selected model**. Then change `controller_type` and `angle_selection` and rerun the selection, configuration, plant-model, solver, and execution cells.

For one W type, generate these nine runs:

| Controller | `pi/8` | `pi/4` | `3pi/8` |
| --- | :---: | :---: | :---: |
| `overlapping` | ✓ | ✓ | ✓ |
| `orthogonal` | ✓ | ✓ | ✓ |
| `recurrence` | ✓ | ✓ | ✓ |

To compare both W types, repeat the full table with `w_type = "non_normal"` and `w_type = "oscillatory"`, for 18 total runs.

The notebook prints terminal `y` and signed error every tenth iteration. Confirm whether each run reports `converged=True`. A run that reaches the 500-iteration cap may still produce saved arrays, but it should be identified as nonconverged in any scientific comparison.

## Step 4: verify generated files

Outputs are organized by controller and W type:

```text
data_overlapping/
├── non_normal/
└── oscillatory/

data_orthogonal/
├── non_normal/
└── oscillatory/

data_recurrence/
├── non_normal/
└── oscillatory/
```

Each W-type folder should contain four files for each angle index:

```text
gocue_<controller>_<index>.npy
ss_<controller>_<index>.npy
uinput_traj_<controller>_<index>.npy
y_<controller>_<index>.npy
```

For example:

```text
data_recurrence/oscillatory/gocue_recurrence_1.npy
data_recurrence/oscillatory/ss_recurrence_1.npy
data_recurrence/oscillatory/uinput_traj_recurrence_1.npy
data_recurrence/oscillatory/y_recurrence_1.npy
```

Expected array layouts are:

| Array | Shape | Contents |
| --- | --- | --- |
| `gocue` | `(T, 1)` | Go-cue state. |
| `ss` | `(T, 2)` | Neural states `[x1, x2]`. |
| `uinput` | `(T, 2)` | Plant inputs `[u1, u2]`. |
| `y` | `(T, 2)` | Point-mass position and velocity `[y, y_dot]`. |

With the default horizon, `T` is 1749 because saved arrays contain the post-transition samples.

## Step 5: configure the analysis notebooks

Each analysis notebook contains a **Data configuration** cell:

```python
data_root = Path(".")
w_types = ("non_normal", "oscillatory")
w_type = "oscillatory"
```

Use `data_root = Path(".")` when Jupyter is launched from the repository root and the generated data folders are located there. Otherwise, point `data_root` to their parent directory.

The analysis `w_type` must match the generated dataset. Each notebook checks for all expected conditions and raises a clear error if a file is absent or has an unexpected shape.

## Step 6: run the collective trajectory analysis

Open and run `generate_plots_collective.ipynb` from top to bottom.

It reproduces:

- go-cue trajectories;
- point-mass position and velocity;
- both neural-state dimensions;
- both plant-input dimensions; and
- movement-period plant-input range.

Movement onset is detected as the first saved go-cue sample greater than or equal to 0.5. The input-range metric is:

```text
maximum over input dimensions of the peak-to-peak input after movement onset
```

Bars show the mean across angles and colored points show individual angle conditions.

## Step 7: run the preparatory/movement subspace analysis

Open and run `gen_plots_orthogonality_collective.ipynb` from top to bottom.

For each controller and angle, the notebook:

1. standardizes plant inputs across the complete trial;
2. divides the input trajectory at the detected go cue;
3. fits separate two-component PCA models to preparation and movement;
4. reconstructs both epochs using the first component from each PCA model; and
5. compares within-epoch and cross-epoch explained variance.

The returned `subspace_scores` array is organized as:

```text
[controller, angle, evaluated epoch, fitted subspace]
```

The epoch and fitted-subspace orders are both `preparation`, then `movement`.

## Step 8: run the energy analysis

Open and run `energy_plot.ipynb` from top to bottom.

For each controller and angle, neural-state energy is defined as:

```text
sum over time and neural dimensions of ss**2
```

The notebook displays per-angle energy and the mean energy across angles for each controller.

## Reproducibility practices

- Keep the same notebook versions for generation and analysis.
- Use the same Python environment for all runs in one comparison.
- Record `controller_type`, `angle_selection`, `w_type`, convergence threshold, horizon, preparatory period, step size, and iteration limit.
- Do not mix files generated with different optimization settings in the same analysis folder.
- Keep raw `.npy` outputs unchanged after generation.
- Retain console output indicating iteration count, terminal error, and convergence status.
- The analysis notebooks fix the NumPy seed, record runtime package versions, validate input paths and shapes, and derive movement onset from the saved go cue.
- Figures are displayed inline and are not written to disk.

## Troubleshooting

### Missing-file error

Confirm that all three controllers and all three angle indices were generated for the selected `w_type`. Check that `data_root` points to the directory containing the three `data_*` folders.

### Analysis loads the wrong dataset

Make sure the analysis notebook's `w_type` exactly matches the W-type subfolder used during model generation.

### Shape-validation error

Regenerate the affected condition with the consolidated notebook. Do not mix outputs produced with different horizons.

### The solver is slow

The default horizon and Jacobian calculations are computationally demanding. Use the production defaults for final results. Reduced horizons may be useful for smoke testing but should not be mixed with production outputs.

### The solver reaches 500 iterations

Inspect the final signed error and treat the condition as nonconverged if it does not satisfy the selected convergence threshold. Preserve that status when reporting or comparing results.

## Minimal replication checklist

- [ ] Create and activate the Python environment.
- [ ] Select one W type.
- [ ] Generate all nine controller-angle conditions.
- [ ] Confirm the expected four arrays for every condition.
- [ ] Confirm convergence status and retain the solver settings.
- [ ] Set the same `w_type` and correct `data_root` in each analysis notebook.
- [ ] Run `generate_plots_collective.ipynb`.
- [ ] Run `gen_plots_orthogonality_collective.ipynb`.
- [ ] Run `energy_plot.ipynb`.
- [ ] Repeat for the second W type if required.

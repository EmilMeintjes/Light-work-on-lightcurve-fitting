# Making-light-work-of-lightcurve-fitting
This is a user-friendly Python pipeline that can be used to fit some astronomical light curves.

The idea is that you, as the user, retain the most control over the fitting procedure to lightcurve without the need to write the code yourself. 

The pipeline is laid out as follows:

lightcurve_fitter/

├── fitting_models.py  — fit functions + prior builder 

├── persistence.py     — JSON region store + .npy MCMC results

├── selector.py        — Stage 1 interactive region picker

├── initialiser.py     — Stage 2 slider-based parameter initialisation

├── fitter.py          — Stage 3 PyAutoFit/emcee wrapper

├── plots.py           — Stage 4 overview, fit, and corner plots

└── main.py            — Command Line Interface entry point wiring all stages together

---

## Stages

**Stage 1 — Region selection (`selector.py`)**
An interactive matplotlib window displays the full lightcurve. Click once to set the start of a region, click again to set the end. Choose a model from the dropdown, add an optional note, and press Enter (or the Confirm button) to save the region. Multiple regions can be defined in a single session. Regions are saved to a JSON file so the session can be resumed at any later stage.

**Stage 2 — Parameter initialisation (`initialiser.py`)**
For each saved region, a slider window opens showing only the data within that region and a live model curve. Drag the sliders to set initial parameter guesses visually. The optional Curve Fit button runs SciPy's Levenberg–Marquardt algorithm from your current slider position to refine the guesses; you can then accept or reject that result before moving on. You may choose to use the parameters estimated by curve_fit or your own estimation for the following step.

**Stage 3 — MCMC fitting (`fitter.py`)**
A robust MCMC fit is performed using PyAutoFit/emcee. Uniform priors are built automatically from the Stage 2 guesses: by default the prior for each parameter spans ±30% of the best-fit parameters determined by Stage 2. The sampler runs 60 walkers for 1500 steps; the first 300 are discarded as burn-in. Best-fitting parameters are reported as the median of the posterior, with uncertainties estimated from the 16th and 84th percentiles (equivalent to a 1 $$\sigma$$ confidence interval).

**Stage 4 — Plots (`plots.py`)**
Three plot types are produced for each fitted region: an overview plot showing the full lightcurve with all model fits overlaid; a per-region fit plot with the posterior median and 16/84 shaded band; and a corner plot showing the full posterior distribution for all parameters.

---

## Available models

All models include a free `y_offset` parameter representing the quiescent baseline level. Time is always in region-relative coordinates (zero at the region start).

| Model | Equation |
|-------|----------|
| Gaussian | $A \mathrm{e}^{-((t - \mu)/\sigma)^2 / 2} + y_{\rm offset}$ |
| Rising exponential | $A \mathrm{e}^{t/\tau_{\rm rise}} + y_{\rm offset}$ |
| Decaying exponential | $A \mathrm{e}^{-t/\tau_{\rm decay}} + y_{\rm offset}$ |

More to be added soon!

---

## Data format

`load_data()` in `main.py` expects a whitespace-delimited text file with either two or three columns:

```
# time   flux   [uncertainty]
58000.0  1.23   0.05
58001.0  1.41   0.06
...
```

Lines beginning with `#` are treated as comments. If no uncertainty column is present, the MCMC likelihood is evaluated without weighting. To load a different format (FITS, CSV, MeerKAT archive output, etc.) replace the body of `load_data()` in `main.py`.

## General usage

Run the full pipeline end-to-end:

```bash
python main.py --stage all --data /path/to/data.txt
```

Run individual stages:

```bash
python main.py --stage select      # Stage 1: pick regions interactively
python main.py --stage init        # Stage 2: set initial guesses via sliders
python main.py --stage fit         # Stage 3: run MCMC
python main.py --stage plot        # Stage 4: generate plots
```

---

## Command-line arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--stage` | str | `all` | Pipeline stage: `select`, `init`, `fit`, `plot`, or `all` |
| `--data` | str | — | Path to the lightcurve data file |
| `--regions` | str | `regions.json` | Path to the regions JSON file |
| `--results` | str | `results/` | Directory for MCMC output files |
| `--plots` | str | `plots/` | Directory for output plots |
| `--ids` | int(s) | — | Restrict `init`/`fit` to specific segment IDs |
| `--xscale` | str | `linear` | X-axis scale: `linear` or `log` |
| `--yscale` | str | `linear` | Y-axis scale: `linear` or `log` |
| `--walkers` | int | 60 | Number of emcee walkers |
| `--steps` | int | 1500 | Number of emcee steps per walker |
| `--burn` | int | 300 | Burn-in steps to discard |
| `--force` | flag | False | Delete stale PyAutoFit directories and refit from scratch |

---


## Example use cases

### 1. Fitting a lightcurve with a log-scale axis

Use `--yscale log` and `--xscale log`:

```bash
python main.py --stage select --data example.txt --yscale log --xscale log --regions not_so_random_name.json
python main.py --stage init   --data example.txt --regions not_so_random_name.json
python main.py --stage fit    --data example.txt --regions not_so_random_name.json
python main.py --stage plot   --data example.txt --regions not_so_random_name.json
```

The log scale is stored in `not_so_random_name.json` after selection, so all subsequent stages restore it automatically without needing to pass `--yscale` and `--xscale` again.

Change to `--yscale linear` and/or `--yscale linear` if you want to revert.

---

### 2. Fitting separate rise and decay phases of the same lightcurve.

If you want to fir the rise and decay of a flare independently, define two regions during selection and assign different models to each using the available dropdown menu of the GUI that appears:

- Region 1: rising exponential, covering the pre-peak data
- Region 2: decaying exponential, covering the post-peak data

```bash
python main.py --stage all --data flare.txt --regions flare_regions.json
# In the selector window:
#   Click to mark region 1 (rise), choose "Rising Exp", confirm
#   Click to mark region 2 (decay), choose "Decaying Exp", confirm
#   Press q to save and quit
```

The overview plot will overlay both model components on the full lightcurve.

---

### 3. Re-fitting a single region after a crashed run

If the MCMC for one region crashes or produces a poor result, use `--ids` to re-run only that segment and `--force` to clear the stale PyAutoFit state:

```bash
# Re-initialise region 3 with fresh slider guesses
python main.py --stage init --data mydata.txt --regions myregions.json --ids 3

# Refit region 3 from scratch
python main.py --stage fit  --data mydata.txt --regions myregions.json --ids 3 --force
```

All other regions are left untouched. 

### 4. Working with multiple sources or frequency bands

Pass a different `--regions` file for each source or band. Results and plots directories can be separated in the same way:

```bash
# 1.4 GHz lightcurve
python main.py --stage all \
    --data nova_1p4GHz.txt \
    --regions nova_1p4GHz_regions.json \
    --results results_1p4GHz \
    --plots   plots_1p4GHz

# 5 GHz lightcurve
python main.py --stage all \
    --data nova_5GHz.txt \
    --regions nova_5GHz_regions.json \
    --results results_5GHz \
    --plots   plots_5GHz
```

Each run is fully independent and can be compared afterwards using the saved summary JSON files in each results directory.

## Output files

After a complete run you will find:

```
results/
├── seg0001_samples.npy      — full posterior sample array  (n_draws × n_params)
├── seg0001_lnprob.npy       — log-probability for each draw
├── seg0001_summary.json     — median + 16/84 percentile statistics per parameter
└── pyautofit/               — internal PyAutoFit state (can be deleted after fitting)

plots/
├── overview.png             — full lightcurve with all fitted models
├── seg0001_fit.png          — data + median model + posterior band for region 1
└── seg0001_corner.png       — posterior corner plot for region 1
```

The summary JSON files are plain text and can be read directly or parsed into a table for publication:

```json
{
  "param_names": ["amplitude", "centre", "sigma", "y_offset"],
  "statistics": {
    "amplitude": {"median": 4.21, "err_lo": 0.18, "err_hi": 0.20},
    ...
  },
  "metadata": {"model": "gaussian", "t_ref": 58035.0, ...}
}
```

**Note that all time-related parameters (`centre`, `tau_rise`, `tau_decay`) are in _region-relative coordinates_ — add `t_ref` (the region start time) to convert back to the original time axis.**

  ## Known limitations for now ;)

- Each region is fitted with a single model component. Overlapping or blended peaks within one region are not currently supported.
- No convergence diagnostics are done after MCMC. If in doubt, increase `--steps`.
- The MCMC cannot be warm-started from a previous interrupted run. Use `--force` to restart from scratch.
- `load_data()` is a stub for plain text files; replace it in `main.py` for other formats (FITS, CSV, etc.).
- Gaussian fit is currently misbehaving in the MCMC fit (likely due to poor prior generation (currently being looked into)




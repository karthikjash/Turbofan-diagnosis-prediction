# Explainable Turbofan Engine RUL Estimation

An end-to-end prognostics and health management (PHM) project for estimating the remaining useful life (RUL) of turbofan engines. It combines sensor preprocessing, PCA, and a Gaussian hidden Markov model (HMM) with a left-to-right degradation topology to produce both RUL estimates and interpretable health-state probabilities.

This implementation is evaluated on **NASA C-MAPSS FD001**. The notebook, `main.ipynb`, contains the complete analysis, model development, evaluation, and visualizations.

## Highlights

- End-to-end analysis from raw engine cycles to end-of-sequence RUL predictions.
- Sensor selection based on within-engine variance, with highly correlated sensor pruning.
- Piecewise-linear RUL labels capped at 125 cycles.
- PCA retaining 95% of variance (10 components in this run).
- Gaussian HMM with five health states and constrained left-to-right transitions.
- Posterior-weighted RUL estimates and per-cycle state probabilities.
- Held-out FD001 test evaluation compared with a mean-RUL baseline.

## Dataset

The repository includes the FD001 train, test, and test-RUL files:

| File | Contents |
|---|---|
| `train_FD001.txt` | 100 run-to-failure engine trajectories |
| `test_FD001.txt` | 100 partial engine trajectories |
| `RUL_FD001.txt` | True RUL for each corresponding test engine at its final observed cycle |

The FD001 training data contains **20,631 cycles** across 100 engines; the test data contains **13,096 cycles** across 100 engines. Each row includes an engine identifier, cycle, three operating settings, and 21 sensor measurements. Dataset source and subset characteristics are listed in the [NASA C-MAPSS dataset catalog](https://catalog.data.gov/dataset/cmapss-jet-engine-simulated-data).

## Method

1. **Load and inspect** FD001 training, test, and RUL files; check for missing values.
2. **Detect outliers** using a 3.0-IQR threshold on sensor measurements. The notebook flags and removes 1,224 of 20,631 training rows (5.93%).
3. **Select sensors** by removing sensors whose mean within-engine variance is below `1e-6`; prune highly correlated sensors above `0.98` absolute correlation. Fourteen sensors remain: `s2`, `s3`, `s4`, `s7`, `s8`, `s9`, `s11`, `s12`, `s13`, `s14`, `s15`, `s17`, `s20`, and `s21`.
4. **Create RUL labels** using `max_cycle - cycle`, clipped at 125 cycles.
5. **Standardize** selected sensors with `StandardScaler`, fitted on the cleaned training data and applied to test data.
6. **Build sliding windows** of 30 cycles with a stride of one for exploratory feature construction. Separately, the HMM is trained on complete per-engine sequences to preserve temporal continuity.
7. **Reduce dimensionality** using PCA, retaining the smallest number of components that explain at least 95% of variance (10 components).
8. **Fit and tune a left-to-right Gaussian HMM.** Transition probabilities below the diagonal are constrained to zero. Each non-final state has a self-loop and a forward transition; the final state is absorbing.
9. **Estimate RUL** by weighting health-state RUL-bin midpoints by each cycle's posterior state probabilities. Final test predictions use the posterior estimate at the last observed cycle, a training-derived bias adjustment, and clipping to `[0, 125]` cycles.

### Health states

The notebook divides the capped RUL range into five equal-width bins. State names are ordered from healthy to critical; the state index mapping is also shown below.

| State index | Label | RUL midpoint used for prediction |
|---:|---|---:|
| 0 | Healthy | 112.5 cycles |
| 1 | Early Degradation | 87.5 cycles |
| 2 | Moderate Degradation | 62.5 cycles |
| 3 | Advanced Degradation | 37.5 cycles |
| 4 | Critical | 12.5 cycles |

The HMM's state sequence is decoded with Viterbi; RUL estimates use the posterior probabilities rather than the decoded state alone.

## Hyperparameters and configuration

| Component | Setting |
|---|---|
| Random seed | `42` |
| Outlier rule | Per-sensor IQR bounds, multiplier `3.0` |
| Within-engine variance cutoff | `1e-6` |
| Absolute sensor-correlation threshold | `0.98` |
| RUL cap | `125` cycles |
| Sliding-window length / stride | `30` / `1` cycle |
| PCA selection | Minimum components reaching `95%` cumulative explained variance; `10` selected |
| HMM type | Gaussian HMM with full covariance |
| HMM state counts searched | `3, 4, 5, 6, 7` |
| Transition topology | Left-to-right; no backward transitions; final state absorbing |
| Initial self-loop probability | `0.85` |
| Initial-state distribution | `[1, 0, ..., 0]` |
| Initial HMM training iterations | Up to `150` |
| Initial HMM convergence tolerance | `1e-4` |
| Fine-tuned HMM training iterations | Up to `200` |
| Fine-tuned HMM convergence tolerance | `1e-5` |
| HMM random state | `42` |
| Test-set bias correction | `+19.48` cycles, estimated as mean `(true RUL - predicted RUL)` on training trajectories |

The grid search selected five states by training-set RMSE. It compared candidate models using RMSE, MAE, and the notebook's asymmetric NASA score. The notebook also reports an EM convergence warning for the fine-tuned model; the metrics below are the stored outputs from that run.

## Results

### FD001 final evaluation

| Metric | Training trajectories* | Held-out test engines |
|---|---:|---:|
| RMSE (cycles) | 30.24 | **30.11** |
| MAE (cycles) | 24.16 | **23.40** |
| NASA score (lower is better) | 427,279.91 | **9,993.96** |
| HMM states | 5 | 5 |
| PCA components | 10 | 10 |

\*The training metrics are calculated on the same run-to-failure trajectories used to fit the HMM, so they are in-sample and should not be interpreted as an independent generalization estimate. The test metrics are computed on the 100 held-out test engines using their final observed cycle and the provided per-engine RUL targets. The scalar bias correction is derived from training predictions.

### Mean-RUL baseline comparison on FD001 test

| Metric | Mean-RUL baseline | PCA-HMM | Relative error reduction |
|---|---:|---:|---:|
| RMSE (cycles) | 42.61 | **30.11** | **29.3%** |
| MAE (cycles) | 35.90 | **23.40** | **34.8%** |

The notebook reports no NASA-score baseline comparison. NASA's asymmetric score penalizes late RUL predictions more heavily than early predictions; lower scores are better.

### HMM state-count search

Scores below are the notebook's training-set grid-search results. The five-state model is selected by RMSE.

| States | RMSE (cycles) | MAE (cycles) | NASA score |
|---:|---:|---:|---:|
| 3 | 38.93 | 29.55 | 4,151,973.84 |
| 4 | 42.54 | 33.55 | 2,364,527.45 |
| **5** | **30.50** | 24.37 | **437,351.65** |
| 6 | 30.97 | **24.01** | 515,144.18 |
| 7 | 39.39 | 30.90 | 1,944,561.56 |

## Visual results

The notebook saves figures under `outputs/`. Selected figures:

| Figure | Description |
|---|---|
| `outputs/final_metrics_summary.png` | Training vs. test RMSE and MAE |
| `outputs/test_evaluation.png` | Test predictions vs. targets and residuals |
| `outputs/test_per_engine_rul.png` | True and predicted final-cycle RUL by test engine |
| `outputs/test_health_dashboard.png` | State posterior heatmaps and RUL trajectories for four test engines |
| `outputs/learned_transition_matrix.png` | Learned HMM transition matrix |
| `outputs/hyperparameter_search.png` | Metrics across candidate HMM state counts |
| `outputs/pca_explained_variance.png` | PCA explained-variance selection |
| `outputs/pca_degradation_trajectory.png` | Selected training-engine trajectories in PC space |
| `outputs/outlier_detection.png` | IQR outlier visualization |
| `outputs/eda_sensor_correlation.png` | Sensor correlation overview |

## Run the notebook

The notebook was developed in a notebook environment and installs its dependencies in the first setup cell. To run it locally, install Python and Jupyter, then install the libraries used by the notebook:

```bash
python -m pip install jupyter hmmlearn scikit-learn pandas numpy matplotlib seaborn scipy tqdm
jupyter notebook main.ipynb
```

Run `main.ipynb` from the repository root so its relative paths resolve to the FD001 data files. The notebook writes visualizations to `outputs/` as configured in this repository.

## Repository contents

```text
.
├── main.ipynb
├── train_FD001.txt
├── test_FD001.txt
├── RUL_FD001.txt
└── outputs/
    └── generated analysis figures
```

## Future work

NASA C-MAPSS comprises **four standard subsets: FD001, FD002, FD003, and FD004**. This project currently evaluates FD001. The other three subsets broaden the operating and fault conditions and provide a useful next step for testing robustness and generalization. NASA's catalog gives these trajectory counts and conditions:

| Subset | Training engines | Test engines | Operating conditions | Fault modes |
|---|---:|---:|---|---|
| FD001 (evaluated here) | 100 | 100 | 1 | 1: HPC degradation |
| FD002 | 260 | 259 | 6 | 1: HPC degradation |
| FD003 | 100 | 100 | 1 | 2: HPC and fan degradation |
| FD004 | 248 | 249 | 6 | 2: HPC and fan degradation |

Promising extensions include:

- Evaluate the same pipeline on **FD002, FD003, and FD004**, reporting results separately by subset and comparing against published or consistently reimplemented baselines.
- Revisit outlier handling and sensor selection for multi-condition subsets, where operating regime changes can be mistaken for degradation without condition-aware preprocessing.
- Use engine-level validation or cross-validation for model selection and keep the official test labels out of tuning decisions.
- Tune HMM state count and topology per subset, then evaluate whether a shared model can transfer across subsets.
- Add uncertainty calibration and report prediction intervals alongside posterior health-state probabilities.
- Compare PCA-HMM with simpler statistical, tree-based, and sequence-model baselines under the same split and scoring protocol.
- Track per-engine errors and the asymmetric NASA score alongside RMSE and MAE to expose early/late prediction trade-offs.

The subset counts and scenario descriptions are from the [NASA C-MAPSS catalog](https://catalog.data.gov/dataset/cmapss-jet-engine-simulated-data). For FD004, NASA lists 248 training trajectories and 249 test trajectories.

## References

- A. Saxena, K. Goebel, D. Simon, and N. Eklund, [“Damage Propagation Modeling for Aircraft Engine Run-to-Failure Simulation,”](https://ti.arc.nasa.gov/tech/dash/groups/pcoe/prognostic-data-repository/) *International Conference on Prognostics and Health Management*, 2008.
- NASA, [C-MAPSS Jet Engine Simulated Data](https://catalog.data.gov/dataset/cmapss-jet-engine-simulated-data).
- Project implementation and output evidence: [`main.ipynb`](main.ipynb).

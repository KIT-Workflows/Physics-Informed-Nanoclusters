# README for `cluster_13_reviewed.ipynb`

This folder contains a Colab-oriented notebook that documents the supervised, interpretable machine-learning part of the paper:

**"Interpretable, Physics-Informed Learning Reveals Sulfur Adsorption and Poisoning Mechanisms in 13-Atom Icosahedra Nanoclusters"**

The notebook is best understood as a compact, self-contained analysis layer built on top of a curated DFT-derived dataset for 30 transition-metal `TM13` icosahedral nanoclusters. Its main role is to support the adsorption-energy prediction, validation, and feature-importance results discussed in the manuscript.

## What this notebook does

The notebook:

- embeds the full `N = 30` dataset directly inside the notebook as a TSV table;
- defines the sulfur adsorption target as `Eads_s = e_int_s + e_dis_s`;
- demonstrates why using `e_int_s` and `e_dis_s` as model inputs would be target leakage;
- trains three interpretable regression models from **pristine-cluster descriptors only**:
  - `ElasticNetCV`
  - `RidgeCV`
  - `ExplainableBoostingRegressor` (EBM)
- evaluates the models under:
  - LOOCV for small-sample benchmarking;
  - group holdout by periodic row (`3d`, `4d`, `5d`) to test chemical extrapolation;
- computes LOFO (leave-one-feature-out / drop-column) importance by full retraining;
- checks whether the HOMO-LUMO gap contributes signal beyond `eb`;
- bootstraps ElasticNet coefficients to assess coefficient stability under correlated descriptors;
- exports manuscript-style figures for feature importance and DFT-vs-ML adsorption trends.

## What this notebook does not do

This notebook does **not** reproduce the full paper workflow end to end. In particular, it does not:

- run VASP or any other DFT calculations;
- optimize pristine or adsorbed nanocluster geometries;
- perform vibrational calculations;
- generate the original adsorption structures or site searches;
- run the `SO2/TM13` follow-up calculations;
- reproduce the unsupervised PCA and k-means analysis described in the full manuscript workflow.

Those steps belong to the broader DFT pipeline described in the paper and Supporting Information. This notebook only covers the compact supervised-ML and interpretability analysis built from the curated descriptor table.

## Files in this folder

- `cluster_13_reviewed.ipynb`: the main Colab notebook.
- `all_no_normalization_data.xlsx`: an archival copy of the curated descriptor table.

Important: the current notebook does **not** read the Excel file during execution. It embeds the table directly as a raw TSV string, which makes the notebook self-contained for Colab use.

## Relationship to the paper

This notebook most directly supports the supervised-learning results associated with the manuscript discussion around feature importance and adsorption-energy prediction.

In practice, the clearest links are:

- the LOFO importance analysis, which supports the interpretation presented around the manuscript's feature-importance discussion and Figure 5;
- the DFT-versus-model trend plot across the `3d`, `4d`, and `5d` series, which corresponds to the type of comparison shown in Figure 6.

The notebook also includes important methodological checks that justify the modeling protocol used in the paper:

- explicit leakage detection;
- outer-fold evaluation logic;
- group-aware validation for chemical extrapolation;
- stability checks under correlated descriptors.

## Data used by the notebook

The notebook contains 30 rows, one for each transition metal considered in the fixed `TM13` icosahedral family:

- `3d`: Sc, Ti, V, Cr, Mn, Fe, Co, Ni, Cu, Zn
- `4d`: Y, Zr, Nb, Mo, Tc, Ru, Rh, Pd, Ag, Cd
- `5d`: Lu, Hf, Ta, W, Re, Os, Ir, Pt, Au, Hg

The main identifier is:

- `MTs`: atomic number of the transition metal

The pristine descriptors used as model inputs are:

- `eb`
- `ecn`
- `dav`
- `cgd`
- `freq_min`
- `freq_max`
- `e_0`
- `gap`

In the plotting cells, these are labeled in manuscript-style notation as:

- `eb` -> binding energy `E_b`
- `ecn` -> effective coordination number `ECN`
- `dav` -> average bond distance `d_av`
- `cgd` -> `epsilon_d`
- `freq_min` and `freq_max` -> vibrational descriptors
- `gap` -> HOMO-LUMO gap `E_gap`

The notebook also stores adsorbed-state and response columns such as:

- `eb_s`, `ecn_s`, `dav_s`, `cgd_s`
- `d_dav_s`, `d_ecn_s`, `d_cgd_s`
- `e_dis_s`, `e_int_s`
- `gap_s`

The target is then reconstructed inside the notebook as:

```python
df["Eads_s"] = df["e_int_s"] + df["e_dis_s"]
```

This target definition is central to the workflow: the notebook first proves that any model using `e_int_s` or `e_dis_s` as predictors would be trivially leaky.

## Recommended execution order

Even though some section numbers in the notebook are not perfectly ordered, the notebook should be run **strictly from top to bottom**.

The effective workflow is:

1. Install Colab dependencies.
2. Import Python packages.
3. Load the embedded `N=30` table into a pandas DataFrame.
4. Define the target `Eads_s` and the pristine feature set.
5. Run the leakage sanity check.
6. Benchmark `ElasticNetCV` and `RidgeCV` under LOOCV.
7. Define periodic-series groups and run group holdout validation (`3d`, `4d`, `5d`).
8. Compute LOFO importance under LOOCV and group holdout.
9. Test whether `gap` adds information beyond `eb` by residualization.
10. Assess ElasticNet coefficient stability by bootstrap resampling.
11. Fit an interpretable nonlinear EBM model and inspect global effects.
12. Generate the manuscript-style figure exports.

Later cells depend on variables and helper functions defined earlier. Running only the final plotting cells without executing the full notebook first will fail.

## Colab dependencies

The notebook installs:

- `interpret`
- `pygam`
- `pymatgen`

Notes:

- `interpret` is required for EBM.
- `pymatgen` is used in the final figure cell to convert atomic numbers into element symbols.
- `pygam` is installed but not essential for the current stored workflow.

The rest of the notebook uses standard scientific Python packages:

- `numpy`
- `pandas`
- `matplotlib`
- `scikit-learn`
- `joblib`

## Validation strategy

This notebook is deliberately conservative for such a small dataset.

### 1. Leakage check

The first diagnostic confirms that:

```text
Eads_s = e_int_s + e_dis_s
```

exactly holds for every row. The notebook then shows that including `e_int_s` and `e_dis_s` as features produces essentially perfect performance, which is correctly treated as leakage rather than a meaningful predictive result.

### 2. LOOCV baselines

Using only pristine descriptors, the notebook evaluates:

- `ElasticNetCV`
- `RidgeCV`
- `EBM`

under leave-one-out cross-validation. Hyperparameters for the linear models are selected inside each outer training fold.

### 3. Group holdout by periodic row

The stricter validation mode holds out one complete chemical series at a time:

- train on `4d + 5d`, test on `3d`;
- train on `3d + 5d`, test on `4d`;
- train on `3d + 4d`, test on `5d`.

This is important because random train/test splits can look overly optimistic when chemically similar metals appear in both training and test sets.

### 4. LOFO importance

Feature importance is computed by dropping one descriptor, retraining the model, and measuring the performance loss. This is more defensible than reading raw coefficients when descriptors are correlated.

## Main generated outputs

If the notebook is run to completion in its current form, it saves:

- `lofo_importance_groupcv.jpg`
- `fig6_trends_3models_logo.jpg`

These correspond to the two most clearly manuscript-facing exported graphics in the notebook.

### `lofo_importance_groupcv.jpg`

This figure is generated from the LOFO cell with:

```python
CV_SCHEME = "group"
```

It reports feature-importance drops across all three models and summarizes the average importance across models.

### `fig6_trends_3models_logo.jpg`

This figure compares DFT adsorption-energy trends with predictions from:

- `ElasticNetCV`
- `RidgeCV`
- `EBM`

for the `3d`, `4d`, and `5d` series separately.

The plotting code automatically switches to `|Eads|` when the target values are predominantly negative, which matches the manuscript presentation.

## Representative outputs stored in the notebook

The notebook already contains saved outputs from a previous run. These numbers are useful as a reference when checking reproducibility.

Key examples from the stored output are:

- leakage test with leaky variables: `R2 ~ 0.999998`, `MAE ~ 0.002 eV`, `RMSE ~ 0.003 eV`
- LOOCV:
  - `ElasticNetCV`: `R2 = -0.145`, `MAE = 1.649`, `RMSE = 2.253`
  - `RidgeCV`: `R2 = -0.026`, `MAE = 1.504`, `RMSE = 2.132`
  - `EBM`: `R2 = 0.036`, `MAE = 1.660`, `RMSE = 2.067`
- group holdout:
  - `ElasticNetCV`: `R2 = -0.431`, `MAE = 1.936`, `RMSE = 2.517`
  - `RidgeCV`: `R2 = -0.109`, `MAE = 1.728`, `RMSE = 2.217`
  - `EBM`: `R2 = -0.059`, `MAE = 1.762`, `RMSE = 2.166`

These results show why the notebook emphasizes methodological discipline:

- leaky features can make the problem look almost perfectly solved;
- chemically meaningful holdout is harder than ordinary LOOCV;
- the final interpretation depends more on robustness checks than on a single headline score.

## Interpretation notes

The notebook is designed to support a cautious, physics-informed interpretation rather than to maximize predictive accuracy at all costs.

Its main methodological messages are:

- use pristine descriptors if the goal is forward prediction;
- separate target construction from model inputs;
- do not trust random or convenience splits alone;
- use drop-column retraining instead of naive coefficient reading when descriptors are correlated;
- compare linear and nonlinear interpretable models before making chemical claims.

## Reproducibility notes

- The notebook sets `random_state=0` where appropriate.
- Linear models standardize the input features inside scikit-learn pipelines.
- The final LOFO figure uses joblib parallelism and limits BLAS thread counts to avoid oversubscription.
- Small numerical differences may still appear across environments because package versions in Colab can change over time.

## Suggested citation context inside the paper package

If this folder is distributed with the manuscript, the safest wording is:

> This notebook reproduces the supervised regression, leakage control, group-aware validation, LOFO feature-importance analysis, and manuscript figure-generation steps based on the curated 30-system descriptor table. It is a support document for the paper's interpretable ML results, not a replacement for the underlying DFT workflow.

## Related manuscript files

For the full scientific context, see:

- `../main.pdf`
- `../SI_main.pdf`

The paper and SI contain the full physical definitions of the descriptors, the DFT protocol, and the broader workflow beyond this notebook.

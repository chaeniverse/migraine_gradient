# migraine_gradient

Whole-brain functional connectome gradient (manifold) analysis of episodic migraine — cortical eigenvectors, subcortical-weighted manifolds, neurotransmitter decoding, and prediction of headache frequency.

This repository accompanies the study *"Whole-brain functional gradients reveal cortical and subcortical alterations in patients with episodic migraine"* (Lee CH, Park H, Lee MJ, Park B; **Human Brain Mapping**, 2023;44(6):2224–2233). Resting-state functional connectivity of patients with episodic migraine and matched healthy controls is projected onto a low-dimensional manifold with diffusion map embedding, yielding **cortical eigenvectors** and **subcortical-weighted manifolds**. Between-group differences are tested with a multivariate surface-based linear model, biologically decoded against PET neurotransmitter receptor/transporter maps, and finally used to predict each patient's monthly headache frequency with supervised machine learning.

- **Paper (publisher):** https://onlinelibrary.wiley.com/doi/full/10.1002/hbm.26204
- **Open access (PMC):** https://pmc.ncbi.nlm.nih.gov/articles/PMC10028679/
- **DOI:** [10.1002/hbm.26204](https://doi.org/10.1002/hbm.26204)
- **Status:** Published (2023)

> **Reproducibility.** The imaging and clinical data cannot be shared publicly (sensitive human-subject data; see [Data availability](#data-availability)). This repository instead provides the full analytical pipeline as four annotated Jupyter notebooks — one per main figure — so that every step from connectivity matrix construction to the reported statistics is transparent and independently verifiable on equivalent data.

## Overview

Migraine is increasingly understood as a disorder of distributed brain network organization rather than of isolated regions. Conventional edge-wise connectivity analyses describe individual connections but not the *macroscale hierarchy* along which those connections are organized. This study applies manifold (gradient) learning to whole-brain functional connectivity to capture that hierarchy, and extends it to subcortical structures — which are hard to study with cortex-only gradients — via subcortical-weighted manifolds.

- **Data source:** Resting-state fMRI and clinical assessments collected at Samsung Medical Center (August 2017 – July 2018).
- **Cohort:** 50 patients with episodic migraine and 50 sex- and age-matched healthy controls (100 participants in total).
- **Cortical parcellation:** Schaefer 7-network atlas, 200 parcels.
- **Subcortical regions:** 9 regions after averaging left and right hemispheres — accumbens, amygdala, caudate, hippocampus, pallidum, putamen, thalamus, cerebellum, brainstem.
- **Manifold:** diffusion map embedding (normalized angle kernel) on the group-level connectome, Procrustes-aligned to an HCP template; the first three eigenvectors are carried through all analyses.
- **Main findings:** patients showed significant eigenvector differences concentrated in **sensory/motor and limbic cortices**; among subcortical structures the **amygdala** showed the strongest alteration; the cortical effect map was spatially associated with several neurotransmitter receptor/transporter distributions; and the same manifold features predicted **monthly headache frequency** at a moderate accuracy.

## Analysis pipeline

Each notebook is self-contained and reproduces one main figure of the paper.

| Notebook | Figure | What it does |
|---|---|---|
| [`fig1.ipynb`](fig1.ipynb) | Figure 1 | Cortico-cortical connectivity → cortical eigenvectors → between-group comparison → significance map on the surface + 7-network radar (spider) plot |
| [`fig2.ipynb`](fig2.ipynb) | Figure 2 | Subcortico-cortical connectivity → subcortical-weighted manifolds → between-group comparison for the 9 subcortical regions + 3D scatter plots |
| [`fig3.ipynb`](fig3.ipynb) | Figure 3 | Spatial correlation between the cortical group-difference *t*-map and 19 PET neurotransmitter receptor/transporter maps, with spin permutation testing |
| [`fig4.ipynb`](fig4.ipynb) | Figure 4 | Association and supervised prediction of monthly headache frequency from cortical eigenvectors and subcortical-weighted manifolds |

**1 — Cortical eigenvectors (`fig1.ipynb`)**

1. Build each participant's 200 × 200 cortico-cortical connectivity matrix (Pearson correlation), Fisher *z*-transform it, and zero the diagonal.
2. Average the 100 individual matrices into a group-level connectome.
3. Fit `GradientMaps(kernel='normalized_angle', approach='dm')` and align the group connectome to the HCP template eigenvectors (`data/gradients_FC.mat`) with Procrustes rotation (`n_components=5`).
4. Align every individual connectome to the HCP-aligned group manifold → individual eigenvectors (first 3 components, 200 nodes each).
5. Fit a multivariate linear model (BrainStat `SLM`) with **age + sex + group** and the contrast *patients − controls*; correct with FDR and map surviving *t*-values onto the cortical surface.
6. Export the unthresholded *t*-values to `fig1/total_t_value.csv` for the neurotransmitter analysis in `fig3.ipynb`, and summarize them per Yeo 7 network as a radar plot.

**2 — Subcortical-weighted manifolds (`fig2.ipynb`)**

1. Build the 17 × 17 subcortico-subcortical and the 200 × 17 subcortico-cortical connectivity matrices per participant; average left/right regions into a 200 × 9 matrix.
2. Element-wise multiply each of the three individual cortical eigenvectors with the subcortico-cortical connectivity profile of each subcortical region → a 3 × 9 × 200 subcortical-weighted manifold per participant.
3. Take the degree centrality (mean across the 200 cortical nodes) to obtain a 3 × 9 summary per participant, and run the same age/sex-adjusted group contrast for each of the 9 regions (FDR-corrected).

**3 — Neurotransmitter association (`fig3.ipynb`)**

1. Load 19 receptor/transporter PET maps parcellated to Schaefer-200 (`data/receptor.mat`).
2. Correlate each map with the cortical group-difference *t*-map (Pearson).
3. Assess significance with **spin permutation tests** (`enigmatoolbox.permutation_testing.spin_test`, `n_rot=1000`) that preserve spatial autocorrelation, repeated over 100 trials; take the most frequent FDR-corrected *p*-value across trials as the final *p*-value.
4. Report the receptors surviving correction and render the receptor atlases beside the correlation bar plot.

**4 — Headache-frequency prediction (`fig4.ipynb`)**

1. Assemble the feature matrix: 600 cortical eigenvector features (3 eigenvectors × 200 nodes) + 27 subcortical-weighted manifold features (3 eigenvectors × 9 regions) = **627 features**.
2. Regress out age and sex from every feature (reduced model: intercept + sex + age) and keep the residuals.
3. Fit an OLS model against monthly headache frequency (`HA_freq_m_fMRI`) for the association analysis.
4. Prediction: **nested cross-validation** — outer `RepeatedKFold(n_splits=5, n_repeats=100, random_state=1219)`, inner `KFold(n_splits=5, shuffle=True)` — with Lasso/Ridge (α grid 0.01–0.10) used for feature selection, refitting a linear model on the selected features.
5. Report accuracy across the 100 trials as Pearson *r*, ICC, MAE and RMSE, with a 1,000-fold permutation test on the actual-vs-predicted correlation; map how often each cortical node and subcortical region was selected across the 100 trials onto the surface.

## Repository structure

```
migraine_gradient
 ├── fig1.ipynb     # cortical eigenvectors + between-group comparison (Figure 1)
 ├── fig2.ipynb     # subcortical-weighted manifolds (Figure 2)
 ├── fig3.ipynb     # neurotransmitter receptor association + spin tests (Figure 3)
 ├── fig4.ipynb     # headache-frequency association and prediction (Figure 4)
 └── data/          # not included in this repository — see Data availability
```

The notebooks expect the following local `data/` layout (paths are relative to the notebook):

```
data
 ├── schaefer200
 │    ├── cortex     # g{1..100}_200.csv      — 200-parcel cortical time series per participant
 │    └── subcortex  # gp{1..50}_200.csv      — 17-region subcortical time series, patients
 │                   # gn{1..50}_200.csv      — 17-region subcortical time series, controls
 ├── gradients_FC.mat   # HCP group-level template eigenvectors (variable: gm_mean)
 ├── receptor.mat       # 19 PET receptor/transporter maps parcellated to Schaefer-200
 └── <clinical>.xlsx    # demographics and clinical scores (Sex, Age, group, HA_freq_m_fMRI, ...)
```

Outputs are written to `fig1/`, `fig3/permutation_result/` and `fig4/ha_freq/`; create these directories before re-running the notebooks. Participants 1–50 are patients and 51–100 are controls throughout.

## Environment setup

The analysis runs in Jupyter with a standard scientific Python stack plus three neuroimaging toolboxes:

```bash
python -m venv .venv && source .venv/bin/activate
pip install jupyter numpy pandas scipy matplotlib seaborn openpyxl \
            scikit-learn statsmodels mlxtend \
            nibabel nilearn brainspace brainstat enigmatoolbox
```

`brainspace` requires VTK for surface rendering; `plot_hemispheres` and `plot_subcortical` need a display (use a virtual framebuffer, e.g. `xvfb-run`, on a headless machine). BrainSpace also downloads the Schaefer-200 parcellation and the Conte69 surfaces on first use, so the first run needs network access.

## Key libraries

| Library | Role in the pipeline |
|---|---|
| [BrainSpace](https://brainspace.readthedocs.io) | `GradientMaps` (diffusion map embedding, Procrustes alignment), parcellation/surface loading, `plot_hemispheres` |
| [BrainStat](https://brainstat.readthedocs.io) | `FixedEffect` / `SLM` multivariate surface linear models with FDR correction |
| [ENIGMA TOOLBOX](https://enigma-toolbox.readthedocs.io) | `spin_test` spatial permutation, `plot_subcortical` |
| nilearn / nibabel | `ConnectivityMeasure` correlation matrices, surface handling |
| scikit-learn | `Lasso`, `Ridge`, `LinearRegression`, `KFold` / `RepeatedKFold` nested cross-validation |
| statsmodels | OLS association models, FDR correction (`multitest.fdrcorrection`) |

No pinned environment file is provided; the notebooks were developed on Python 3 with the library versions current at the time of publication (2022–2023). Note that `fig2.ipynb` uses `fig.gca(projection='3d')`, which was removed in Matplotlib 3.6 — use `fig.add_subplot(projection='3d')` on newer versions.

## Reproducing the analysis

1. Place the data under `data/` following the layout above.
2. Run the notebooks in order: `fig1.ipynb` → `fig2.ipynb` → `fig3.ipynb` → `fig4.ipynb`. `fig3.ipynb` depends on `fig1/total_t_value.csv` written by `fig1.ipynb`.
3. Random seeds are fixed where a random state is required: `random_state=0` for the gradient decomposition, `random_state=1219` for the outer repeated cross-validation, and `np.random.seed(0)` for the inner folds. Spin and permutation tests remain stochastic and are therefore repeated 100 times and summarized.

## Data availability

The imaging and clinical data cannot be shared publicly because they contain sensitive information about human subjects. Data may be available to researchers who meet the criteria for access to confidential data, subject to institutional approval. Requests should be directed to the corresponding author of the paper. The HCP template gradients and the PET receptor maps used here derive from publicly released resources described in the paper.

## Citation

If you use this code, please cite:

> Lee CH, Park H, Lee MJ, Park B. *Whole-brain functional gradients reveal cortical and subcortical alterations in patients with episodic migraine.* Human Brain Mapping. 2023;44(6):2224–2233. doi:10.1002/hbm.26204

```bibtex
@article{lee2023migraine,
  title   = {Whole-brain functional gradients reveal cortical and subcortical alterations in patients with episodic migraine},
  author  = {Lee, C. H. and Park, H. and Lee, M. J. and Park, B.},
  journal = {Human Brain Mapping},
  volume  = {44},
  number  = {6},
  pages   = {2224--2233},
  year    = {2023},
  doi     = {10.1002/hbm.26204}
}
```

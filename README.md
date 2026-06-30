# Beyond the Cut-off: Modelling the Glucose–Atherosclerosis Continuum

A small data science project built around a question that keeps coming up in clinical chemistry: why do we treat a fasting glucose of 125 mg/dL as "fine" and 126 mg/dL as "diabetic," when the vascular damage that leads to heart attacks and strokes doesn't actually respect that line?

This repository pulls real patient data from NHANES (the CDC's National Health and Nutrition Examination Survey), runs it through proper biochemical validation, and builds a predictive model that treats cardiovascular risk as what it actually is — a continuum, not a yes/no switch.

## Why this exists

Diabetes diagnosis relies on hard cut-offs (FPG ≥ 126 mg/dL, HbA1c ≥ 6.5%). These thresholds are clinically useful but statistically arbitrary in one specific sense: they were calibrated against the point where diabetic retinopathy risk spikes, not against cardiovascular risk. Plenty of damage to the vascular endothelium is already happening in the "pre-diabetic" range, well before anyone crosses 126.

This project tries to make that continuum visible and quantifiable:

- it pools several NHANES cycles into one larger, more representative dataset
- it applies the actual clinical formulas correctly (Friedewald, Pooled Cohort Equations) instead of approximating them
- it trains a Random Forest to estimate 10-year cardiovascular risk for patients the classic equations can't cover (anyone under 40 or over 79)
- it cross-checks the model against Leave-One-Out validation, not just a single train/test split

The result isn't meant to replace clinical judgement — it's a teaching exercise in why "biology doesn't read cut-off tables."

## What's in here

| File / folder | What it does |
|---|---|
| `notebooks/principal_analysis.ipynb` | The whole pipeline: data loading, cleaning, biochemical checks, modelling, and plots |
| `data/` | Raw NHANES `.XPT` files, organised by cycle (you provide these — see below) |
| `results/imgs/` | Output figures get saved here automatically |

## Data: NHANES, multiple cycles

The notebook is built to pull from several NHANES survey cycles at once rather than relying on a single year, since pooling cycles gives the model a lot more patients to learn from and reduces the risk of one cycle's quirks skewing the results.

Each cycle needs its own folder, named by year range, sitting under `data/`:

```
data/
├── 2011-2012/
│   ├── DEMO_G.XPT
│   ├── GLU_G.XPT
│   ├── GHB_G.XPT
│   ├── TCHOL_G.XPT
│   ├── HDL_G.XPT
│   ├── TRIGLY_G.XPT
│   ├── BPX_G.XPT
│   ├── BMX_G.XPT
│   ├── HSCRP_G.XPT
│   ├── SMQ_G.XPT
│   └── BPQ_G.XPT
├── 2013-2014/   (same files, suffix _H)
├── 2015-2016/   (same files, suffix _I)
├── 2017-2018/   (same files, suffix _J)
└── 2019-2020/   (same files, suffix _K)

results/
    └── imgs/    (created automatically by the notebook)
```

You don't need all five cycles — the loader checks which folders exist and quietly skips whatever's missing, so even a single cycle (e.g. just `2017-2018/`) will run fine. More cycles just means a bigger, sturdier dataset.

All files come straight from the CDC's NHANES archive: https://wwwn.cdc.gov/nchs/nhanes/

A quick note on file naming: NHANES changes the suffix every cycle (`_G`, `_H`, `_I`, `_J`, `_K`), and a handful of variable names shift between cycles too (HDL cholesterol, for instance, was coded `LBXHDD` in early cycles and `LBDHDD` later on). The loader already accounts for these aliases, so you shouldn't need to rename anything manually — just keep the original CDC filenames.

## Setting up

You'll need Python 3.10+ and the following packages:

```bash
pip install pandas numpy matplotlib seaborn scipy scikit-learn pyreadstat jupyter
```

Once dependencies are installed:

```bash
jupyter notebook notebooks/principal_analysis.ipynb
```

Run the cells top to bottom — later cells depend on dataframes built earlier (the merged dataset, the Friedewald LDL values, the PCE risk scores, etc.), so skipping around will break things.

## What the pipeline actually does

**1. Loads and pools multiple NHANES cycles**, namespacing patient IDs by year so the same SEQN from different cycles doesn't get treated as the same person.

**2. Runs biochemical quality control** — most notably, LDL cholesterol is recalculated using the Friedewald formula (`TC − HDL − TG/5`) rather than trusting the raw NHANES value blindly, and any patient with triglycerides above 400 mg/dL gets flagged, since the formula breaks down at that point and shouldn't be trusted.

**3. Classifies glycaemic status** using both FPG and HbA1c, treating HbA1c as the more reliable signal where both are present (it reflects a 2–3 month average rather than a single noisy measurement).

**4. Computes 10-year cardiovascular risk** using the Pooled Cohort Equations, with the correct demographic coefficient set applied per patient (white/black, male/female) rather than a single generic formula across everyone.

**5. Stratifies borderline cases with hs-CRP** — patients sitting right around the risk threshold get reclassified if their inflammatory marker suggests otherwise, following the same logic as the Reynolds Risk Score.

**6. Simulates glycaemic trajectories** — since NHANES is cross-sectional, the notebook approximates a 5-year glucose trend per patient to give the model a sense of *direction*, not just a snapshot.

**7. Trains a Random Forest** on patients for whom the Pooled Cohort Equations are technically valid (ages 40–79), then extends predictions to everyone else — including younger and older patients the classic clinical formula simply isn't built to handle.

**8. Validates rigorously**, comparing 5-fold cross-validation against Leave-One-Out cross-validation on a stratified subsample, since LOO is too expensive to run on the full pooled dataset but gives a more honest estimate of how the model generalises.

## A few things worth knowing before you dig in

- The Pooled Cohort Equations are only validated for ages 40–79. Risk scores outside that range are left blank by design — that gap is exactly what the Random Forest is there to fill.
- Patients outside the "white" / "black" PCE categories fall back to the white coefficient set, since that's the closest approximation the original equations support. This is a known limitation of the PCE itself, not something introduced here.
- The glycaemic trajectory data is simulated, not observed — NHANES doesn't follow the same patient across cycles, so there's no way to get a *real* longitudinal slope from this dataset alone. It's a reasonable stand-in for a teaching project, but shouldn't be mistaken for real patient history.
- None of this is a diagnostic tool. It's a sandbox for understanding how lab medicine, biostatistics, and machine learning intersect — not something to plug into a clinical workflow.

## References

- American Diabetes Association, *Standards of Care in Diabetes* (2024)
- Goff DC et al., *2013 ACC/AHA Guideline on the Assessment of Cardiovascular Risk*, Circulation (2014)
- Friedewald WT, Levy RI, Fredrickson DS, *Estimation of the concentration of low-density lipoprotein cholesterol in plasma*, Clinical Chemistry (1972)
- NCEP ATP III Guidelines
- NHANES survey documentation, CDC/NCHS
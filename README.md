# COMPAS Assignment 5 – ML Security & Abuse Pathways

## Purpose of the Analysis

Hi Professor, this assignment implements a full adversarial security audit of the COMPAS recidivism prediction pipeline. The goal is to move beyond reliability testing and stress-test the model against three classes of adversarial attacks: evasion, poisoning, and membership inference. The audit asks not just *how well* the model performs, but *how it fails under attack*, *whether fairness can be silently corrupted*, and *whether training data can be inferred from model outputs* — the three threat dimensions that separate responsible deployment from operationally naïve ML.

> **Note:** This notebook is cumulative. Assignments 1–4 occupy cells 1–63. **Assignment 5 begins at cell 64.** All prior cells must be run first as they build the `df` object, feature engineering, and model pipeline that Assignment 5 depends on.

---

## What This Project Does

This project extends the Lecture 5 COMPAS security pipeline into a structured adversarial audit. Two models are evaluated — Logistic Regression (LR) and Gradient-Boosted Trees (GBT) — across three attack dimensions:

| Part | Attack Type | Core Question |
|------|-------------|---------------|
| 1 | PGD Evasion Audit | Are LR and GBT equally vulnerable to adversarial input perturbation? |
| 2 | Poisoning Loop with Fairness Monitoring | Can label-flip poisoning silently corrupt fairness metrics without triggering AUC or PSI alerts? |
| 3 | Membership Inference Depth | Do the models leak information about their training data? |
| 4 | Reflection | What is the highest-risk finding and how should it be mitigated? |

---

## Python Libraries Used

- `pandas`, `numpy` — data manipulation
- `matplotlib` — visualization
- `scikit-learn` — model training, metrics, shadow models
- `scipy` — statistical testing

---

## Key Tasks Completed

**Part 1 — PGD Evasion Audit**
- Implemented PGD for LR using analytical gradient (sign of coefficients)
- Implemented PGD for GBT using finite-difference gradient approximation
- Swept ε ∈ {0.25, 0.5, 1.0, 2.0} for both models
- Reported FPR by race and AIR at each ε level
- Identified ε at which AIR crosses 0.80 (if any)
- Written paragraph comparing model vulnerability and model-selection implications

**Part 2 — Poisoning Loop with Fairness Monitoring**
- Extended the lecture's label-flip loop to also target Caucasian defendants
- Swept poison rates 0%–30% for both target-race variants
- Plotted AUC and AIR degradation curves on the same axes
- Identified the stealth zone: poison rates where AUC drop ≤ 2pp while AIR moves outside [0.80, 1.25]
- Computed PSI per feature at each poison rate to test whether a drift monitor would detect the attack
- Verified that PSI = 0.0 on all features at all poison rates (label-flip is invisible to feature-distribution monitoring)

**Part 3 — Membership Inference Depth**
- Trained 10 shadow GBT models to build a meta-classifier for membership inference
- Computed shadow-model MI AUC for both LR and GBT
- Plotted confidence-gap histograms side by side
- Tested whether generalization gap predicts MI AUC across the two models
- Swept L₂ regularization on LR (C ∈ {0.01, 0.1, 1.0, 10.0}), recomputed MI AUC, and plotted MI AUC vs. C
- Assessed privacy–performance–fairness tradeoff across C values

**Part 4 — Reflection**
- Identified the single highest-risk finding across all three parts
- Proposed one proactive and one reactive mitigation
- Quantified each mitigation's effect from experimental results
- Discussed disparate impact the mitigations may introduce on either racial group

---

## Key Findings

### Part 1 — LR Is Highly Vulnerable; GBT Is Completely Immune
The two models are at opposite extremes. Under PGD, LR's FPR for African-American defendants rises from 0.281 to 1.000 at ε = 2.0, and Caucasian FPR reaches 1.000 as well — the model collapses into flagging everyone as high-risk. AIR converges toward 1.0 but never falls below 0.80. GBT is completely unaffected at every ε value (FPR and AIR remain fixed at 0.317, 0.178, and 1.782), because finite-difference perturbations do not push samples across tree leaf boundaries. GBT's robustness is structurally significant for model selection, though its baseline disparity (AIR = 1.782) remains a governance concern.

### Part 2 — Label-Flip Poisoning Is Invisible to Both AUC and PSI Monitoring
The African-American-targeted attack raises AIR from 1.961 to 3.010 at 30% poison rate (flipping 345 labels) while AUC drops only 0.34pp — undetectable by performance monitoring. The Caucasian-targeted attack has minimal effect on AIR (stays 1.84–2.04). PSI on all 7 features = 0.0 at all poison rates for both variants, confirming that a PSI-based drift monitor would never trigger an alert. This reveals a fundamental governance gap: PSI is designed for covariate shift and is completely blind to label corruption.

### Part 3 — Neither Model Leaks Meaningful Training Data
Both LR and GBT produce MI AUC ≈ 0.50 (effectively random), indicating no meaningful membership inference leakage. LR has a negative generalization gap (−0.008), meaning it generalizes better on test data than training data — which explains its near-zero privacy risk. GBT's larger generalization gap (+0.080) yields a slightly higher MI AUC (0.500 vs 0.494), directionally consistent with the overfitting–privacy hypothesis. The L₂ sweep on LR (C = 0.01 to 10.0) produces MI AUC values of 0.493–0.495 across all C values, with no meaningful tradeoff between privacy, performance, or fairness — regularization is a zero-cost control for this model.

### Part 4 — Highest Risk: Silent Fairness Corruption via Poisoning
The AA-targeted label-flip attack is the most dangerous finding: high impact (AIR +1.05 at 30%), low detectability (AUC −0.34pp, PSI = 0.0), and low attacker effort (345 flipped labels out of 1,151 eligible). The proactive mitigation is label auditing — flagging when the high-risk rate for any group drops more than 5% between training cycles. The reactive mitigation is AIR change monitoring — alerting when AIR shifts more than 0.20 between model versions, which avoids false-alarming on the pre-existing baseline disparity.

---

## Instructions to Reproduce the Results

1. Open `COMPAS_Assignment5_HassanAlshamrani.ipynb` in Google Colab
2. Run **all cells from top to bottom** — cells 1–63 set up prior assignments; Assignment 5 starts at **cell 64**
3. Note: Part 1 GBT sweep takes ~2–5 minutes due to finite-difference gradient computation; Part 3 MI pipeline takes ~10–15 minutes for shadow model training
4. No additional data downloads required — data is fetched directly from the ProPublica GitHub repository

---

## Overall Audit Verdict

The COMPAS pipeline faces asymmetric attack risk. LR is operationally fragile under evasion attacks, while GBT is structurally resistant. Both models are free from meaningful membership inference leakage. The most serious threat is label-flip poisoning targeting African-American defendants: an insider or compromised data pipeline can triple the existing racial disparity while AUC monitoring reports a healthy model and PSI drift monitors show no alert. Responsible deployment requires moving beyond AUC and PSI as the sole governance signals — direct fairness monitoring (AIR change tracking) and data-pipeline integrity controls (label distribution auditing per group) are necessary conditions for safe operation.

---

Thanks, Professor.  
**Hassan Alshamrani**

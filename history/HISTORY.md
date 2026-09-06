# Project history — how the study was built, iteration by iteration

The final notebook is [`../sar_assisted_landcover.ipynb`](../sar_assisted_landcover.ipynb).
This folder keeps every earlier executed version, with outputs intact, so the path from a simple
baseline to the final analysis is visible — including the things that did not work.

Each version was fully executed before being superseded; the numbers quoted below are the ones
that version actually produced.

---

## v1 — [`v1_baseline_A_B_C.ipynb`](v1_baseline_A_B_C.ipynb)
**The required minimum baseline plus the first SAR comparison.**

- Dataset selection, cleaning, multi-label formulation, blob-based cloud simulation.
- Three SimpleCNN models: **A** (clean optical), **B** (mask-augmented optical),
  **C** (early fusion +SAR).
- Trained first at **14 epochs**, then re-run at **30** after the training curves showed the
  models were still improving (both result files are kept: `results_14ep.csv`,
  `results_30ep_3models.csv`).

**Result:** A collapses under masking (0.753 → 0.41 at 75% coverage); B recovers it; C only
clearly beats B at 100% coverage. Mid-coverage gaps of ±0.5 pts were within noise — and the
14→30-epoch comparison showed the 50% gap *flipping sign* between runs, which is what motivated
the statistical machinery added in v2.

## v2 — [`v2_late_fusion_and_bootstrap.ipynb`](v2_late_fusion_and_bootstrap.ipynb)
**Statistical rigour, plus two more fusion designs.**

- Added **paired bootstrap confidence intervals** (1,000 shared resamples) so every claimed gap
  gets an error bar.
- Added **model D** (dual-encoder late fusion) and **model E** (deep ResNet-style dual encoder,
  ~14 M params) — the "does a bigger model help?" test.

**Result:** SAR gains significant only at ≥90% coverage. Model E was significantly *worse* than
the 1.9 M-param SimpleCNN below 90% — the first sign that capacity was not the bottleneck.

## v3 — [`v3_seeds_class_conditional_hybrid.ipynb`](v3_seeds_class_conditional_hybrid.ipynb)
**Training-noise control and the class-conditional insight.**

- **Seed replicates**: B, C, D retrained with seeds 43 and 44, evaluation masks held fixed.
- **Class-conditional analysis**: classes grouped by expected C-band signature
  (water / urban / agriculture / forest), each with its own paired bootstrap CIs.
- **Model H**: selective per-class fusion, choosing between the optical and SAR model per class
  using validation only.

**Result:** SAR deltas positive in 9/9 model×seed comparisons at 75% and 90%. Water gains are
significant at *every* coverage level ≥25%, while agriculture is significantly *hurt* at 90% —
the macro average had been hiding two opposite effects. H beat both of its parents.

## v4 — [`v4_architecture_matrix.ipynb`](v4_architecture_matrix.ipynb)
**The full 4 architectures × 3 fusion variants matrix.**

- Added **VGG11**, **ResNet18-from-scratch** and **ImageNet-pretrained ResNet18** families, each
  in B / C / D variants; vectorised the bootstrap so CIs for all 13 models cost seconds.
- Added **label accuracy** and **subset accuracy** alongside macro-F1 and mAP.

**Result:** VGG11 extracts significant SAR gains from 50% coverage upward — the first
architecture to beat the optical baseline in the mid-coverage regime — and VGG11-D became the
best model in the degraded regime. ImageNet pretraining **failed**: significantly worse than the
simple baseline everywhere, memorising the training set by epoch 4.

## Final — [`../sar_assisted_landcover.ipynb`](../sar_assisted_landcover.ipynb)
**Diagnosis of the architecture result, and the formulation evidence.**

- **Capacity diagnosis**: train-vs-test gap per family, showing the gap grows with parameter
  count while test performance does not.
- **Capacity sweep**: VGG11 and ResNet18 retrained at ¼ and ½ width to test whether shrinking
  the same architectures fixes the overfitting.
- Explicit evidence for the multi-label formulation, and a complete experiment inventory.

**Result:** narrowing made the architectures *monotonically worse*, and VGG11 at half the
baseline's parameter count still lost to it — so parameter count is not the explanatory
variable. The likely cause is the downsampling schedule versus the 120 px input size (SimpleCNN
reaches global pooling at 7×7; VGG11 at 3×3; ResNet18 at 4×4).

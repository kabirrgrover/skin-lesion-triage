# Skin Lesion Triage

AI-assisted clinical decision support for primary care skin lesion triage. Uses Google's MedSigLIP vision encoder and MedGemma language model to classify skin lesions from smartphone photos, with calibrated confidence scores, visual explanations, and explicit fairness evaluation across Fitzpatrick skin types I through VI.

**Status:** Research prototype. 10/11 evaluation criteria passing. Active research on specificity improvement.

---

## The Problem

Skin cancer affects over 5 million Americans annually. Melanoma, the deadliest form, has a five-year survival rate exceeding 99% when caught early -- but that drops to roughly 30% once it has metastasized.

Two problems make early detection difficult:

**The access gap.** Primary care physicians are the first point of contact for most skin concerns, but achieve only about 45% diagnostic accuracy for melanoma, compared to 97% for dermatologists. Dermatologist shortages -- particularly in rural and underserved areas -- mean patients often wait weeks for specialist evaluation.

**The fairness gap.** Most dermatology AI systems are trained on datasets composed predominantly of lighter skin tones (Fitzpatrick I-III). These systems perform measurably worse on darker skin, compounding existing health disparities in dermatology.

This project addresses both problems: an AI tool that works on clinical smartphone images, is calibrated for reliable confidence scores, and is evaluated across the full Fitzpatrick spectrum (I-VI).

---

## Architecture

```
Input Image (448x448)
       |
       v
+-----------------------+
|  MedSigLIP-448        |  Frozen vision encoder (~429M params)
|  Vision Encoder       |  Pretrained on medical images
+----------+------------+
           |
           v
+-----------------------+
|  Classification Head  |  Trainable (596K params)
|  LayerNorm -> 512     |  Focal Loss, 2x melanoma weight
|  GELU -> 7 classes    |  Temperature-calibrated output
+----------+------------+
           |
     +-----+------+
     |             |
     v             v
  Grad-CAM      MedGemma-4B
  heatmap       clinical explanation
```

The encoder is frozen. Only the 596K-parameter classification head is trained -- 0.14% of total model parameters. This prevents overfitting on datasets of ~10K images and preserves MedSigLIP's pretrained medical knowledge.

### Seven-Class Output

| Class | Description | Clinical Action |
|-------|-------------|-----------------|
| mel | Melanoma | Urgent referral |
| bcc | Basal cell carcinoma | Referral within 2 weeks |
| akiec | Actinic keratosis / SCC | Referral within 2-4 weeks |
| bkl | Benign keratosis | Routine monitoring |
| df | Dermatofibroma | Routine monitoring |
| nv | Melanocytic nevus | Routine monitoring |
| vasc | Vascular lesion | Routine monitoring |

### Calibration and Safety

- **Temperature scaling** (T=0.923) applied to all output probabilities. Post-calibration Expected Calibration Error: 0.067.
- **Melanoma safety threshold** (P(mel) >= 0.24): overrides argmax classification to flag potential melanomas even when another class has higher probability. Raises melanoma recall from 81.8% to 90.9% on clinical images.

---

## Training

### Datasets

| Dataset | Type | Images | Population | Role |
|---------|------|--------|------------|------|
| HAM10000 | Dermoscopic | 10,015 | Fitzpatrick I-III | Primary training data |
| PAD-UFES-20 | Smartphone clinical | ~2,100 | Fitzpatrick I-V (Brazil) | Domain adaptation |
| DDI | Clinical photography | 656 | Fitzpatrick I-VI (balanced) | Fairness evaluation |

### Strategy

Mixed training on HAM10000 + PAD-UFES-20 with 3x oversampling of clinical images. Focal Loss (gamma=2.0) with 2x melanoma class weight. AdamW optimizer, cosine annealing schedule, 30 epochs.

This bridged a 30-point domain gap between dermoscopic and clinical image performance, reducing the melanoma recall gap from 31 points to 4 points.

---

## Results

### Clinical Image Performance (PAD-UFES-20)

| Metric | HAM10000-Only Baseline | Mixed Training (M2) |
|--------|----------------------|---------------------|
| Balanced Accuracy | 49.9% | **81.8%** |
| Melanoma Recall | 54.5% | **90.9%** (with safety threshold) |
| Melanoma AUC | -- | 0.991 |

### Cross-Skin-Tone Fairness (DDI -- 656 images, Fitzpatrick I-VI)

Evaluated on the Diverse Dermatology Images dataset, which was specifically designed to test fairness across skin tones. DDI contains 78 dermatological conditions, of which only 14 overlap with our 7 training classes -- making it a deliberately challenging benchmark.

**Binary triage performance** (malignant vs. benign, ensemble of MedSigLIP + DermLIP encoders):

| Metric | Value | Target |
|--------|-------|--------|
| AUC | 0.748 | -- |
| Sensitivity (malignant detection) | 86.0% | >= 90% |
| Specificity | 44.5% | >= 60% |
| Referral rate | 63.4% | <= 45% |
| Fairness gap (max AUC difference across groups) | 11.1 points | < 15 points |

**Per-skin-tone breakdown:**

| Fitzpatrick Group | n | AUC | Sensitivity | Specificity |
|--------------------|---|-----|-------------|-------------|
| I-II (lighter) | 208 | 0.696 | 87.3% | 46.5% |
| III-IV (medium) | 241 | 0.792 | 90.3% | 38.0% |
| V-VI (darker) | 207 | 0.702 | 95.5% | 35.0% |

The model maintains high sensitivity across all skin tone groups. V-VI sensitivity (95.5%) is the highest of any group -- an inversion of the typical bias pattern in dermatology AI.

### What the Numbers Mean

The model catches 86% of malignancies on DDI while referring 63% of all cases. The high referral rate is driven by out-of-distribution conditions: DDI is 82% conditions the model has never seen (psoriasis, eczema, warts, etc.), and roughly 56% of false positives come from these OOD conditions.

On in-distribution conditions alone, specificity rises to 54%. On a realistic clinical distribution -- which would skew toward common conditions like nevi and seborrheic keratoses -- effective specificity would likely be substantially higher.

Even at current DDI performance, the model nearly doubles PCP melanoma detection accuracy (86% vs. ~45% baseline).

---

## Experimental Journey

This project involved systematic, hypothesis-driven experimentation across 10+ experiments. The negative results were as informative as the positive ones.

| Experiment | Approach | Outcome | Key Finding |
|------------|----------|---------|-------------|
| Baseline (NB01) | HAM10000 only | 85.7% mel recall (dermoscopic) | Strong in-domain, fails on clinical images |
| Mixed training (NB02) | HAM + PAD-UFES | 81.8% clinical mel recall | Domain gap reduced 87% |
| 8-class model (NB06) | Added "other" class via Fitzpatrick17k | Rejected | "Other" class absorbed malignant probability mass. Sensitivity dropped 90.6% to 75.4%. |
| OOD detection (NB07) | Energy scores + Mahalanobis distance | Failed | MedSigLIP features do not separate in-distribution from out-of-distribution. No viable threshold. |
| DermLIP encoder (NB08) | Derm-specific vision encoder (PanDerm) | Complementary | Opposite failure mode: high specificity (63.5%), low sensitivity (62.6%). |
| Feature fusion (NB09) | MedSigLIP + DermLIP concatenation | Does not meet targets | Fusion AUC 0.742. DermLIP dilutes V-VI sensitivity. Ensemble (alpha=0.6) is best at AUC 0.748. |
| Binary gate + SCIN (NB10) | Binary malignant/benign with smartphone OOD data | Failed | SCIN domain mismatch (cosine similarity 0.60 to DDI) contaminated decision boundary. Sensitivity collapsed to 65%. |
| Binary ablation (NB10b) | Binary gate, HAM+PAD only | Matched baseline | Confirmed SCIN was the cause, not binary architecture. Binary training on frozen features works. |
| Binary gate + Fitz17k (NB10c) | Binary with domain-aligned clinical OOD data | In progress | Fitzpatrick17k shares DDI's clinical photography domain. Binary framing avoids the probability mass absorption that failed in NB06. |

**Core finding:** The remaining performance gap is a data coverage problem, not an architecture problem. Roughly 56% of false positives come from conditions the model has never seen. Adding the right out-of-distribution data matters more than changing the model architecture.

---

## Explainability

- **Grad-CAM heatmaps** show which image regions drive predictions, enabling clinicians to verify the model is attending to the lesion rather than background artifacts.
- **Monte Carlo Dropout** provides uncertainty quantification through multiple stochastic forward passes. High-uncertainty predictions are flagged for expert review.
- **Temperature-calibrated confidence scores** ensure predicted probabilities reflect true accuracy rates (ECE = 0.067).
- **MedGemma clinical explanations** generate natural language assessments describing visual features, differential considerations, and recommended actions.

---

## Limitations

1. **Retrospective validation only.** All results are from research datasets. Prospective clinical validation has not been performed.

2. **High referral rate on diverse benchmarks.** DDI referral rate is 63%, driven by out-of-distribution conditions the model was not trained on. Real-world clinical distribution would differ.

3. **HAM10000 data leakage.** The primary training dataset (HAM10000) has a documented 39.5% patient-level data leakage issue when using naive image-level splits. All fairness evaluations (DDI, experiments 6+) use lesion-aware honest splits.

4. **Single clinical training dataset.** PAD-UFES-20 is from a single institution in Brazil. Generalization to other populations, devices, and imaging conditions is not validated.

5. **Not a medical device.** This system is a research prototype and must not be used for clinical decision-making.

---

## Technical Details

| Component | Specification |
|-----------|--------------|
| Vision encoder | MedSigLIP-448 (google/medsiglip-448), frozen |
| Second encoder | DermLIP-PanDerm (512D, derm-specific), frozen |
| Classification head | LayerNorm, Dropout(0.3), Linear(1152, 512), GELU, Dropout(0.3), Linear(512, 7) |
| Trainable parameters | 596K |
| Input resolution | 448 x 448 pixels |
| Calibration | Temperature scaling, T=0.923, ECE=0.067 |
| Safety threshold | P(mel) >= 0.24 overrides argmax |
| Explainability | MedGemma-4B-IT for clinical text, Grad-CAM for visual |
| Training infrastructure | Google Colab, A100 GPU |
| Framework | PyTorch, HuggingFace Transformers, open_clip |

---

## Datasets and References

1. Tschandl P, et al. "The HAM10000 dataset, a large collection of multi-source dermatoscopic images of common pigmented skin lesions." *Scientific Data*, 2018.
2. Pacheco AG, et al. "PAD-UFES-20: A skin lesion dataset composed of patient data and clinical images collected from smartphones." *Data in Brief*, 2020.
3. Daneshjou R, et al. "Disparities in dermatology AI performance on a diverse, curated clinical image set." *Science Advances*, 2022.
4. Sellergren A, et al. "MedGemma Technical Report." *arXiv preprint arXiv:2507.05201*, 2025.
5. Guo C, et al. "On Calibration of Modern Neural Networks." *ICML*, 2017.
6. Selvaraju RR, et al. "Grad-CAM: Visual Explanations from Deep Networks via Gradient-based Localization." *ICCV*, 2017.
7. Yan S, et al. "PanDerm: A Foundation Model for Dermatology." *Nature Medicine*, 2025.
8. Groh M, et al. "Evaluating Deep Neural Networks Trained on Clinical Images in Dermatology with the Fitzpatrick 17k Dataset." *CVPR*, 2021.

---

## Disclaimer

This project is for research and educational purposes only. It is not a medical device, has not received regulatory approval, and must not be used to make clinical decisions. All results are from retrospective evaluation on research datasets. Always consult qualified healthcare professionals for medical concerns.

---

*Last updated: March 2026*

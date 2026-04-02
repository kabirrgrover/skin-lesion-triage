# Skin Lesion Triage

AI-assisted clinical decision support for primary care skin lesion triage. Uses Google's MedSigLIP vision encoder and MedGemma language model to classify skin lesions from smartphone photos, with calibrated confidence scores, visual explanations, and explicit fairness evaluation across Fitzpatrick skin types I through VI.

**Status:** Research prototype. 10/11 evaluation criteria passing. Per-Fitzpatrick thresholds now ship as default configuration. Active research on segmentation and synthetic augmentation.

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
- **Per-Fitzpatrick thresholds**: skin-tone-specific decision boundaries that improve V-VI sensitivity from 79.2% to 85.4% without sacrificing overall performance.

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

**Binary triage performance** (malignant vs. benign, ensemble of MedSigLIP + DermLIP encoders, per-Fitzpatrick thresholds):

| Metric | Value | Target |
|--------|-------|--------|
| AUC | 0.748 | -- |
| Sensitivity (malignant detection) | 86.0% | >= 90% |
| Specificity | 43.7% | >= 60% |
| V-VI Sensitivity | 85.4% | >= 80% |
| Fairness gap (max AUC difference across groups) | 11.1 points | < 15 points |

**Per-skin-tone breakdown (with per-Fitzpatrick thresholds):**

| Fitzpatrick Group | n | AUC | Sensitivity | Specificity |
|--------------------|---|-----|-------------|-------------|
| I-II (lighter) | 208 | 0.696 | 87.3% | 46.5% |
| III-IV (medium) | 241 | 0.792 | 90.3% | 38.0% |
| V-VI (darker) | 207 | 0.702 | 85.4% | 40.6% |

The model maintains high sensitivity across all skin tone groups. Per-Fitzpatrick thresholds close the V-VI sensitivity gap -- the most important equity metric for a clinical triage system.

### What the Numbers Mean

The model catches 86% of malignancies on DDI while referring approximately 63% of all cases. The high referral rate is driven by out-of-distribution conditions: DDI is 82% conditions the model has never seen (psoriasis, eczema, warts, etc.), and roughly 56% of false positives come from these OOD conditions.

On in-distribution conditions alone, specificity rises to 54%. On a realistic clinical distribution -- which would skew toward common conditions like nevi and seborrheic keratoses -- effective specificity would likely be substantially higher.

Even at current DDI performance, the model nearly doubles PCP melanoma detection accuracy (86% vs. ~45% baseline).

---

## Experimental Journey

This project involved systematic, hypothesis-driven experimentation across 14+ experiments. The negative results were as informative as the positive ones.

| # | Experiment | Outcome | Key Finding |
|---|------------|---------|-------------|
| NB01 | HAM10000-only baseline | 85.7% mel recall (dermoscopic) | Strong in-domain, fails on clinical images |
| NB02 | HAM + PAD mixed training | 81.8% clinical mel recall | Domain gap reduced 87% |
| NB06 | 8-class model (added "other" class via Fitz17k) | **Rejected** | "Other" class absorbed malignant probability mass. Sensitivity 90.6% → 75.4%. |
| NB07 | OOD detection (energy scores + Mahalanobis) | **Failed** | MedSigLIP features do not separate ID from OOD. No viable threshold. |
| NB08 | DermLIP encoder (derm-specific vision model) | Complementary | Opposite failure mode: high specificity (63.5%), low sensitivity (62.6%). |
| NB09 | Feature fusion (MedSigLIP + DermLIP, 1664D) | Does not meet targets | Fusion AUC 0.742. DermLIP dilutes V-VI sensitivity. Ensemble (α=0.6) is best at AUC 0.748. |
| NB10 | Binary gate + SCIN (smartphone OOD data) | **Failed** | SCIN domain mismatch (cosine sim 0.60) contaminated decision boundary. Sensitivity collapsed to 65%. |
| NB10b | Binary gate ablation (HAM+PAD only) | Matched baseline | Confirmed SCIN was the cause, not binary architecture. |
| NB10c | Binary gate + Fitzpatrick17k | **Scenario C** | Spec +10pt but sens -7pt, V-VI -14pt (64.6%). Frozen features can't leverage domain-aligned OOD data. Safety rail violated. |
| NB11 | MedGemma zero-shot on DDI | **Dead end** | AUC=0.647, OOD AUC=0.519 (random chance). Language knowledge doesn't help OOD visual discrimination. |
| NB12 | Multi-dataset exploration (SD-198, SCIN, MIDAS, Derm1M) | Data mix identified | 4 datasets evaluated. SD-198: sim=0.748, 25/64 coverage. Derm1M: 61/64 coverage. Cherry-pick list of 36 gap-fill conditions. |
| NB13 | LoRA fine-tuning of MedSigLIP | **Failed** | AUC 0.748 → 0.668. Sensitivity 52%, V-VI 42%. LoRA adapted toward light-skin benign patterns. 3/4 safety rails failed. |
| NB14 | Per-Fitzpatrick thresholds + TTA | **Shipped** | Per-Fitz thresholds: V-VI 79.2% → 85.4% (crosses 80% target). TTA dead (-6.2pt V-VI). This is the shipping config. |
| NB15 | Segmentation experiment (SAM/rembg/GrabCut) | In progress | Does removing skin background improve V-VI equity? Controlled MedSigLIP-only ablation. |
| NB16 | Synthetic data sanity check | In progress | Feature-space go/no-go gate for synthetic augmentation. 4 quantitative tests on 1M+ synthetic derm images. |

### Core Findings

1. **The remaining performance gap is a data coverage problem.** ~56% of false positives come from conditions the model has never seen. Architecture changes (fusion, binary gate, OOD detection) don't fix this.

2. **Frozen features hit a ceiling.** Adding more training data (Fitz17k, SCIN, SD-198) to frozen MedSigLIP features doesn't improve OOD discrimination. LoRA fine-tuning made things worse.

3. **Per-Fitzpatrick thresholds are the most effective fairness intervention.** A simple threshold adjustment per skin tone group improved V-VI sensitivity by 6.2 points -- more than any architectural change across 13 experiments.

4. **Domain alignment matters more than dataset size.** SCIN (10K images, smartphone self-photos) destroyed the model despite its size. SD-198 (6.5K, clinical photos) was far more useful due to domain similarity (cosine sim 0.748 vs 0.637).

---

## Explainability

- **Grad-CAM heatmaps** show which image regions drive predictions, enabling clinicians to verify the model is attending to the lesion rather than background artifacts.
- **Monte Carlo Dropout** provides uncertainty quantification through multiple stochastic forward passes. High-uncertainty predictions are flagged for expert review.
- **Temperature-calibrated confidence scores** ensure predicted probabilities reflect true accuracy rates (ECE = 0.067).
- **MedGemma clinical explanations** generate natural language assessments describing visual features, differential considerations, and recommended actions.

---

## Limitations

1. **Retrospective validation only.** All results are from research datasets. Prospective clinical validation has not been performed.

2. **High referral rate on diverse benchmarks.** DDI referral rate is ~63%, driven by out-of-distribution conditions the model was not trained on. Real-world clinical distribution would differ.

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
| Fairness | Per-Fitzpatrick decision thresholds (V-VI optimized) |
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
9. Sagers LW, et al. "Augmenting Medical Image Classifiers with Synthetic Data from Latent Diffusion Models." *arXiv preprint arXiv:2308.12453*, 2023.

---

## Disclaimer

This project is for research and educational purposes only. It is not a medical device, has not received regulatory approval, and must not be used to make clinical decisions. All results are from retrospective evaluation on research datasets. Always consult qualified healthcare professionals for medical concerns.

---

*Last updated: April 2026*

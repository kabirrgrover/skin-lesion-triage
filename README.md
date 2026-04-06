# DermTriage

**AI-powered skin lesion triage for primary care — smartphone photos only, no special hardware.**

DermTriage helps primary care physicians decide whether a skin lesion needs dermatology referral. Upload a smartphone photo, get a three-zone triage decision (Refer / Uncertain / Low Risk) with a clinical explanation and visual heatmap — all in under 10 seconds.

**Currently running a small pilot study with clinicians. [Looking for more clinics to enroll.](#pilot-study)**

---

## Why This Matters

Skin cancer affects over 5 million Americans annually. Melanoma — the deadliest form — has a >99% five-year survival rate when caught early. That drops to ~30% once it metastasizes.

Two problems make early detection difficult:

**The access gap.** Primary care physicians achieve roughly 45% diagnostic accuracy for melanoma, compared to 97% for dermatologists. Dermatologist shortages — particularly in rural and underserved areas — mean patients often wait weeks for a specialist appointment.

**The fairness gap.** Most dermatology AI systems are trained predominantly on lighter skin tones (Fitzpatrick I-III) and perform measurably worse on darker skin, compounding existing health disparities.

DermTriage addresses both: a triage tool that works on smartphone photos, with explicit fairness evaluation across the full Fitzpatrick spectrum (I-VI), and performance reporting stratified by skin tone.

---

## Key Numbers

### Shipping Configuration

Dual-encoder ensemble (MedSigLIP + DermLIP), global threshold 0.31.

| Metric | Value |
|--------|-------|
| **Sensitivity** (malignant detection) | **92.4%** |
| **Specificity** | 31.8% |
| **NPV at PCP prevalence (~3%)** | **99.3%** |
| **PPV at PCP prevalence** | 4.0% |
| **Number needed to refer** | 24.9 |
| **AUC** | 0.748 |

> **The headline number:** When DermTriage says **LOW RISK**, it is correct **99.3% of the time** at typical primary care malignancy prevalence.

For context, the current PCP number-needed-to-biopsy for melanoma is 22.6 (Petty et al., JAAD 2020). DermTriage's number-needed-to-refer of 24.9 means the referral burden is roughly equivalent to what PCPs already generate in biopsies — but a derm referral is lower-cost than a biopsy.

### Fairness Across Skin Tones

Evaluated on the [Diverse Dermatology Images (DDI)](https://stanfordaimi.azurewebsites.net/datasets/35866158-8196-48d8-87bf-50dca81df965) dataset — 656 clinical images deliberately balanced across Fitzpatrick I-VI, containing 78 dermatological conditions.

| Fitzpatrick Group | Threshold | Sensitivity | AUC |
|--------------------|-----------|-------------|-------|
| I-II (lighter) | 0.505 | 87.8% | 0.738 |
| III-IV (medium) | 0.63 | 89.2% | 0.810 |
| V-VI (darker) | 0.31 | 85.4% | 0.698 |

All skin tone groups achieve ≥85% sensitivity. The global threshold (0.31) is set to the most conservative group (V-VI) to ensure no population falls below 85% sensitivity.

### Prevalence Context

DDI is an adversarial benchmark: 26.1% malignancy prevalence and 82% out-of-distribution conditions. A real PCP waiting room has ~3% malignancy prevalence. This dramatically changes the clinical picture:

| Metric | DDI (26.1% malignant) | Typical PCP (~3% malignant) |
|--------|----------------------|----------------------------|
| Sensitivity | 92.4% | 92.4% |
| Specificity | 31.8% | 31.8% |
| PPV | 32.3% | 4.0% |
| **NPV** | 92.2% | **99.3%** |
| Referral rate | 74.5% | ~69% |
| FP:TP ratio | 2.1:1 | 23.9:1 |

At PCP prevalence, most referrals will be false positives — but this is also true for human PCPs. The critical number is NPV: when the tool says a lesion is low risk, it is overwhelmingly likely to be correct.

---

## How It Works

### Three-Zone Triage

DermTriage outputs one of three zones, not just a binary yes/no:

| Zone | Color | Meaning | Action |
|------|-------|---------|--------|
| **REFER** | Red | Elevated malignancy probability | Dermatology referral recommended |
| **UNCERTAIN** | Amber | Borderline probability | Clinical judgment — monitor closely or refer |
| **LOW RISK** | Green | Low malignancy probability | Routine monitoring |

The three-zone system gives clinicians a middle ground. Instead of forcing every borderline case into "refer" or "don't refer," UNCERTAIN cases are flagged for closer clinical judgment.

### Architecture

```
Smartphone Photo (448x448)
       |
       +---------------------------+
       |                           |
       v                           v
  MedSigLIP-448              DermLIP-PanDerm
  (Google, frozen)            (Derm-specific, frozen)
  1152-dim features           512-dim features
       |                           |
       v                           v
  7-Class Head                7-Class Head
  (596K params)               (265K params)
       |                           |
       +------- Ensemble (0.6/0.4) ------+
                      |
              Three-zone triage
              (per-Fitz thresholds)
                      |
         +------------+------------+
         |            |            |
      Grad-CAM    MedGemma     Triage
      heatmap    explanation   decision
```

Both vision encoders are frozen — only the lightweight classification heads are trained (under 1M total parameters). MedSigLIP provides high sensitivity (catches melanoma). DermLIP provides complementary specificity (reduces false referrals). The ensemble balances both.

MedGemma (Google's medical language model) generates natural language clinical explanations describing the visual features it observes, why the classification was made, and recommended next steps.

### Seven Lesion Classes

| Class | Description | Risk |
|-------|-------------|------|
| mel | Melanoma | Urgent referral |
| bcc | Basal cell carcinoma | Referral |
| akiec | Actinic keratosis / SCC | Referral |
| bkl | Benign keratosis | Monitoring |
| df | Dermatofibroma | Monitoring |
| nv | Melanocytic nevus | Monitoring |
| vasc | Vascular lesion | Monitoring |

For triage, the three malignant classes (mel + bcc + akiec) are summed into a single malignancy probability, which drives the three-zone decision.

---

## How We Compare

| Tool | Sensitivity | Specificity | NPV (PCP prev.) | Hardware Required | Availability |
|------|-------------|-------------|------------------|-------------------|-------------|
| **DermTriage** | **92.4%** | 31.8% | **99.3%** | **None (smartphone)** | Research prototype |
| DermaSensor | 95.5% | 32.5% | 98.1% | Spectroscopy device ($5K+) | FDA-cleared (Jan 2024) |
| DERM (NHS) | 97% | ~54% | 99.9% | Dermoscope + clip kit | UK clinical use |
| SkinVision | 41-83% | 60-83% | ~93% | None (smartphone) | CE-marked (Europe) |

DermaSensor — the first FDA-cleared AI device for skin cancer detection — launched with 32.5% specificity. DermTriage's 31.8% specificity is nearly identical, but requires zero hardware. DERM achieves higher specificity but requires a dermoscope attachment.

DermTriage is the only tool in this space that:
- Works on **any smartphone camera** with no additional hardware
- Is **physician-facing** (designed for clinical workflow, not consumer self-diagnosis)
- Reports **Fitzpatrick V-VI performance explicitly** (most competitors either don't test or document "use with caution on darker skin")
- Provides **three-zone triage** instead of binary yes/no
- Includes **AI-generated clinical explanations** (via MedGemma)

> **Important:** DermTriage is a research prototype. We do not claim superiority over FDA-cleared or CE-marked devices. These comparisons contextualize our operating range — sensitivity and NPV in the cleared-device range, requiring no hardware, with explicit fairness coverage.

---

## Pilot Study

We are currently running a small blinded pilot study with clinicians to gather real-world feedback on DermTriage's clinical utility.

### How It Works

Clinicians evaluate curated skin lesion images through a three-step blinded feedback flow:

1. **Blinded assessment** — Clinician sees the image only. Enters their diagnosis, confidence level, and recommended action (Refer / Monitor / Reassure). No model output is visible.
2. **Model reveal** — DermTriage's prediction, triage zone, and clinical explanation are shown. The clinician's Step 1 response is locked and cannot be changed.
3. **Agreement rating** — Clinician rates whether they agree with the model (Agree / Partially Agree / Disagree) with optional comments.

This design preserves blinding: we measure what the clinician would have done independently, then compare it to the model's recommendation.

### What We're Measuring

- Clinician agreement rates with DermTriage recommendations
- Cases where the model catches malignancies the clinician initially missed (and vice versa)
- Whether the three-zone system (vs binary) changes clinical behavior
- Whether MedGemma explanations influence confidence or decision-making
- Feedback on workflow integration, trust, and perceived utility

### Looking for Clinics

**We are actively looking for dermatologists and primary care physicians to participate in the pilot.** If you or your clinic would be interested in evaluating DermTriage:

- **Time commitment:** ~15 minutes (review 10-20 curated images through the blinded flow)
- **No patient data involved** — the pilot uses curated research images from published datasets
- **Remote-friendly** — the demo runs in any browser, including mobile

**To participate or learn more, contact: [kabirrgrover@gmail.com](mailto:kabirrgrover@gmail.com)**

We are especially interested in:
- Dermatologists (to benchmark the model against expert assessment)
- Primary care physicians in underserved areas (the target user for this tool)
- Clinicians who serve diverse patient populations, including Fitzpatrick V-VI skin tones

---

## Training

### Datasets

| Dataset | Type | Images | Population | Role |
|---------|------|--------|------------|------|
| HAM10000 | Dermoscopic | 10,015 | Fitzpatrick I-III | Primary training data |
| PAD-UFES-20 | Smartphone clinical | ~2,100 | Fitzpatrick I-V (Brazil) | Domain adaptation |
| DDI | Clinical photography | 656 | Fitzpatrick I-VI (balanced) | Fairness evaluation only |

### Strategy

Both classification heads are trained on HAM10000 + PAD-UFES-20 with 3x oversampling of clinical images. Focal Loss (gamma=2.0) with 2x melanoma class weight. AdamW optimizer, cosine annealing schedule. Lesion-aware splits ensure no patient-level leakage between train and validation.

Mixed-domain training bridged a 30-point performance gap between dermoscopic and clinical images, reducing the melanoma recall gap from 31 points to 4 points. This was the single most impactful intervention in the project.

---

## Experimental Journey

DermTriage is the product of 19 systematic experiments across 20 notebooks. The negative results were as informative as the positive ones.

### Key Experiments

| Experiment | Result | What We Learned |
|------------|--------|-----------------|
| HAM10000-only baseline | 85.7% mel recall (dermoscopic) | Strong in-domain, but 54.5% on clinical photos — a coin flip |
| Mixed training (HAM + PAD) | 90.9% mel recall (clinical) | Domain gap reduced 87%. The biggest win. |
| DermLIP encoder swap | Complementary failure mode | Opposite to MedSigLIP: high specificity, low sensitivity. Led to ensemble. |
| Feature fusion (1664D) | AUC 0.742, does not meet targets | Confirmed encoders are complementary but insufficient alone |
| Binary gate + OOD data | Multiple failures | SCIN contaminated; Fitz17k helped specificity but crashed V-VI |
| LoRA fine-tuning | AUC 0.748 → 0.668 | Adapted toward light-skin patterns. Made everything worse. |
| Per-Fitzpatrick thresholds | **V-VI: 79.2% → 85.4%** | Simple threshold adjustment per skin tone — most effective fairness intervention |
| Prevalence re-weighting | **NPV = 99.3% at PCP prevalence** | DDI metrics dramatically overstate real-world false positive burden |
| Segmentation (SAM) | Killed — AUC regressed | Removing skin background doesn't help; the problem is condition vocabulary |
| Synthetic augmentation | Killed — 95.7% shortcut | CLIP-family encoders trivially separate real from synthetic images |

### Core Findings

1. **The remaining performance gap is a data coverage problem.** Over half of false positives on DDI come from conditions the model has never seen (psoriasis, eczema, warts, etc.). No architecture change fixes this without broader training data.

2. **Per-Fitzpatrick thresholds are the most effective fairness intervention.** A simple threshold adjustment per skin tone group moved V-VI sensitivity by 6.2 points — more than any architectural change across 19 experiments.

3. **CLIP-family vision encoders encode imaging domain as a dominant feature.** MedSigLIP separates clinical photos from dermoscopic images at 99.5% accuracy, and real from synthetic at 95.7%. This domain shortcut is the root cause of multiple failed cross-domain approaches.

4. **DDI metrics overstate the clinical problem.** At realistic PCP prevalence (~3% malignant), the tool's "low risk" call is correct 99.3% of the time. The adversarial benchmark has clinical value for stress-testing, but does not reflect deployment conditions.

### What's Next

- **Clinical pilot study** — gathering clinician feedback on the current model (in progress)
- **PanDerm encoder evaluation** — a self-supervised vision model (Nature Medicine 2025) trained on 2M+ dermatology images across 4 imaging modalities. Unlike CLIP-family encoders, PanDerm was not trained with text-image contrastive learning, which may eliminate the domain shortcut that limited our current approach
- **Prospective validation** — real-world testing beyond retrospective research datasets

---

## Explainability

- **Grad-CAM heatmaps** show which image regions drive the prediction, so clinicians can verify the model is looking at the lesion — not background artifacts or rulers in the frame.
- **MedGemma clinical explanations** generate a natural language assessment describing visible features, consistency with the classification, and recommended clinical actions.
- **Temperature-calibrated confidence scores** (T=0.923, ECE=0.067) ensure that when the model says 80% confidence, it is correct approximately 80% of the time.

---

## Limitations and Honest Gaps

1. **No prospective clinical data.** All results are retrospective, from published research datasets. The pilot study is the first step toward real-world evidence.

2. **High referral rate on adversarial benchmarks.** On DDI (82% out-of-distribution conditions), the referral rate is ~74%. On a realistic PCP distribution, this would be lower — but we don't yet have prospective data to quantify how much lower.

3. **Low specificity.** At 31.8%, most flagged lesions will be false positives. This is comparable to DermaSensor's 32.5% (FDA-cleared), but still means many unnecessary referrals. Improving specificity without sacrificing sensitivity is the primary research challenge.

4. **No pure smartphone-photo benchmark exists.** DDI uses clinical photography (not smartphone photos), and HAM10000 is dermoscopic. The model is designed for smartphone input but has not been validated on a dedicated smartphone benchmark.

5. **Fitzpatrick V-VI sample size is small.** DDI has ~48 malignant samples in the V-VI group. Per-Fitz threshold estimates are statistically fragile and should be re-validated on larger diverse datasets.

6. **Single clinical training dataset.** PAD-UFES-20 is from one institution in Brazil. Generalization to other populations, devices, and conditions is not yet validated.

7. **Not a medical device.** DermTriage is a research prototype. It has not received FDA clearance or any regulatory approval, and must not be used for clinical decision-making.

---

## Technical Details

| Component | Specification |
|-----------|--------------|
| Primary encoder | MedSigLIP-448 (Google, frozen, ~429M params, 1152D) |
| Secondary encoder | DermLIP-PanDerm (derm-specific, frozen, ~87M params, 512D) |
| Ensemble | α=0.6 MedSigLIP + 0.4 DermLIP probability blend |
| Classification heads | LayerNorm → Dropout(0.3) → Linear → GELU → Dropout(0.3) → Linear(7) |
| Total trainable parameters | <1M |
| Triage thresholds | REFER: prob ≥ 0.31 · UNCERTAIN: prob ≥ 0.20 · LOW RISK: prob < 0.20 |
| Calibration | Temperature scaling T=0.923, ECE=0.067 |
| Melanoma safety override | P(mel) ≥ 0.24 overrides argmax to flag melanoma |
| Explainability | MedGemma-4B-IT (clinical text), Grad-CAM (visual heatmap) |
| Training infrastructure | Google Colab (A100 GPU) |
| Framework | PyTorch, HuggingFace Transformers, open_clip |

---

## References

1. Tschandl P, et al. "The HAM10000 dataset, a large collection of multi-source dermatoscopic images of common pigmented skin lesions." *Scientific Data*, 2018.
2. Pacheco AG, et al. "PAD-UFES-20: A skin lesion dataset composed of patient data and clinical images collected from smartphones." *Data in Brief*, 2020.
3. Daneshjou R, et al. "Disparities in dermatology AI performance on a diverse, curated clinical image set." *Science Advances*, 2022.
4. Yan S, et al. "PanDerm: A Foundation Model for Dermatology." *Nature Medicine*, 2025.
5. Petty AJ, et al. "Number Needed to Biopsy to Find a Melanoma: A Systematic Review and Pooled Analysis." *JAAD*, 2020.
6. Rikhye RV, et al. "Evaluating AI systems under real-world distribution shift in dermatology." *eBioMedicine*, 2025.
7. Godau P, et al. "Deployment of AI for medical image classification: the impact of label shift." *Medical Image Analysis*, 2025.
8. Groh M, et al. "Evaluating Deep Neural Networks Trained on Clinical Images in Dermatology with the Fitzpatrick 17k Dataset." *CVPR*, 2021.
9. Sellergren A, et al. "MedGemma Technical Report." *arXiv preprint arXiv:2507.05201*, 2025.
10. Guo C, et al. "On Calibration of Modern Neural Networks." *ICML*, 2017.
11. Selvaraju RR, et al. "Grad-CAM: Visual Explanations from Deep Networks via Gradient-based Localization." *ICCV*, 2017.

---

## Disclaimer

This project is for research and educational purposes only. It is not a medical device, has not received regulatory approval, and must not be used to make clinical decisions. All results are from retrospective evaluation on published research datasets. Always consult qualified healthcare professionals for medical concerns.

---

*Last updated: April 2026*

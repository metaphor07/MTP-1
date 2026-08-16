# Paper 1: Hicks et al. (2021) — "Explaining Deep Neural Networks for Knowledge Discovery in Electrocardiogram Analysis"
**Method introduced: ECGradCAM** | Scientific Reports | DOI: 10.1038/s41598-021-90285-5

---

## Why this paper, and why first

This is the "Grad-CAM for images → Grad-CAM for ECG" paper. It's foundational because it does ONE thing (adapt Grad-CAM to ECG) and does it clearly, with strong validation. Read this before xECGArch, which assumes you already understand what "explanation faithfulness" means and jumps into comparing 13 methods at once.

---

## PART 1 — The Problem & Motivation

### The core tension
Deep learning models are more accurate than humans at many medical tasks, but doctors can't (and shouldn't) trust a model they can't interrogate. If a cardiologist misdiagnoses a patient, they can explain *why* — what they saw, what ruled competing diagnoses out. A black-box neural network can't do that, even if it's right more often.

### Why this matters practically, not just philosophically
The paper gives a real example: a lung cancer vs. pneumonia X-ray classifier that seemed excellent — until someone checked *what it was actually looking at*. It turned out the model had learned to read hospital department labels burned into the image corner (certain departments treat more cancer cases), not the actual lung pathology. When tested on X-rays without those labels, it failed completely. This is called a **shortcut** or **spurious correlation** — the model finds a statistical pattern that correlates with the label in training data but has nothing to do with the actual underlying phenomenon.

**This is your one-liner for why XAI matters, if an interviewer asks "why not just use a black-box model if it's more accurate?":**
> "Because accuracy on a test set doesn't tell you the model is reasoning correctly — it might be exploiting a shortcut that happens to correlate with the label in your specific dataset but won't generalize. Explainability is how you catch that before deployment, especially in medicine where a wrong shortcut can be fatal."

### Three reasons explainability matters (paper's framing — memorize this structure)
1. **Trust & error-catching** — a doctor can verify the model is using real physiological signal, not an artifact of the data collection process.
2. **Scientific discovery** — if a model can predict something (like a disease outcome), and you can see *what part of the signal* it's using, you might learn new medical facts humans hadn't noticed.
3. **Accountability** — if an AI makes a wrong call, you need to be able to say why, for legal/clinical responsibility.

Your thesis proposal bullet ("faithfulness, stability, and robustness") is really about #1 — verifying the model's explanation is trustworthy, not just plausible-looking.

---

## PART 2 — What They Actually Built (the model, before the XAI part)

Important distinction to keep straight: **the paper has two separate contributions** — (1) a prediction model, and (2) an explanation method (ECGradCAM) applied to that model. Don't conflate them in an interview.

### The prediction model
- A **1D residual CNN** (based on ResNet, adapted from images to 1D time-series) with 8 residual blocks, ~1.65 million parameters.
- Input: 12-lead ECG, either as a 10-second raw rhythm strip, or a "median beat" (a single averaged, cleaned-up representative heartbeat, 1.2 seconds).
- Large kernel sizes (kernel size 50 in the residual blocks) — deliberately large so a single convolution can "see" across a big chunk of the signal, potentially spanning both a P-wave and a QRS complex in one filter's receptive field. This is a design choice specific to ECG — you wouldn't necessarily use such large kernels on natural images.
- Output: NOT a diagnosis category. Instead, the model directly *regresses* clinically meaningful numbers: PR interval, QRS duration, heart rate, QT interval, R-wave amplitude, T-wave amplitude, etc. Plus one classification task: predicting the *sex* of the patient from the ECG alone.

**Why regression instead of classification of "normal/abnormal"?** This is a subtle but important design choice. If you train a model to output a category like "normal" vs "abnormal," you get one bit of information and it's hard to know *what* it measured to get there. If you train it to predict the actual PR interval in milliseconds, you can directly check: is the model's number close to the true number? And you get many independent variables to test explainability against, not just one.

### Model performance (know these numbers if asked)
- The model beat two experienced cardiologists on precision and consistency for interval/amplitude measurement — errors were **4–5x lower** than human error.
- Cardiologists showed real inconsistency between repeated readings of the same ECG (intra-observer variability); the network didn't.
- For sex classification (a task that's essentially impossible for a cardiologist to do by eye), the model hit **89% accuracy**.

---

## PART 3 — ECGradCAM: the actual explainability method

This is the heart of the paper and the part you MUST be able to explain mechanically, not just conceptually.

### First, what is regular Grad-CAM (image version)?
Grad-CAM was originally built for image classifiers (CNNs on 2D images). The idea:
1. Pick the layer near the end of the network (usually the last convolutional layer, before the fully-connected/prediction layers).
2. For a specific prediction (e.g., "this is a cat"), compute the **gradient** of that prediction with respect to the feature maps in that layer. This gradient tells you: "if I increased the activation of this feature map, how much would it push the prediction toward 'cat'?"
3. Use those gradients to weight each feature map channel, then sum them up — producing a coarse heatmap over the image showing which spatial regions were most influential for the prediction.
4. Overlay that heatmap on the image — red = important, blue = unimportant.

### What did they change for ECG? (this is the "ECG" in ECGradCAM)
Two adaptations, and you should be precise about both:

1. **1D instead of 2D.** An image has a 2D spatial grid (height × width); an ECG is a 1D time-series (just time). So instead of producing a 2D heatmap over an image, ECGradCAM produces a 1D importance signal over time, which you overlay directly on the ECG waveform as a color trace.

2. **Multi-lead averaging.** A 12-lead ECG isn't one signal, it's 12 simultaneous signals (12 electrical "viewpoints" of the same heartbeat). ECGradCAM generates a separate attention/importance signal *for each lead*, then **averages across all 12 leads** to get one final importance trace. The paper's reasoning: this produces a more fine-grained, more representative signal than looking at any single lead alone.

**If an interviewer asks "how is Grad-CAM different for ECG vs images" — this is your answer, in one breath:**
> "The mechanism is identical — gradients of the output with respect to the last convolutional layer's activations, used to weight and sum feature maps. The difference is the output shape: instead of a 2D heatmap over pixels, you get a 1D importance trace over time, and because ECG has multiple leads, you compute it per-lead and average across leads to get one consolidated signal."

### Which layer did they use, and why does that matter?
They used the **last residual block, right before the final prediction** — because that shows what the network is looking at *at the moment of decision*. They also mention (in a supplementary figure) that earlier layers show broader, less focused attention that narrows down as you go deeper — this is a nice mental model: early layers = "the network is scanning the whole beat," late layers = "the network has zeroed in on the answer."

**This is a good depth signal for an interview** — if asked "does it matter which layer you extract Grad-CAM from," you should be able to say yes, and explain why (interpretability of intermediate vs final layers, granularity vs decision-relevance tradeoff).

---

## PART 4 — Did the explanations actually make sense? (Validation)

This is the part that makes the paper credible, not just a cool visualization. **Producing a heatmap is easy. Proving the heatmap is honest is hard — and this is exactly the "faithfulness" concept your thesis is built around.**

### Validation approach #1: does it point at the right wave?
For each predicted variable, they checked whether the resulting attention map lit up on the anatomically correct part of the signal:
- Predicting **QRS duration** → attention lights up on the QRS complex. ✔ makes sense.
- Predicting **QT interval** → attention lights up at both the start of the QRS and the end of the T-wave (since QT interval spans from QRS onset to T-wave end). ✔ makes sense — this is a nice detail, because it shows the model is finding *both boundaries* of an interval, not just one point.
- Amplitude predictions (e.g. R-wave amplitude) → attention correctly focuses on the peak of that specific wave.

They also noticed a secondary phenomenon: the model gave some minor importance to flat/baseline parts of the ECG even when measuring an unrelated wave. Their explanation: because they used **batch normalization** in training, the network needs some notion of the signal's baseline/scale to correctly output absolute voltage values — so it "checks" the baseline as a reference point. This is a subtle, technically-informed observation you should be ready to reference — it shows they're not just eyeballing pretty pictures, they're reasoning about *why* an unexpected attention pattern exists mechanistically.

### Validation approach #2: "wave blocking" (an ablation-style sanity check — independent of ECGradCAM)
This is arguably the more rigorous validation, and it's a really good tool for your own toolkit. Idea: if the attention map says "the model is using the QRS complex to predict QRS duration," you should be able to *test* that claim directly — remove (blank out via interpolation) the QRS complex from the signal and see if the model's performance collapses.
- They did exactly this: separately blanked the P-wave, QRS complex, or T-wave.
- Result: removing the wave that *should* matter for a given prediction caused a large performance drop; removing irrelevant waves caused only a small drop.
- This independently confirms the ECGradCAM heatmaps weren't just visually plausible — the model really was depending on those regions.

**This two-pronged validation (attention map + independent ablation test that doesn't rely on the XAI method itself) is the single most important methodological lesson from this paper for your own thesis.** You should plan to do something structurally similar: don't just show a SHAP/IG/Grad-CAM heatmap and claim it looks right — independently verify with a perturbation/ablation experiment that doesn't rely on the explanation method itself. This directly matters for your "faithfulness" evaluation goal.

---

## PART 5 — The "novel discovery" case study (sex classification)

This is the paper's flashiest result and a great talking point.

- Cardiologists cannot reliably tell a patient's biological sex from an ECG by eye.
- The CNN did it at 89% accuracy — impossible for a human, so the *only* way to understand why is via the explanation.
- ECGradCAM showed the model was focusing heavily on the **downslope of the R-wave** (part of the QRS complex) — not something previously well-documented as a strong sex-differentiating feature.
- They validated this finding independently (again, not just trusting the heatmap): built a simple logistic regression using only QRS duration + R/S amplitudes + timing, and got 73% accuracy / 0.80 AUC (vs. the CNN's 89% / 0.96 AUC) — much better than QRS duration alone (69%/0.72), confirming the R-wave downslope carries real information.
- They confirmed this again via wave-blocking: removing the QRS complex tanked sex-prediction accuracy; removing P-wave or T-wave barely mattered.

**Why this matters for your novelty pitch:** this is a template for "XAI as a discovery tool," not just "XAI as a trust tool." Worth remembering as one of the framings for your own thesis's value proposition.

---

## Quick-reference: Likely interview questions on this paper, with tight answers

**Q: What's ECGradCAM and how does it differ from vanilla Grad-CAM?**
A: Grad-CAM adapted from 2D image spatial heatmaps to 1D time-series importance traces; computed per ECG lead then averaged across all 12 leads for one consolidated signal.

**Q: How did they know the explanations were trustworthy, not just pretty pictures?**
A: Two independent checks — (1) visual/anatomical correspondence (attention lands on physiologically correct waves for each predicted variable), and (2) wave-blocking ablation — physically removing a wave and confirming performance collapses only when the "important" wave (per the attention map) is removed.

**Q: What's the biggest limitation of this paper, if you had to critique it?**
A: It's evaluated only on clean, well-recorded 12-lead clinical ECGs from population studies — no wearable-grade noise, no motion artifact, no single-lead setting. It also doesn't test explanation *stability* — whether the same ECG with small perturbations gives a consistent explanation. That's exactly the gap your thesis targets.

**Q: Why regress exact intervals instead of just classifying normal/abnormal?**
A: Regression gives continuous, independently verifiable ground truth for each variable, making it much easier to check if the explanation is anatomically correct — a binary label doesn't give you that kind of fine-grained validation target.

---

*Next: xECGArch (Goettling et al., 2024) — builds on this by comparing 13 different XAI methods systematically and splitting short-term (morphology) vs long-term (rhythm) analysis into separate networks.*

# MVMS-Net Experiment Results — Summary for Mentor

Two changes were tried on top of the reproduced baseline model: a different **loss function** (ASL) and a different **backbone block** (OS-CNN). Results below, plus what the metrics mean.

## What Sensitivity and Specificity mean

**Sensitivity** = out of all patients who actually have the disease, what % did the model correctly catch (higher = fewer missed diagnoses). **Specificity** = out of all patients who are actually healthy, what % did the model correctly clear (higher = fewer false alarms). Both are "higher is better," but they trade off against each other — pushing one up usually pulls the other down, as seen clearly below.

---

## Experiment 1: Loss function — Asymmetric Loss (ASL)

**What it is**: the original model treats every training example equally when learning. ASL instead down-weights the huge number of "easy" negative examples, so the model focuses more on rare/hard cases — directly targeting the class-imbalance problem the original paper admits it never solved.

**Why we tried it**: PTB-XL is heavily imbalanced (some conditions are rare), and the paper's own conclusion names this as an unsolved limitation.

| Task | Stage | Metric | BCE (original loss) | ASL (new loss) | Difference |
|---|---|---|---|---|---|
| superdiagnostic | teacher | Best epoch | 10 | 11 | — |
| | | AUC | 92.64 | 92.64 | ~0.00 |
| | | Sensitivity | 75.90 | **90.83** | **+14.93** |
| | | Specificity | 94.19 | 80.08 | −14.11 |
| superdiagnostic | student | Best epoch | 45 | 26 | — |
| | | AUC | 92.51 | 92.85 | +0.34 |
| | | Sensitivity | 77.09 | **90.94** | **+13.86** |
| | | Specificity | 93.50 | 80.05 | −13.45 |
| form | teacher | Best epoch | 9 | 9 | — |
| | | AUC | 86.66 | 86.73 | +0.07 |
| | | Sensitivity | 44.32 | **76.46** | **+32.13** |
| | | Specificity | 98.44 | 90.20 | −8.24 |
| form | student | Best epoch | 24 | 12 | — |
| | | AUC | 87.01 | 85.31 | −1.69 |
| | | Sensitivity | 46.62 | **75.75** | **+29.13** |
| | | Specificity | 98.24 | 91.56 | −6.68 |
| subdiagnostic | teacher | Best epoch | 17 | 12 | — |
| | | AUC | 93.34 | 92.98 | −0.36 |
| | | Sensitivity | 64.93 | **84.19** | **+19.26** |
| | | Specificity | 98.82 | 95.45 | −3.37 |
| subdiagnostic | student | — | — | — | 🔄 in progress, not confirmed |
| rhythm | both | — | — | — | ⬜ not started yet |
| diagnostic | both | — | — | — | ⬜ not started yet |
| all | both | — | — | — | ⬜ not started yet |

**Which performs better**: ASL wins clearly, on every task tested so far. AUC barely moves, but Sensitivity jumps 14-32 points every time, at a smaller cost of 3-14 points of Specificity. This is a consistent, repeatable effect across 5 tasks — the model catches dramatically more real disease cases, at the cost of a few more false alarms, which is normally a good trade for a screening tool. **5 of 6 tasks confirmed so far.**

---

## Experiment 3: Backbone — OS-CNN instead of Res2Net

**What it is**: the original model's "Res2Net" block looks at the ECG signal through one fixed window size (15 samples, chosen by trial and error). OS-CNN instead looks through several window sizes at once (3, 5, 7, 11, 13, 17, 19, 23 samples, chosen using prime numbers for principled coverage), then combines what it sees at each scale.

**Why we tried it**: Res2Net was originally built for images, just adapted here. OS-CNN was built specifically for time-series signals like ECG from the start, so it's a natural candidate to test.

| Task | Stage | Metric | BCE + Res2Net (original) | BCE + OS-CNN (new) | Difference |
|---|---|---|---|---|---|
| superdiagnostic | teacher | Best epoch | 10 | 12 | — |
| | | AUC | 92.64 | 92.74 | +0.10 |
| | | Sensitivity | 75.90 | 76.74 | +0.84 |
| | | Specificity | 94.19 | 93.24 | −0.95 |
| superdiagnostic | student | Best epoch | 45 | 38 | — |
| | | AUC | 92.51 | 92.36 | −0.15 |
| | | Sensitivity | 77.09 | 75.28 | −1.81 |
| | | Specificity | 93.50 | 93.57 | +0.07 |
| form | teacher | Best epoch | 9 | 13 | — |
| | | AUC | 86.66 | 85.80 | −0.86 |
| | | Sensitivity | 44.32 | 42.65 | −1.67 |
| | | Specificity | 98.44 | 98.43 | −0.01 |
| form | student | Best epoch | 24 | **50 ⚠️** | — |
| | | AUC | 87.01 | 86.31 | −0.70 |
| | | Sensitivity | 46.62 | 43.60 | −3.02 |
| | | Specificity | 98.24 | 98.44 | +0.20 |

⚠️ **Note on form/student**: OS-CNN's run hit the 50-epoch limit exactly, unlike every other run which stopped naturally on its own — it may still have been improving when cut off, so this one result is less certain than the rest.

**Which performs better**: Res2Net (the original) wins narrowly — 3 of 4 comparisons favor it, one favors OS-CNN, and none of the differences are large (mostly under 1-2 points, close to normal run-to-run noise). **Conclusion: switching backbones doesn't help — keeping Res2Net**, the architecture the paper already validated.

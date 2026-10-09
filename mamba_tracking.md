# Mamba Backbone Tracking — `15_mamba_variants.ipynb`

Live tracking of the Mamba backbone experiment (Experiment 3c — a
standard, established architecture unrelated to MVMS-Net's own Res2Net
design, direct 12-lead training, no distillation; see
`src/mamba_backbone.py`). Updated as each variant's results come in.

**Status so far: 3 of 5 variants complete (`base`, `bidirectional`,
`multiscale`), all 6 tasks each.** `coordatt`, `convstem` not yet reported.

**Backend**: official `mamba_ssm` CUDA kernel confirmed working on Kaggle
(0.02s/iteration, ~2 min for a 20-epoch run) — see prior chat log. Not the
pure-PyTorch fallback.

## Test AUC, by task and variant

| Task | base | bidirectional | multiscale | Winner |
|---|---|---|---|---|
| all | 0.9032 | 0.9024 | **0.9108** | multiscale (+0.0076 over base) |
| diagnostic | **0.9209** | 0.9154 | 0.9180 | base (+0.0029 over multiscale) |
| form | 0.8259 | 0.8149 | **0.8464** | multiscale (+0.0205 over base) |
| rhythm | 0.9409 | 0.9564 | **0.9634** | multiscale (+0.0070 over bidirectional) |
| subdiagnostic | 0.9122 | **0.9167** | 0.9120 | bidirectional (+0.0045 over base) |
| superdiagnostic | 0.9201 | 0.9198 | **0.9207** | multiscale (+0.0006, effectively tied) |

**So far**: `multiscale` is the strongest of the three completed variants
— wins (or ties) on 4 of 6 tasks, including a clear win on `form`
(+0.0205) where every Mamba variant has otherwise struggled most. `base`
still wins on `diagnostic`, `bidirectional` still wins on `subdiagnostic`.
No variant dominates everywhere — consistent with the OS-CNN result
elsewhere in this project (architecture swaps on top of a working design
don't always help, and when they do it's task-dependent, not universal).

## Context: how does Mamba compare to the actual MVMS-Net reproduction?

Not a clean apples-to-apples comparison (the MVMS-Net number below is the
**distilled student**, Mamba here is **direct-trained, no distillation**
— same caveat already noted for the InceptionTime experiment), but useful
context:

| Task | MVMS-Net student (res2net, BCE) | base | bidirectional | multiscale |
|---|---|---|---|---|
| superdiagnostic | 0.9251 | 0.9201 | 0.9198 | 0.9207 |
| form | 0.8701 | 0.8259 | 0.8149 | 0.8464 |
| subdiagnostic | 0.9334 | 0.9122 | 0.9167 | 0.9120 |
| rhythm | 0.9422 | 0.9409 | 0.9564 | **0.9634** |
| diagnostic | 0.9081 | **0.9209** | 0.9154 | 0.9180 |
| all | 0.9185 | 0.9032 | 0.9024 | 0.9108 |

**Worth flagging**: on two tasks, a directly-trained Mamba variant
actually **beats** the distilled res2net student — `multiscale` on
`rhythm` (0.9634 vs 0.9422, the largest margin of any Mamba variant so
far) and `base` on `diagnostic` (0.9209 vs 0.9081; `multiscale` also beats
it here, 0.9180 vs 0.9081). On the other four tasks Mamba still trails
the res2net student, though `multiscale` has narrowed the gap
considerably on `form` (-0.0237, versus -0.0442/-0.0552 for
base/bidirectional). This is an early, partial result (3 of 5 variants)
— not a final conclusion.

## Open items

- `coordatt`, `convstem` variants: not yet run/reported.
- No SEN/SPEC numbers captured yet in this file — only test AUC was in
  the pasted summary table. Pull these from the actual
  `ptbxl_mamba_*_summary.csv` / per-task `results.csv` files once
  downloaded, same as done for the reproduction table in `paper/main.tex`.
- Once all 5 variants are in, decide (per earlier plan) which one "wins"
  and gets the explainability + single-lead treatment in
  `16_mamba_explain_and_single_lead.ipynb`.

---

*Last updated: 2026-10-10 (date per session clock), after `base`,
`bidirectional`, and `multiscale` variants finished, all 6 tasks each.
Update the tables above when `coordatt`/`convstem` results come in.*

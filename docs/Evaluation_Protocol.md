# Koza Clip Engine — Evaluation Protocol

**Companion to [`../Draft_Architecture.md`](../Draft_Architecture.md).** This document makes the
research claims (H1–H4) *falsifiable*: it defines the datasets, ground truth, metrics, baselines,
and statistical procedure used to accept or reject each hypothesis. Source figures for system
benchmarks are drawn from the internal report **[BvB-2026]**.

> **Principle.** No subsystem is considered "done" until it beats a named baseline on a named
> metric over the benchmark, with the result reproducible from a pinned config.

---

## 1. Hypotheses → Tests (traceability matrix)

| Hypothesis | Subsystem | Metric(s) | Baseline | Accept if |
|-----------|-----------|-----------|----------|-----------|
| **H1** Hybrid selector > single-mode | S5 Selection | Highlight relevance (IoU, P@k, recall vs. references) | Transcript-only; multimodal-only | Hybrid wins relevance at ≤ multimodal cost |
| **H2** Active-speaker reframe > center-crop | S7 Reframe | Subject-retention %; off-center error | Static center-crop | Higher retention, lower error, p<0.05 |
| **H3** Explainable score correlates w/ engagement | S10 Score | Spearman ρ vs. most-replayed; ranking nDCG | Random; duration-only | ρ significantly > 0 |
| **H4** Transcript-first competitive on speed | whole pipeline | TTFC, clips/hour | [BvB-2026] vendor figures | TTFC within fast-tier band (≈5–12 min/90-min) |

---

## 2. Datasets & Ground Truth

### 2.1 Long-form benchmark (the core gap — academic sets skew to ~2.5-min clips)
Curate a held-out set spanning the genres SaaS tools target **[BvB-2026 §1]**:
- **Podcasts / interviews** (multi-speaker, conversational) — tests diarization + speaker-aware reframe.
- **Talks / webinars** (single dominant speaker, slides) — tests information-density selection.
- **Streams / gaming / sports** (visual, non-verbal peaks) — tests where transcript-only fails.

Target: ≥ 20 sources, 20–120 min each, CC-licensed / permissioned, with stable public URLs for
reproducibility.

### 2.2 Ground truth
1. **Human highlight references** — ≥ 2 annotators mark clip-worthy spans per source using the
   **clip-worthiness rubric** (`[OPEN-F]` in Draft_Architecture §9 — must be frozen before
   labeling). Report **inter-annotator agreement** (Cohen's κ / span IoU).
2. **Weak labels: YouTube "most-replayed"** — heatmap peaks as a noisy engagement proxy (per
   CHI-2025 "most-replayed" line of work). Used for H3 correlation, *not* as primary selection GT.
3. **(Optional) SaaS reference outputs** — where ToS/licensing permit, store clips from 1–2
   vendors for pairwise human preference comparison only (never as training data).

---

## 3. Metrics (definitions)

### 3.1 Selection (S5) — RQ1/H1
- **Highlight IoU / overlap** between predicted spans and human references.
- **Precision@k / Recall@k** at the operating point *k* (default k = number of reference clips).
- **Ranking quality:** nDCG of the predicted ranking against graded references.

### 3.2 Reframing (S7) — RQ2/H2
- **Subject-retention rate:** fraction of frames where the active speaker's face bbox center lies
  inside the 9:16 crop. (Active speaker from S3 diarization + face detection.)
- **Off-center error:** mean normalized distance of subject center from crop center.
- **Path smoothness:** jerk/variance of crop center trajectory (penalizes jitter).

### 3.3 Captions (S8)
- **WER** of caption text vs. reference transcript.
- **Alignment error:** mean |predicted − reference| word start time (ms).

### 3.4 Engagement score (S10) — RQ3/H3
- **Spearman ρ** between Koza score and most-replayed intensity over candidate spans.
- **nDCG** of score-induced ranking vs. weak labels.
- **Explainability check:** every score ships with per-feature contributions (no black box).

### 3.5 System / cost — RQ4/H4
- **TTFC** (time to first clip) and **clips/hour**, compared to **[BvB-2026]** bands
  (TTFC 5–25 min; throughput ~17–31 clips/hr; Submagic transcript-led ~8–12 min on 90-min input).
- **Cost/run:** GPU-seconds + any API tokens, logged by the cost accountant.

### 3.6 End-to-end human preference
- **Pairwise A/B** (Koza vs. baseline/SaaS) judged on hook strength, watchability, caption
  quality; report win-rate + binomial CI.

---

## 4. Baselines & Ablations

**Baselines:** (a) transcript-only selector (LLM over transcript, no signals/visual);
(b) multimodal-only selector; (c) static center-crop reframer; (d) duration-only / random score;
(e) SaaS reference (preference only, licensing permitting).

**Ablations:** signal-priors on/off (H1); LLM local (Qwen/LLaMA) vs. remote (Claude/GPT);
reframe smoothed-path vs. speaker-switch; multimodal refinement on/off (latency vs. quality).

---

## 5. Statistical Procedure
- Fixed train/dev/**held-out test** split at the *source* level (no leakage across clips of one
  source). Test set touched only for final numbers.
- Report mean ± 95% CI over sources; bootstrap (≥1000 resamples) for metric CIs.
- Significance: paired test across sources (Wilcoxon signed-rank) for H1–H3; α = 0.05.
- All runs pinned: config hash, seeds, component versions recorded in the `Project.run` record.

---

## 6. Reproducibility Checklist (per reported number)
- [ ] Config file committed; seed fixed.
- [ ] Input source list with stable URLs/hashes.
- [ ] Component versions pinned (lockfile).
- [ ] Stage artifacts content-hashed and cached.
- [ ] One command regenerates the figure/table.
- [ ] Result logged with CI + n (sources).

---

## 7. Leaderboard (to publish with release)
A public table reporting, per method/ablation: highlight IoU, P@k, subject-retention, caption WER,
score ρ, TTFC, clips/hr, cost/run — enabling the community to compare open methods on a common,
long-form benchmark (the artifact the field currently lacks, per **[BvB-2026 §2]**).

---

## References
See [`../Draft_Architecture.md`](../Draft_Architecture.md) §References. Primary system-benchmark
figures: **[BvB-2026]** Parts 1–2.

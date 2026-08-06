# Koza Clip Engine — Draft Architecture & Research Design

**An open-source, reproducible pipeline for automatic long-form → short-form video generation**

| | |
|---|---|
| **Document type** | Draft architecture / research design (living document) |
| **Status** | `DRAFT v0.2` — for iteration. Design decisions are explicitly flagged `[OPEN]`. |
| **Principal Investigator** | Shawn Krishnan (Kokka Co.) |
| **Scope of v1** | Core Python pipeline, no UI. Open-source, research-paper standards. |
| **Last updated** | 2026-06-27 |
| **Companion docs** | [`docs/Evaluation_Protocol.md`](docs/Evaluation_Protocol.md) · [`docs/ADI_Format_Spec.md`](docs/ADI_Format_Spec.md) · Source: `build_vs_buy_ai_video_clipping_report_20260619.pdf` (cited herein as **[BvB-2026]**) |

> **How to read this document.** It is written as a research design, not a spec sheet. Each
> subsystem states its **objective**, the **design rationale** (why this choice over
> alternatives), and the **open decisions** still to be resolved. Nothing is final until the
> `[OPEN]` items in §9 are closed. This mirrors how a PI scopes a project before committing
> engineering effort: motivate → hypothesize → specify → evaluate → de-risk.

---

## Abstract

Automated video clipping — turning a long-form source (live stream, podcast, webinar) into
multiple captioned, vertical, social-ready short clips — has matured into a crowded commercial
SaaS market led by OpusClip (10M+ users), with fast-following entrants (Reap, Klap, Submagic,
Vizard, Riverside) differentiating on speed, caption quality, and developer integration
**[BvB-2026, Part 1]**. In parallel, a capable open-source component ecosystem now exists
(Whisper/faster-whisper, pyannote, local LLMs, MediaPipe/YOLO, FFmpeg) that makes a private,
fully-owned pipeline viable **[BvB-2026, Part 2]**. **This project takes the "Build" path** and
asks a sharper question than any vendor answers publicly: *can an open, reproducible pipeline
match commercial clip quality while being fully transparent, benchmarkable, and self-hostable?*
We propose **Koza**, a modular Python pipeline plus a public benchmark and evaluation protocol,
positioned as the open, research-grade reference implementation the field currently lacks.

---

## 1. Background & Motivation

The **[BvB-2026]** report establishes three relevant facts that motivate this work:

1. **The capability is real and commercially proven.** Vendors reliably identify highlights,
   reframe to 9:16, caption, and even predict "virality" (OpusClip's 0–100 *AI Virality Score*;
   ClipAnything multimodal highlight search; ReframeAnything object-tracking reframe)
   **[BvB-2026 §1.1]**.
2. **The open-source building blocks are mature** but un-integrated: there is no canonical,
   reproducible, openly-benchmarked reference pipeline — only fragmented components and
   closed-source products **[BvB-2026 §2]**.
3. **The differentiators that favor Build are data ownership, customization, and cost-at-scale.**
   No major SaaS offers on-prem deployment; raw video + transcripts leave the customer's
   environment; per-minute pricing (e.g. ~$0.10–0.15/min) punishes high volume
   **[BvB-2026 §1, §4]**.

**The gap this project fills:** the research and OSS communities lack a *transparent, measurable,
reproducible* clipping pipeline against which methods (highlight selection, reframing, virality
estimation) can be compared. Koza aims to be that reference — code + benchmark + paper.

---

## 2. Problem Statement & Research Questions

**Problem.** Given a long-form audiovisual source `V` (minutes to hours, 1–N speakers), produce
a ranked set of short clips `{c_1 … c_k}` (≈15–90 s, vertical 9:16, captioned) that maximize
audience engagement while preserving narrative self-containment — *reproducibly and without the
source leaving the operator's environment.*

We organize the work around four research questions:

- **RQ1 (Selection).** What combination of transcript-level (LLM) and signal-level
  (audio/visual) features best predicts human-judged clip-worthiness on *long* sources?
  (Most academic highlight models are trained on ~2.5-min clips and generalize poorly to
  hour-long inputs.)
- **RQ2 (Reframing).** Can an open saliency + active-speaker reframer (MediaPipe/YOLO + smoothed
  camera path) match the subject-retention quality of commercial ReframeAnything-style systems?
- **RQ3 (Virality estimation).** Is an open, explainable engagement score (vs. OpusClip's
  closed 0–100) attainable, and does it correlate with real-world performance signals
  (e.g. YouTube "most-replayed")?
- **RQ4 (Architecture).** Does a *transcript-first* pipeline with optional multimodal refinement
  deliver the speed of Reap/Submagic (TTFC ≈ 5–12 min on 90-min input) without the quality loss
  of transcript-only methods? **[BvB-2026 §1, Performance]**

---

## 3. Hypotheses

- **H1.** A *hybrid* selector — cheap signal priors (audio energy, laughter, sentiment peaks,
  scene cuts) to shortlist candidates, then an LLM to rank and justify — will outperform either
  transcript-only or multimodal-only baselines on long sources, at a fraction of full
  frame-by-frame cost. (Directly tests RQ1; motivated by the speed/quality trade-off in
  **[BvB-2026 §1]** where multimodal = slow, transcript-only = misses non-verbal moments.)
- **H2.** Active-speaker-aware reframing materially improves subject-retention over static
  center-crop on multi-speaker content (RQ2).
- **H3.** An explainable engagement score built from interpretable features will correlate
  positively with "most-replayed" weak labels, providing an open alternative to closed virality
  scores (RQ3).
- **H4.** A transcript-first, adapter-based architecture achieves competitive TTFC while keeping
  every stage swappable and reproducible (RQ4).

---

## 4. Specific Aims

| Aim | Description | Primary RQ | Deliverable |
|-----|-------------|-----------|-------------|
| **A1** | Build the modular end-to-end pipeline (ingest → render) with a typed `Project` data model and per-stage adapters | RQ4 | `koza` package, thin E2E slice |
| **A2** | Design + implement the hybrid highlight selector | RQ1 | `selection` module + ablations |
| **A3** | Implement open active-speaker reframing | RQ2 | `reframe` module |
| **A4** | Define + implement the explainable engagement score | RQ3 | `scoring` module |
| **A5** | Assemble the Koza benchmark + evaluation harness | all | public benchmark + metrics |
| **A6** | Reproducibility & open-science release (paper, pinned env, configs) | all | paper + repo |
| **A7** | `.adi` interchange + Neo4j (bidirectional) + DaVinci MCP editable-timeline integration | RQ1, RQ4 | `.adi` writer, graph loader, MCP compiler |

---

## 5. Related Work & Prior Art

*Grounded in [BvB-2026]; to be expanded into a formal related-work section for the paper.*

### 5.1 Commercial SaaS (what we benchmark against, not with) **[BvB-2026 §1]**
- **OpusClip** — market leader (10M+ users); multimodal *ClipAnything*, *ReframeAnything*,
  *AI Virality Score* (0–100); SOC 2 Type II; multimodal ⇒ slower (~25 min TTFC on 90-min input).
- **Reap** — agent-native, *native MCP server + CLI*, full REST API on entry tiers; transcript-first
  ⇒ ~5× faster TTFC than OpusClip in the April-2026 benchmark. Most architecturally relevant to us.
- **Submagic** — caption/style specialist, ~98.9% transcription accuracy, word-by-word animated
  captions; per-minute pricing.
- **Klap** — speed/volume + multilingual dubbing (29 langs); batch/REST API, webhooks.
- **Vizard / Riverside Magic Clips / Descript** — collaborative text-based editing, podcast-integrated
  clipping, top-tier transcription respectively.

### 5.2 Open-source pipeline components (what we build with) **[BvB-2026 §2.2]**
- **Ingestion:** `yt-dlp`; job queue (Redis/Bull or RabbitMQ) for scale.
- **ASR:** Whisper / **faster-whisper** (CTranslate2, 4–8× faster) / **WhisperX** (forced
  alignment → word-level timestamps). CPU fallback: `whisper.cpp`.
- **Diarization:** **pyannote.audio** (~85–90% attribution); NeMo, Resemblyzer alternatives.
  *Note: pyannote pretrained models carry a research license — licensing flagged in §8.*
- **Highlight detection:** local LLMs (LLaMA/Mistral/Qwen via Ollama/LangChain) or remote
  (Claude/GPT) over the transcript; structured-JSON scoring on hook strength, narrative
  completeness, information density, emotional content.
- **Reframing/CV:** MediaPipe face detection, YOLOv8, SAM (saliency, compute-heavy); smoothed
  FFmpeg `crop` path following the active speaker.
- **Rendering:** **FFmpeg** (de facto; single-pass crop+caption ≈ 35% faster than multi-pass),
  MoviePy wrapper, NVENC GPU encode; **OpenTimelineIO** for editorial interchange.

### 5.3 Academic highlight detection
Moment-DETR / MH-DETR family; unsupervised audio-visual recurrence (WACV-2025); "most-replayed"
length-controlled highlighting (CHI-2025); human-inspired multimodal editing frameworks. Key
limitation to address per RQ1: benchmarks skew to short videos (~2.5 min) and generalize poorly
to long sources.

### 5.4 AI ↔ NLE integration (future option, not v1) **[BvB-2026 §3]**
Model Context Protocol (MCP) as "USB-C for AI"; `premiere-pro-mcp` (269 tools via ExtendScript),
`davinci-resolve-mcp` (Python API). Relevant as a *downstream export target* (OTIO + MCP) so Koza
clips can be handed to human editors rather than only rendered flat. Tracked as a stretch aim.

---

## 6. Proposed System Architecture

### 6.1 Design principles
1. **Modular, adapter-based stages.** Each stage is a function over a shared `Project` object;
   cloud vs. self-hosted is a per-stage adapter, not a global commitment. This makes the
   speed/quality/cost trade-offs in **[BvB-2026]** *empirically tunable* rather than baked in.
2. **Transcript-first, multimodal-optional.** Default to the fast transcript-led path (Reap/
   Submagic philosophy); enable multimodal refinement only where it earns its latency (H1, H4).
3. **Reproducible by construction.** One config fully determines a run; seeds pinned; every stage
   emits a content-hashed artifact (resumable, cacheable, auditable).
4. **Measurable by construction.** Every stage emits artifacts the evaluation harness can score.
5. **Data never leaves by default.** Fully self-hostable; remote APIs are opt-in adapters.

### 6.2 Pipeline overview

```
 long-form source (file | stream | yt-dlp)
        │
 [S1] Ingest & preprocess ──► audio (16k mono), shots (PySceneDetect), media meta
        │
 [S2] Transcribe + align ──► word-level transcript (faster-whisper → WhisperX)
        │
 [S3] Diarize ──────────────► speaker turns (pyannote)            ┐
        │                                                          ├─ fused labelled transcript
 [S4] Signal priors ────────► audio energy / laughter / sentiment / scene-cut features
        │
 [S5] Highlight selection ──► ranked candidate segments  ◀──── CORE CONTRIBUTION (hybrid, H1)
        │                       (signal shortlist → LLM rank + rationale → structured JSON)
        │
 [S6] Boundary refinement ──► clean cuts (snap to sentence/pause), hook-first, 15–90s shaping
        │
 [S7] Reframe to 9:16 ──────► per-frame crop path (MediaPipe/YOLO active-speaker, smoothed)  (H2)
        │
 [S8] Caption + style ──────► word-level animated caption track
        │
 [S9] Render ───────────────► final MP4 per clip (single-pass FFmpeg crop+caption)
        │
 [S10] Metadata + score ────► title/desc/hashtags/thumbnail + explainable engagement score (H3)
        │
 output: ranked clips + run report + (optional) OTIO timeline for NLE handoff
```

### 6.3 Shared data model (`Project`) — the inter-stage contract
A single serializable object (proposed: pydantic) threads through all stages:
`source`, `media` (audio/shots), `transcript` (words w/ timestamps), `speakers`, `signals`,
`candidates` (with per-criterion scores + rationale), `clips` (reframe path, caption track,
render path, metadata, engagement score), `run` (config snapshot, seeds, timings, costs,
component versions). This object *is* the unit the evaluation harness consumes. **[OPEN-A]**
pydantic vs dataclass+JSON (lean pydantic).

### 6.4 Stage specifications (objective · approach · rationale · open)
- **S1 Ingest** — *Obj:* normalized audio + shot list. *Approach:* `yt-dlp` + FFmpeg +
  PySceneDetect; optional Redis/Bull queue for scale. *Open:* `[OPEN-B]` VOD/file only vs. live
  capture in v1 (lean VOD first).
- **S2 Transcribe** — *Obj:* word-level transcript. *Approach:* faster-whisper → WhisperX
  alignment. *Rationale:* 4–8× speedup w/ minimal accuracy loss; word timing is required by S6
  (cut precision) and S8 (animated captions). *Open:* `[OPEN-C]` self-host default vs. cloud ASR
  adapter for accuracy-critical jobs.
- **S3 Diarize** — *Obj:* speaker turns. *Approach:* pyannote. *Rationale:* feeds active-speaker
  reframing (S7) and speaker-balanced selection. *Open:* `[OPEN-D]` pyannote license posture.
- **S4 Signal priors** — *Obj:* cheap engagement cues. *Approach:* RMS energy, laughter/applause
  detection, transcript sentiment, scene-cut density. *Rationale:* shortlist before paying for
  LLM/multimodal (core to H1/H4).
- **S5 Selection (core)** — *Obj:* ranked candidates + rationale. *Approach:* signal shortlist →
  LLM ranks on {hook strength, narrative completeness, information density, emotional content} →
  structured JSON. *Open:* `[OPEN-E]` local LLM (Qwen/LLaMA via Ollama) vs. Claude/GPT adapter;
  `[OPEN-F]` the formal clip-worthiness rubric (this rubric IS the eval ground truth — must be
  fixed jointly with §7).
- **S6 Boundary refinement** — snap to sentence/pause boundaries; hook-first ordering; duration
  shaping. *Open:* `[OPEN-G]` contiguous-only vs. stitched/jump-cut clips in v1.
- **S7 Reframe (H2)** — active-speaker crop path; split-screen for simultaneous speakers; static
  center-crop fallback. *Open:* `[OPEN-H]` handcrafted vs. deep saliency; smoothed single path vs.
  speaker-switch cuts.
- **S8 Caption** — word-by-word reveal, keyword/emoji emphasis, templated themes. *Open:*
  `[OPEN-I]` FFmpeg subtitle/ASS filter vs. frame compositing.
- **S9 Render** — single-pass FFmpeg crop+caption (≈35% faster than multi-pass); NVENC option.
- **S10 Metadata + score (H3)** — LLM titles/description/hashtags; explainable engagement score
  from interpretable features (+ rationale). *Open:* `[OPEN-J]` feature set + label source.

### 6.5 Cross-cutting concerns
Caching/resumability (content-hash keys), structured logging + per-stage timing, cost accounting
for cloud adapters, deterministic seeds, single-file YAML config with per-platform presets
(TikTok/Reels/Shorts).

### 6.6 Output interchange: the `.adi` file, Neo4j graph, and DaVinci MCP
*(Full spec: [`docs/ADI_Format_Spec.md`](docs/ADI_Format_Spec.md). Promotes former stretch aim
M7 into a first-class layer.)*

The pipeline's editorial decisions are serialized to a custom **`.adi` (Koza AI DaVinci
Interchange)** file — a **property-graph document with a render projection** that drives three
consumers from one source of truth, all sharing **content-hashed IDs** for full traceability:

1. **Neo4j (bidirectional).** *Downstream:* every run `MERGE`s its `nodes`/`edges`
   (Clip/Segment/Speaker/Topic/… with `DERIVED_FROM`, `FEATURES`, `ABOUT`, `SIMILAR_TO`,
   `CONTINUES` relationships) into a persistent graph — idempotent via the content-hashed IDs.
   *Upstream:* before selection (S5), Koza queries the graph (graph-RAG) for existing
   topics/speakers/near-duplicates/series continuity; the result is injected as `graph_refs` in
   the next `.adi`, steering selection away from redundancy. This closes a feedback loop that a
   flat edit list cannot.
2. **DaVinci Resolve via MCP (editable timeline, human-in-the-loop).** The `.adi` `davinci`
   projection compiles to ordered `davinci-resolve-mcp` tool calls that place clips, apply
   reframe as transform keyframes, attach captions as a subtitle track, and add markers carrying
   each clip's score/rationale — leaving an editable timeline for a human, not an auto-render.
3. **Koza itself** — provenance block (run id, config hash, component versions, seeds) for
   reproducible re-runs.

The "relational data" that makes the graph fruitful = the explicit + **derived** edges
(embedding `SIMILAR_TO`, cross-run `CONTINUES`, speaker co-occurrence), enabling reuse, dedup,
lineage, and graph-informed selection.

---

## 7. Evaluation Plan (summary; full protocol in companion doc)

See **[`docs/Evaluation_Protocol.md`](docs/Evaluation_Protocol.md)** for the executable detail.
Headline design:

- **Datasets:** a curated long-form benchmark (podcasts, talks, streams) with (a) human-labeled
  highlight references and (b) YouTube "most-replayed" as weak labels.
- **Metrics:** highlight relevance (overlap/IoU vs. reference + most-replayed), reframing
  subject-retention (% frames subject in crop), caption WER + alignment error, end-to-end human
  preference (pairwise vs. SaaS outputs), engagement-score correlation, and system metrics
  (TTFC, clips/hour) benchmarked against the **[BvB-2026]** figures (TTFC 5–25 min, ~17–31
  clips/hr).
- **Baselines:** transcript-only selector, multimodal-only selector, static center-crop, and
  (where licensing permits) SaaS reference outputs.
- **Ablations:** signal-priors on/off, LLM local vs. remote, reframe smoothed vs. switch.

---

## 8. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| R1 | Long-source generalization (academic models tuned for ~2.5-min clips) | High | High | Hybrid signal+LLM selector (H1); windowed transcript processing; long-source benchmark |
| R2 | pyannote / model **licensing** for commercial use | Med | High | Track license posture `[OPEN-D]`; provide license-clean fallback (NeMo/Resemblyzer); make diarization optional |
| R3 | Reframing quality on busy multi-speaker scenes (even SaaS struggles — Vizard noted weak) | Med | Med | Active-speaker detection + split-screen; center-crop fallback; subject-retention metric gates regressions |
| R4 | Cost/latency of multimodal or remote LLM at scale | Med | Med | Transcript-first default; signal shortlist before LLM; cost accounting per run |
| R5 | Reproducibility drift across model/dep updates | Med | Med | Pinned env, seeds, content-hashed artifacts, CI that re-runs the E2E slice |
| R6 | Benchmark validity / weak-label noise | Med | Med | Combine human labels + most-replayed; report inter-annotator agreement |
| R7 | Data privacy if cloud adapters used | Low (default off) | High | Self-host default; adapters opt-in; document data-flow per adapter |

---

## 9. Open Decisions (must close before scaling past the E2E slice)
`[OPEN-A]` data-model lib · `[OPEN-B]` VOD vs live · `[OPEN-C]` ASR self-host vs cloud ·
`[OPEN-D]` pyannote licensing · `[OPEN-E]` local vs remote LLM · `[OPEN-F]` clip-worthiness rubric
(blocks S5 + §7) · `[OPEN-G]` contiguous vs stitched · `[OPEN-H]` reframe strategy ·
`[OPEN-I]` caption render method · `[OPEN-J]` engagement-score features/labels ·
`[OPEN-K]` target `davinci-resolve-mcp` tool schema · `[OPEN-L]` reframe-in-Resolve method ·
`[OPEN-M]` captions-in-Resolve method · `[OPEN-N]` `.adi` JSON vs JSONL threshold ·
`[OPEN-O]` embedding model + cross-run `Speaker` identity for `SIMILAR_TO`/dedup.
*(K–O detailed in [`docs/ADI_Format_Spec.md`](docs/ADI_Format_Spec.md) §9.)*

> **Recommended order to resolve:** `[OPEN-F]` (rubric drives selection + eval) →
> `[OPEN-E]`/`[OPEN-C]` (model hosting posture) → `[OPEN-G]`/`[OPEN-H]` (output shape) → rest.

---

## 10. Milestones & Deliverables

| Phase | Milestone | Exit criterion |
|-------|-----------|----------------|
| **M0** | Repo scaffold (src layout, pyproject, configs, CI, LICENSE, CITATION) + `Project` schema | `pip install -e .`; schema validates; CI green |
| **M1** | **Thin E2E slice**: file → faster-whisper → naive top-k selection → center-crop → captions → render | One real video produces ≥1 playable captioned 9:16 clip via one command |
| **M2** | Hybrid selector (A2) + signal priors (S4) + ablations | Beats transcript-only baseline on the benchmark (H1) |
| **M3** | Active-speaker reframing (A3) | Subject-retention beats center-crop (H2) |
| **M4** | Engagement score (A4) + diarization-aware selection | Score correlates with most-replayed weak labels (H3) |
| **M5** | Benchmark + harness (A5) public release | Reproducible numbers; baselines + ablations reported |
| **M6** | Paper + reproducibility release (A6) | One-command repro of headline results; preprint |
| **M7** | `.adi` writer + Neo4j load (downstream catalog) + DaVinci MCP editable timeline (A7) | `.adi` emitted; graph `MERGE`s idempotently; clips land on an editable Resolve timeline via MCP **[BvB-2026 §3]** |
| **M8** | Neo4j upstream feedback (graph-RAG into S5) | Selection consumes `graph_refs`; measurable dedup / series-continuity improvement |

---

## 11. Reproducibility & Open-Science Commitments
Pinned dependencies + lockfile; fixed seeds; deterministic stage artifacts; all configs version-
controlled; one-command reproduction of every figure/number in the paper; permissive OSS license;
`CONTRIBUTING.md`, issue/PR templates, CI; `CITATION.cff`; public benchmark + leaderboard.

---

## 12. Immediate Next Steps
1. Resolve `[OPEN-F]` (clip-worthiness rubric) and `[OPEN-E]`/`[OPEN-C]` (model hosting) — these
   unblock the core selector and the evaluation.
2. Approve the `Project` schema (§6.3) as the inter-stage contract.
3. Scaffold the repo (M0) and build the thin E2E slice (M1) to validate pipeline shape before
   optimizing any single stage.

---

## References
- **[BvB-2026]** *Build vs. Buy: Auto-Generated Video Clipping & AI-Driven NLE — An Enterprise
  Research Report* (2026-06-19). Internal source PDF in repo root. Parts 1–4 cited inline.
- Google **AutoFlip** — open framework for intelligent video reframing (saliency + object tracking).
- **`KazKozDev/auto-vertical-reframe`** — face/pose/saliency vertical reframing CLI.
- Highlight detection: **Moment-DETR**, **MH-DETR**; unsupervised audio-visual recurrence
  (WACV 2025); length-controlled "most-replayed" highlighting (CHI 2025).
- Components: **faster-whisper** (CTranslate2), **WhisperX**, **pyannote.audio**, **MediaPipe**,
  **YOLOv8**, **SAM**, **FFmpeg**, **MoviePy**, **OpenTimelineIO**; **MCP** (`premiere-pro-mcp`,
  `davinci-resolve-mcp`).

*(External references to be upgraded to full citations during the paper-writing phase, A6.)*

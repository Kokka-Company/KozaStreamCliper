# `.adi` Format & Viral-Clip Pipeline — Design Spec v0.1

**Purpose:** Define a feature-capture file format (`.adi`) and the analysis layer that turns a long stream VOD (`.mov`) into ranked, DaVinci-ready viral clips.

**Pipeline at a glance:**

```
.mov  ─► extractor ─► .adi (flat signals) ─► loader ─► Neo4j (moments + relationships) ─► ranking query ─► DaVinci Resolve MCP
 +chat                                                                                        │
 +facecam                                                                                     └─► {in_tc, out_tc, title}[]
 +HUD
```

The division of labor is deliberate:

- **`.adi`** is the *capture* artifact — dense, flat, per-timestamp signals. Portable, diff-able, reproducible.
- **Neo4j** is the *reasoning* layer — detected moments and the relationships between them (co-occurrence, callbacks, entities). This is what a flat file cannot express.
- **DaVinci** is the *execution* layer — it only ever receives a list of timecodes + titles.

---

## 1. Inputs

This spec assumes all four signal sources are available:

| Source | What it gives you | Why it matters |
|---|---|---|
| `.mov` audio | energy, laughter, vocal excitement | hype detection, reaction peaks |
| `.mov` video | scene/shot changes, facecam region | visual cuts, streamer reaction |
| Chat log | message velocity, emote spam, copypasta | **strongest single virality signal** |
| Game HUD/overlay | killfeed, score, events via OCR | ties hype to in-game cause |

Chat must be time-aligned to the VOD clock. Twitch/YouTube/Kick exports give per-message timestamps; confirm the offset between chat time and video time (recording start vs. stream start often differ).

---

## 2. The `.adi` Format

### 2.1 Container choice

**v0.1 = JSON** (one file, UTF-8). Human-readable, trivial to extend, easy to version-control alongside the extractor. Migrate to SQLite-in-a-`.adi`-wrapper only when VOD length makes the JSON unwieldy (multi-hour streams with word-level transcripts can get large). Schema below is storage-agnostic, so the migration is mechanical.

### 2.2 Top-level structure

```json
{
  "adi_version": "0.1",
  "source": {
    "file": "vod_2026-06-28.mov",
    "duration_s": 14820.4,
    "fps": 60.0,
    "resolution": [1920, 1080],
    "chat_offset_s": -3.2,
    "regions": {
      "facecam": [1440, 760, 480, 320],
      "hud_killfeed": [1400, 80, 500, 200]
    }
  },
  "tracks": {
    "audio_energy":   [ ... ],
    "audio_spikes":   [ ... ],
    "laughter":       [ ... ],
    "scene_changes":  [ ... ],
    "transcript":     [ ... ],
    "chat":           [ ... ],
    "chat_bursts":    [ ... ],
    "facecam":        [ ... ],
    "hud_events":     [ ... ]
  }
}
```

Signals live in named **tracks** (arrays of events). Every event carries a timestamp in seconds on the VOD clock. Keep dense/raw data sampled (e.g. energy at 10 Hz) and event data sparse.

### 2.3 Track schemas

**audio_energy** — dense, sampled envelope (used for relative scoring, not stored as events per peak):
```json
{ "t": 12.40, "rms": 0.42, "peak": 0.61 }
```

**audio_spikes** — detected onsets/peaks above adaptive threshold:
```json
{ "t": 12.40, "magnitude": 0.91, "kind": "onset" }
```

**laughter** — from an audio event classifier (e.g. YAMNet/PANNs):
```json
{ "start": 44.1, "end": 47.8, "confidence": 0.86, "intensity": 0.7 }
```

**scene_changes** — shot boundaries:
```json
{ "t": 45.0, "confidence": 0.88, "kind": "cut" }
```

**transcript** — word-level timestamps, speaker-tagged where possible:
```json
{ "start": 0.5, "end": 4.2, "text": "no way that just happened",
  "speaker": "host", "words": [{"w":"no","s":0.5,"e":0.7}, ...],
  "sentiment": 0.8, "highlight_phrase": true }
```

**chat** — raw messages (may be downsampled if huge):
```json
{ "t": 50.3, "user": "viewer123", "msg": "KEKW KEKW",
  "emotes": ["KEKW"], "is_copypasta": false }
```

**chat_bursts** — *derived* windows where message rate / emote density spikes (this is the high-value track):
```json
{ "start": 49.0, "end": 58.0, "msgs_per_s": 14.2, "baseline_mult": 6.1,
  "top_emotes": ["KEKW","LULW"], "kind": "emote_spam" }
```

**facecam** — reaction detection on the webcam region:
```json
{ "t": 46.2, "expression": "surprise", "confidence": 0.79, "motion": 0.6 }
```

**hud_events** — OCR/template-matched in-game events:
```json
{ "t": 45.5, "type": "kill", "detail": "triple", "confidence": 0.7 }
```

### 2.4 Extraction tooling (per track)

| Track | Tool |
|---|---|
| audio extract | `ffmpeg` → wav |
| audio_energy / audio_spikes | `librosa` (RMS, onset detect, peak pick) |
| laughter | `YAMNet` / `PANNs` audio event classifier |
| scene_changes | `PySceneDetect` (content detector) |
| transcript (+words, +diarization) | `faster-whisper` + `pyannote` for speakers |
| sentiment / highlight phrases | lightweight text classifier + keyword list ("no way", "let's go", "oh my god") |
| chat / chat_bursts | parse export (JSON/IRC), rolling-window velocity + emote density |
| facecam | crop region → `mediapipe`/`opencv` expression model |
| hud_events | crop region → `tesseract`/`easyocr` + template match |

---

## 3. Neo4j Graph Model

You do **not** load raw tracks into Neo4j. You first run **moment detection** (Section 4) to collapse co-occurring signals into candidate `Moment` nodes, then load moments + their relationships. The graph's job is ranking and *assembly*, not storing time series.

### 3.1 Nodes

```
(:VOD     {id, file, duration_s, recorded_at})
(:Moment  {id, start, end, peak_t, score, type})
(:Signal  {kind, t, magnitude})          // attached to a moment, not global
(:Utterance {text, start, end, sentiment})
(:Speaker {name})
(:Entity  {name, type})                  // person | game | meme | topic | place
(:Clip    {id, title, start, end, status})  // a produced/proposed clip
```

### 3.2 Relationships

```
(VOD)-[:HAS]->(Moment)
(Moment)-[:HAS_SIGNAL]->(Signal)
(Moment)-[:CONTAINS]->(Utterance)
(Utterance)-[:SPOKEN_BY]->(Speaker)
(Moment)-[:MENTIONS]->(Entity)
(Moment)-[:NEXT]->(Moment)               // temporal adjacency
(Moment)-[:CO_OCCURS {window_s}]->(Moment)
(Moment)-[:CALLBACK_TO {gap_s}]->(Moment) // running joke / setup→payoff
(Clip)-[:INCLUDES]->(Moment)
```

### 3.3 What the graph buys you (vs. a flat file)

- **Multi-signal hype scoring** — a `Moment` with audio_spike + laughter + chat_burst + facecam-surprise all attached scores far higher than a lone loud peak. This is a one-hop aggregation in Cypher.
- **Callbacks / narrative clips** — a phrase at `0:12:00` echoed at `2:45:00` becomes a `CALLBACK_TO` edge, letting you assemble a clip that includes *setup + payoff* instead of just the loud moment. This relationship is impossible to query cheaply in a flat timeline.
- **Entity threading** — `MATCH (m)-[:MENTIONS]->(:Entity {name:'guest_x'})` pulls every moment a guest/game/meme appears in for themed compilations.
- **Sequencing** — `NEXT` chains let you grow a clip outward to a natural cut point.

---

## 4. Moment Detection & Virality Scoring

### 4.1 Detection (pre-Neo4j, in the loader)

1. Build a unified timeline of all sparse events from the `.adi`.
2. Slide a window (e.g. 8–15 s). For each window, gather signals present.
3. Where ≥ N signal types co-occur (or any single signal exceeds a hard threshold), emit a `Moment` spanning the union, with `peak_t` at the densest point.
4. Attach each contributing signal as a `Signal` node.

### 4.2 Scoring (in Neo4j)

Weighted sum of normalized signal magnitudes, with chat weighted highest:

```cypher
MATCH (m:Moment)-[:HAS_SIGNAL]->(s:Signal)
WITH m,
  sum(CASE s.kind
        WHEN 'chat_burst' THEN 3.0 * s.magnitude
        WHEN 'laughter'   THEN 2.0 * s.magnitude
        WHEN 'audio_spike' THEN 1.5 * s.magnitude
        WHEN 'facecam'    THEN 1.5 * s.magnitude
        WHEN 'hud_event'  THEN 1.0 * s.magnitude
        ELSE 0.5 * s.magnitude END) AS score
SET m.score = score
RETURN m ORDER BY score DESC LIMIT 50
```

Weights are tunable — treat them as the main knob you'll iterate on against real "did it pop" outcomes.

### 4.3 Padding to a cuttable clip

A scored moment is a *peak*, not a clip. Expand to head/tail handles (e.g. start 3–5 s before first signal, end at the next `scene_change` or `NEXT` moment, capped at a max length for the target platform — ~ 20–60 s).

---

## 5. DaVinci Resolve Handoff

The final query emits a flat list; the DaVinci MCP consumes only this:

```json
[
  { "in_tc": "00:12:04:30", "out_tc": "00:12:38:12",
    "title": "no way triple kill", "score": 0.94 },
  { "in_tc": "00:45:01:00", "out_tc": "00:45:29:20",
    "title": "chat goes feral", "score": 0.89 }
]
```

Timecodes are HH:MM:SS:FF at the VOD's fps (convert from seconds using `source.fps`). The MCP then imports the media, creates a timeline, and lays in/out points per entry — optionally one timeline per clip, or all clips on a single "highlights" timeline. Keep DaVinci dumb: all selection logic stays upstream so you can re-rank without touching the editor.

---

## 6. Tech Stack

- **Runtime:** Python 3.11+
- **Media:** `ffmpeg`, `librosa`, `PySceneDetect`, `opencv`, `mediapipe`
- **Speech:** `faster-whisper`, `pyannote.audio` (diarization)
- **Audio events:** `YAMNet`/`PANNs`
- **OCR:** `easyocr` or `tesseract`
- **Graph:** `neo4j` (server) + `neo4j` Python driver
- **Orchestration:** one CLI (`adi extract`, `adi load`, `adi rank`) wrapping the stages

---

## 7. Open Questions / Decisions Before Build

1. **Platform of the VOD** (Twitch/YouTube/Kick) — determines chat export format and emote sets.
2. **GPU available?** — faster-whisper + pyannote on multi-hour VODs is much faster on GPU; sets realistic runtime.
3. **Target clip platform** (TikTok/Shorts/Reels) — fixes max length and aspect-ratio reframing (the facecam region may become the vertical focus).
4. **Is Neo4j worth it for v1, or start with the `.adi` + a scoring script?** Recommendation: build `.adi` + flat scorer first, add Neo4j once you want callbacks/entity threading — the graph pays off most on the *relational* features, which are v2 polish.
5. **Ground truth** — do you have past clips that went viral to tune the scoring weights against?

---

*Spec v0.1 — feature capture (`.adi`) and graph analysis layer for automated viral-clip extraction.*

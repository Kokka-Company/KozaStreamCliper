# `.adi` — Koza AI DaVinci Interchange — Format Specification

**Companion to [`../Draft_Architecture.md`](../Draft_Architecture.md).**

| | |
|---|---|
| **Format name** | Koza AI DaVinci Interchange (`.adi`) |
| **Status** | `DRAFT v0.1` — designing now; schema is proposed, not frozen |
| **Purpose** | A single, typed file emitted by Koza's Python orchestrator that drives **three** consumers from one source of truth: (1) the **Neo4j** property graph (bidirectional — catalog + feedback), (2) the **DaVinci Resolve MCP** layer (builds an *editable* timeline, human-in-the-loop), and (3) re-runs of Koza itself (provenance/idempotency). |

> **Core design idea.** `.adi` is a **property-graph document with a render projection**. The
> graph part (`nodes`/`edges`) loads directly into Neo4j; the `davinci` projection compiles to
> MCP tool calls; stable content-hashed IDs make every Neo4j node, every Resolve timeline item,
> and every Koza artifact **the same identity** — full traceability across all three systems.

---

## 1. Design requirements (why the format looks the way it does)

| Req | Driver |
|-----|--------|
| **Property-graph native** (nodes/edges with labels + props) | Neo4j loads it via `MERGE` with zero remodeling |
| **Stable, content-hashed IDs** | Idempotent re-imports (enrich, not duplicate); 1:1 map graph ↔ timeline ↔ artifact |
| **Frame-accurate + seconds timecodes** | DaVinci needs frames@fps; the graph/analysis layer wants float seconds |
| **Editable-timeline projection** (not a flat render) | DaVinci step is human-in-the-loop; carries rationale as markers |
| **Graph references (read-back)** | Upstream feedback: `.adi` can cite existing graph entities (series, recurring speakers/topics) |
| **Provenance block** | Reproducibility (run id, config hash, component versions, seeds) |
| **Versioned schema** | `version` field gates parser behavior as the format evolves |

---

## 2. Container & encoding
- **Encoding:** UTF-8 JSON. For very long sources, a **JSONL sidecar** (`*.adi.nodes.jsonl`,
  `*.adi.edges.jsonl`) MAY stream nodes/edges; the root `.adi` then references the sidecars.
- **Top-level sections:** `meta`, `source`, `graph_refs`, `nodes`, `edges`, `davinci`.
- **IDs:** `"<type>_<base32(blake2s(natural_key))[:8]>"`, e.g. `clip_k7f3a9q2`. Natural keys are
  defined per node type in §4 so the *same* logical entity always hashes to the same ID.

---

## 3. Skeleton

```jsonc
{
  "version": "adi/0.1",
  "meta": {
    "run_id": "run_2026_06_30_8814",
    "config_hash": "cfg_3df1...",
    "created_at": "2026-06-30T14:02:11Z",
    "components": { "asr": "faster-whisper@1.x", "diarize": "pyannote@3.x", "llm": "qwen2.5@..." },
    "seeds": { "global": 42 }
  },
  "source": {
    "id": "src_a1b2c3d4", "uri": "s3://.../stream.mp4",
    "duration_s": 5412.3, "fps": 30000/1001, "resolution": [1920,1080], "content_hash": "..."
  },
  "graph_refs": {                         // UPSTREAM feedback — entities already in Neo4j
    "series_id": "series_kokka_pod",
    "recurring_speakers": ["spk_host"],
    "seen_topics": ["top_ai_ethics"],
    "dedup_against": ["clip_prev_113"]    // avoid re-cutting near-duplicates
  },
  "nodes": [ /* §4 */ ],
  "edges": [ /* §5 */ ],
  "davinci": { /* §6 */ }
}
```

---

## 4. Node types (Neo4j labels)

Each node: `{ "id", "type", ...properties }`. `type` becomes the Neo4j label.

| Type (label) | Natural key (for ID hash) | Key properties |
|--------------|---------------------------|----------------|
| `SourceMedia` | uri + content_hash | `uri, duration_s, fps, resolution` |
| `Segment` | source_id + t_in + t_out | `t_in_s, t_out_s, kind` (topic/scene) |
| `Utterance` | source_id + t_in + speaker | `text, t_in_s, t_out_s` |
| `Speaker` | series_id + diarization_label (or voiceprint) | `label, display_name?` |
| `Topic` | normalized topic string | `label, taxonomy_id?` |
| `Clip` | source_id + t_in + t_out | `t_in_s, t_out_s, fps, frame_in, frame_out, hook, title, score, rationale` |
| `ReframePath` | clip_id | `keyframes:[{t_s, cx, cy, zoom}], strategy` |
| `CaptionTrack` | clip_id | `words:[{text, t_in_s, t_out_s}], style_id` |
| `EditOp` | clip_id + op + ordinal | `op, params` |
| `Timeline` | run_id + name | `name, width, height, fps` |
| `RenderTarget` | clip_id + platform | `platform, aspect, preset` |

> **Time fields:** every time-bearing node carries **both** `*_s` (float seconds, canonical for
> analysis/graph) **and** frame-accurate `frame_*` + `fps` (canonical for DaVinci). Converters
> live in the Python layer; the file stores both to prevent drift.

---

## 5. Edge types (relationships)

Each edge: `{ "from", "rel", "to", ...properties }`. This is the **relational data** that makes
the graph fruitful — without it Neo4j just holds disconnected records.

```
(SourceMedia)-[:HAS_SEGMENT]->(Segment)
(Segment)-[:HAS_UTTERANCE]->(Utterance)
(Utterance)-[:SPOKEN_BY]->(Speaker)
(Clip)-[:DERIVED_FROM]->(Segment)
(Clip)-[:FEATURES]->(Speaker)
(Clip)-[:ABOUT {weight}]->(Topic)
(Clip)-[:HAS_REFRAME]->(ReframePath)
(Clip)-[:HAS_CAPTIONS]->(CaptionTrack)
(EditOp)-[:APPLIES_TO]->(Clip)
(Timeline)-[:CONTAINS {order}]->(Clip)
(Clip)-[:RENDERS_TO]->(RenderTarget)
(Clip)-[:SIMILAR_TO {score, method}]->(Clip)   // DERIVED via embeddings (enrichment)
(Clip)-[:CONTINUES {series_id}]->(Clip)         // DERIVED across runs (series continuity)
```

**Derived edges are where "fruitfulness" comes from.** `SIMILAR_TO`, `CONTINUES`, speaker
co-occurrence, and topic links are computed in the Python enrichment step (embeddings + graph
queries) and written alongside the literal edges — turning a flat edit list into a traversable
graph that supports reuse, dedup, lineage, and graph-RAG selection.

---

## 6. The `davinci` projection (→ MCP, editable timeline)

Compiles to ordered DaVinci Resolve MCP tool calls (see report **[BvB-2026 §3]**:
`davinci-resolve-mcp` exposes hundreds of tools). Editable = clips placed + reframe as **transform
keyframes** + captions as a **subtitle track** + **markers** carrying rationale; *no auto-render*.

```jsonc
"davinci": {
  "project": "Koza_2026_06_30",
  "timeline": { "name": "Koza_Shorts", "width": 1080, "height": 1920, "fps": 30000/1001 },
  "ops": [
    { "tool": "import_media",            "args": { "path": "src_a1b2c3d4" } },
    { "tool": "create_timeline",         "args": { "ref": "Koza_Shorts" } },
    { "tool": "append_clip_to_timeline", "args": { "clip_id": "clip_k7f3a9q2",
        "src_in_frame": 18372, "src_out_frame": 19785 } },
    { "tool": "set_clip_transform",      "args": { "clip_id": "clip_k7f3a9q2",
        "keyframes_ref": "reframe_of:clip_k7f3a9q2" } },     // ReframePath → Pan/Tilt/Zoom
    { "tool": "add_subtitle_track",      "args": { "clip_id": "clip_k7f3a9q2",
        "captions_ref": "captions_of:clip_k7f3a9q2" } },
    { "tool": "add_marker",              "args": { "clip_id": "clip_k7f3a9q2",
        "color": "Blue", "note": "score=0.81 — hook: '...'; rationale: ..." } }
  ]
}
```
- `*_ref` values resolve to node IDs in §4 → **same identity as the graph**.
- Tool *names* are placeholders to be mapped to the actual `davinci-resolve-mcp` schema during
  implementation (`[OPEN-K]`).

---

## 7. Loading into Neo4j (idempotent)

One-time constraints (natural-key uniqueness → safe re-imports):
```cypher
CREATE CONSTRAINT IF NOT EXISTS FOR (c:Clip)     REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT IF NOT EXISTS FOR (s:Speaker)  REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT IF NOT EXISTS FOR (t:Topic)    REQUIRE t.id IS UNIQUE;
// ... per node type
```
Load (Python emits this from `nodes`/`edges`, or via `LOAD CSV` for bulk):
```cypher
UNWIND $nodes AS n  CALL apoc.merge.node([n.type], {id:n.id}, n.props) YIELD node RETURN count(*);
UNWIND $edges AS e
  MATCH (a {id:e.from}), (b {id:e.to})
  CALL apoc.merge.relationship(a, e.rel, {}, e.props, b) YIELD rel RETURN count(*);
```
Re-running a source updates properties and adds new edges **without duplicating** — because IDs
are content-hashed (§2).

---

## 8. The bidirectional loop (how feedback closes)

```
            ┌──────────────────── upstream feedback ───────────────────┐
            │ graph_refs: series, recurring speakers/topics, dedup set  │
            ▼                                                           │
[Koza pipeline S1–S10] ── emits ──► (.adi) ──► Neo4j MERGE (catalog) ──┘
            │                          │
            │                          └──► DaVinci MCP ──► editable Resolve timeline (human finishes)
            ▼
   selection (S5) queries Neo4j (graph-RAG): "what topics/speakers/clips already exist?
   what's near-duplicate? what continues the series?" → better, non-redundant picks
```
- **Downstream (catalog):** every run writes its clips/entities/edges to the graph.
- **Upstream (feedback):** before selection, S5 queries the graph and the result is injected as
  `graph_refs` in the *next* `.adi`, steering away from duplicates and toward series continuity.

---

## 9. Open decisions (add to Draft_Architecture §9)
- `[OPEN-K]` Exact `davinci-resolve-mcp` tool names/params to target (audit the server schema).
- `[OPEN-L]` Reframe in Resolve: transform keyframes vs. Resolve "Smart Reframe" vs. input-sizing.
- `[OPEN-M]` Captions in Resolve: subtitle track (editable, simple) vs. Fusion Text+ (animated, complex).
- `[OPEN-N]` JSON single-file vs. JSONL sidecars threshold for large sources.
- `[OPEN-O]` Embedding model + threshold for `SIMILAR_TO`/dedup; voiceprint identity for cross-run `Speaker`.

---

## References
See [`../Draft_Architecture.md`](../Draft_Architecture.md) §References. DaVinci/MCP specifics:
**[BvB-2026 §3]** (`davinci-resolve-mcp`, MCP tool-schema model).

# The Memery System

## Overview
The Memery system is a two-layer system to enhance local AI agents:
1. **The Infinite Approximator (IA):** A compression engine that trades processing time for effective context size, allowing local models to process arbitrarily large inputs by breaking them into overlapping chunks, compressing each into opaque token sequences, and stitching them together at semantic seams — no fine-tuning required.
2. **The VibeNcoder:** A tiered long-term memory system that captures everything during active use (zero LLM calls), then batch-processes the day's raw data during idle time into scored, summarized, and embedded memories stored in a single SQLite file — mimicking the biological pattern of waking capture and sleep consolidation.

---

## The Infinite Approximation Engine

### Goal
To allow local models to process arbitrarily large inputs/tasks (did someone say near-infinite (sequential) sub-agents?) by trading processing time for an effective mental-bandwidth boost. The approximator breaks ideas down into byte-size chunks near the size of its context window.

### Core Idea: Context Preservation via Compression
- **"Untrained" (model agnostic) Self-Compression:** Infinite Approximation exploits an LLM's latent ability to compress text into opaque token sequences without fine-tuning (it can simplify content beyond human recognition and still "recall" the core of the original text). This provides an immediate, zero-setup path for context extension; it will likely require some fine-tuning per model, but nothing like training from scratch.
- **Syntactic & Semantic Hybrid:**
  - **Opaque Encoding:** Compresses ideas that are too big into special LLM gibberish as a precursor to stitching together multiple notions.
  - **Stable Lookup Maps/Static content insertion (TBA):** Handles recurring/static content (scaffolding, templates, code patterns, passwords anybody??) losslessly via string manipulation.

### Mechanism: The Quilting Pipeline
1. **Patches & Overlap Zones:** Input is divided into fixed-size segments near the model's attention span. Segments overlap to ensure full visibility of text context at boundaries, preventing incomplete ideas. `[longformer]` `[unlimiformer]`
2. **Seam Selection & Stitching:** Within overlaps, the system identifies natural semantic boundaries (sentence end, paragraph break, topic shift) to arrange notions in the most space-efficient and logically coherent way; this seam detector will probably be a smaller, separate model (SaT).
3. **The Quilting Pass:** The system processes notions sequentially, picking seams in overlap zones and chaining compressed ideas. Ideally the whole input is processed in a single pass, with overhead proportional to overlap depth.

---

## VibeNcoder: The Memory Store

### Philosophy: Memory over Storage
**'Remembered' is automatic. 'Remembering' is selective.** Every handler prompt triggers a memory lookup (pure math, zero LLM calls). Only information that earns its spot through importance and reinforcement gets stored long-term; the mechanism for bootstrapping importance is TBD, likely hard-coded files with prebuilt parameters for the model to ingest on first-run. This also makes it possible to share plug-and-play skillsets, knowledgebases and personalities.

When retrieval confidence is low, the system doesn't guess or go silent; it can trigger the agent to ask clarifying questions. This serves double duty: gives the agent a mechanism to identify unknowns, and gives us an opportunity to log a weighted response. Even low-weight user feedback is worth saving; at worst it is forgotten naturally.

The store is composed of two layers:
- **The Fossil Record:** Raw source notes (.md) are preserved and heavily compressed forever as the ground truth. Never deleted.
- **The "Memery" Store:** A derived SQLite database containing increasingly lossy summaries, embeddings, and relational metadata. Everything here is rebuildable from the fossils.

In CoALA's taxonomy, this provides **working memory** (assembled context), **episodic memory** (raw notes and summaries), and **semantic memory** (entities, relationships, master topic entries). `[coala]`

### Semantic Hierarchy (Tiered Memory)
Inspired by hardware memory hierarchies (registers → cache → RAM → disk), content is organized by accessibility and persistence. `[soar]` `[demandpaging]`

1. **The Kernel Layer (The Soul):** The agent's DNA and presence of mind. S-tier persistence; some core rules are literally immutable. Critical elements from the Heart may be bumped up in extreme cases; non-critical instructions may bump down if redundant or under-utilized.
2. **The Persona Layer (The Heart):** Internal monologue, self-reminders, and subtext/tone logging. Tracks the "Vibe" of past interactions to modulate register (tone, formality, subtext). Low-decay; enlightened ideas can bump up to the Soul layer.
3. **Priority & Regular Layers:**
   - **Conscious Layer:** High-relevance + active context. Memories enter/leave dynamically based on current project or conversation. Also a landing area for daily info likely to be stored long-term.
   - **Unconscious Layer:** External database; bulk of stored memories. Subject to the full meme half-life cycle, likely with more parameters than upper layers. `[act-r]`

### Memery Metabolism: The Path of Data
Information moves through a lifecycle of varying densities. `[timem]` `[oblivion]`
- **Level 0 (Working):** Active, high-fidelity (compressed) summaries and embeddings in the retrieval layer.
- **Level 1 (Dormant):** Extra-crushed or **Opaque-encoded (IA)** representation. Designed to be symbiotic with the VibeNcoder system, ensuring register-modulation and tone-persistence survive extreme compression.
  - *Future:* **Typoglycemia Compression** — LLMs can parse very degraded text ie; "Aoccdrnig to rscheearch". `[typoglycemia]`
- **Level 2 (Synthesized):** Merged into big memes (still compressed, possibly more compressed) during Nightly Consolidation. `[raptor]`
- **Level 3 (Archive):** Maximum-compression fossil record for ground-truth audits and system re-indexing.
- **Level 99 (Death):** Pruning occurs when information entropy falls below a utility floor. The system releases the memories back into the wild. `[nepenthe]`

### Pipeline: Ingest & Retrieval
**Ingest (Selective & Structured):**
- **Extraction:** Fast LLM pass for tags, entities, relationships, and importance; primary LLM creates compressed summaries.
- **Chunking (IA):** Use the Infinite Approximator when required to ingest big bytes of info.
- **Canonicalization:** Write-time normalization of tags and entities (unicode normalization, lowercasing, etc) to prevent data drift. Multi-language support is a MUST. `[engram]`

**Retrieval (Automatic & Zero-LLM):**
- **Integrated Scoring:** Combining independent signals: `[gen-agents]`
  - **Similarity** — likely most important signal
  - **Importance** — ingest-time judgment; varies on other signals
  - **Recency** — ideas decay over time; fortnight half-life or something similar
  - **Frequency/Tags/Entities** — metadata-based score bonuses
- **Score Floor:** Reject related but unimportant memories. `[engram]`
- **Expansion-Testing (PER):** Before deleting old memes, ponder them for a moment to ensure they are truly irrelevant. `[per]`

### Database Schema (SQLite)
<!-- render: variables -->
<!-- render: schema -->
```sql
CREATE TABLE memories (
    id INTEGER PRIMARY KEY,
    source_file TEXT NOT NULL,
    source_date DATE NOT NULL,
    chunk_index INTEGER DEFAULT 0,
    summary TEXT NOT NULL,
    importance REAL DEFAULT 0.5,
    confidence REAL DEFAULT 0.5,
    consolidated BOOLEAN DEFAULT FALSE,
    parent_id INTEGER REFERENCES memories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_accessed TIMESTAMP,
    access_count INTEGER DEFAULT 0
);

CREATE TABLE tags (memory_id INTEGER REFERENCES memories(id), tag TEXT NOT NULL);
CREATE TABLE entities (memory_id INTEGER REFERENCES memories(id), entity TEXT NOT NULL, role TEXT);
CREATE TABLE relations (entity1 TEXT, relation TEXT, entity2 TEXT, source_memory_id INTEGER);
CREATE TABLE embeddings (memory_id INTEGER UNIQUE, vector BLOB NOT NULL);
CREATE TABLE forgotten_sources (source_file TEXT PRIMARY KEY, forgotten_at TIMESTAMP);
CREATE VIRTUAL TABLE memories_fts USING fts5(summary, content=memories, content_rowid=id);

CREATE INDEX idx_tags_tag ON tags(tag);
CREATE INDEX idx_entities_entity ON entities(entity);
CREATE INDEX idx_memories_date ON memories(source_date);
CREATE INDEX idx_memories_importance ON memories(importance);
CREATE INDEX idx_memories_parent ON memories(parent_id);
```

### Rules & Guardrails
1. **Never delete raw source notes.** They are the only sane record.
2. **Summaries are lossy by design.** Approximate recall is the default; drill-down for exact audit.
3. **Benchmark the whole pipeline.** Final answer quality is the only metric that matters.
4. **Total Overhead Accounting (TQ):** Count all bytes (embeddings, metadata, indexes) against the budget. `[turboquant]`
5. **Outlier-First Quantization (TQ):** Protect high-relevance seam zones and Vibe-sensitive targets from aggressive "crushing."
6. **Data-Oblivious Design (TQ):** Favor online methods that process memories independently without needing corpus-wide retraining.

---

## Why.?

The Infinite Approximator lives in the **working memery layer** and serves two roles in the VibeNcoder pipeline:

- **At ingest:** Long notes are processed through IA's quilting pipeline before summarization — richer input than truncation. `[ia]`
- **At retrieval:** When retrieved context overflows the prompt window, IA compresses overflow rather than dropping it. All 12 memories instead of top 5.
- **In metabolism:** IA's opaque encoding is the representation at Level 1 (Dormant) — memories that are too cold for full fidelity but too valuable to merge or archive yet. The Persona layer's register/tone data is specifically protected during this compression (Outlier-First Quantization).
- **Chain reading:** The agent walks compressed chains left-to-right. Map entries expand instantly; opaque blobs cost one LLM call per patch.

### Unknowns
<!-- render: risk -->
- **Encoding Instability:** Encoding is non-deterministic and each model will have its own unique output (isn't that cute). Each model will act like an idiosyncratic encryption key; memery transfers require either decoding the database with the current agent; or re-encoding the archival data with the new Ai.
- **Fidelity Degradation:** Extreme re-compression eventually becomes noise. Requires specific tuning per model (isn't that cute); on the upside it can be a useful tool for when to discard information when tuned properly.
- **Content Outliers:** Not all patches are equally compressible. Dense passages or precise quantities may require richer encoding or exemption (Lossless Passthrough). Conversely, we will also experiment with finding the floor here; if we can reliably break problems down into their smallest processable components as part of the system itself, then we will have something truly abstract.

---

## References
- **[engram]** Cheng et al. (2026): Canonicalization, gating, and independent-signal scoring.
- **[turboquant]** Zandieh et al. (2025): Total overhead accounting, outlier protection, and end-to-end evaluation.
- **[gen-agents]** Park et al. (2023): Recency/Importance/Relevance scoring and Reflection.
- **[memgpt]** Packer et al. (2023): Tiered virtual memory paging architecture.
- **[soar]** Laird (2022): Cognitive architecture with distinct memory tiers.
- **[act-r]** Anderson (1991): Base-level activation tied to frequency and recency.
- **[coala]** Sumers et al. (2023): Cognitive Architectures for Language Agents.
- **[timem]** (2026): Progressive consolidation into hierarchical summaries.
- **[oblivion]** (2026): Decay-driven activation; forgetting as reduced accessibility.
- **[demandpaging]** (2025): Literal demand paging for LLM context windows.
- **[nepenthe]** (2024): Entropy-based pruning floor.
- **[typoglycemia]** (2025): Semantic reconstruction under word-form corruption.
- **[per]** Schaul (2015): Prioritized experience replay as a valuation gate.
- **[quilting]** Efros & Freeman (2001): Conceptual inspiration for patch-based stitching.
- **[icae] / [gist] / [500x] / [ztokens]**: Validation of self-compression into latent codes/opaque tokens.
- **[compressive]** Rae et al. (2019): Proved lossy compression beats dropping context.
- **[autocomp] / [unlimiformer] / [homer]**: Segment-and-carry-forward architectures.
- **[longformer]** Beltagy et al. (2020): Sliding window and cross-window information principles.
- **[sat] / [lumberchunker]**: Boundary detection and semantic segmentation tools.
- **[streamingllm]** Xiao et al. (2024): Zero-shot recency preservation via attention sinks.
- **[llmlingua]** Jiang et al. (2023): Perplexity-based token pruning.
- **[raptor]** Sarthi et al. (2024): Hierarchical summarization trees.


# The Memery System

*Two subsystems, one problem: infinite context with intelligent organization.*

---

## Overview

The Memery System is a memory architecture for AI assistants. It combines two components neither of which work alone at scale:

**The vibeNcoder**; a memory system where the AI organizes knowledge by its own contextual judgment. It captures everything during active use (no LLM calls, no filtering); then batch-processes the day's buffer during idle time into scored summaries with metadata. Importance, decay, consolidation; all decided by the LLM's 'vibes': what surprised it, what mattered relative to everything else it saw that day, what patterns it noticed across the full context.

Everything lives in a SQLite file. Retrieval is automatic at every prompt; zero LLM calls on read, zero on capture, batch LLM on process.

 
**The Infinite Approximator (IA)**; an attention augmentation system that takes whatever vibeNcoder retrieves and fits it into working memory by quilting compressed patches at semantic seams. Trade processing time for effective context size. `[cogload]` Zero-shot, no fine-tuning, no extra parameters.

The symbiosis is the point. vibeNcoder without IA retrieves more than you can fit; great memories that hit a context wall. The AI remembers intelligently *and* reads it all back. Arbitrarily long contexts into any given prompt.

Compute runs on GPU at full throughput. Cold state (staged KV entries, hibernated agent contexts) lives in CPU RAM. Transfers overlap with generation rather than blocking it; outperforming GPU-only (VRAM exhaustion) and CPU-only (slow generation).

---

## Design Philosophy

---

### The vibeNcoder principle:

The system doesn't ask the user what's important. It watches, infers, and judges. Importance scores, open questions, and consolidation priorities are all determined by the LLM's own contextual reading; its vibes about what matters relative to everything else it saw that day. No rigid rules. No user-tagged metadata. The AI's contextual judgment *is* the organizing force.

**Remembering is cheap; making memeries is expensive.** Retrieval is cheap; zero LLM calls, milliseconds. Raw capture is cheap; zero LLM calls, microseconds. Processing long-term memory costs LLM calls per chunk; so we batch process, combining memories by likeness scores. This produces better vibes because the LLM sees the full day's context when scoring importance `[agent-memory-survey]`.

**Compress, don't drop.** Lossy compression beats truncation at every layer `[compressive]`. Raw notes stay at full fidelity but are archived; summaries are lossy but readable; embeddings capture semantic shape; opaque encoding preserves approximate representations when even summaries don't fit. `[acc]` The opaque encoding is fragile by design — the LLM acts as an encryption key, and the patterns that emerge from compressed representations reflect the model's intrinsic geometry `[halluc-geo]` `[form-constants]`. Before swapping your main text processing LLM, restore from lossless backups or have your current agent export the data in plaintext.

**Capture everything, process later.** You don't know what you don't know, but knowing you might need something later is a known-unknown. Biological memory handles this by capturing broadly during waking hours and consolidating selectively during sleep `[hippocampal-replay]` `[sleep-forget]`. The vibeNcoder follows the same pattern: lossless capture during active use, selective consolidation during idle time.

### The IA principle: Fit everything, forget nothing.

**Patch, overlap, compress, quilt.** Break retrieved memories into fixed-size patches. Compress each one as much as we can get away with. Stitch them together at semantic seams; not arbitrary token boundaries. Smallest amount of passes possible. The whole day's relevant memories fit in working memory, not just the top five.

**Approximation beats omission.** `[map-territory]` A compressed patch that preserves the gist of a memory is more useful than a memory that was dropped entirely because it didn't fit. The reading LLM is robust to character-level corruption `[charperturb]`; it reconstructs meaning primarily from word form rather than context `[typoglycemia]`. IA exploits this: approximate everything rather than discarding anything.

**Zero-shot, zero cost.** No fine-tuning. No extra parameters. The compression is done at inference time by the reading LLM itself. Lower ratios than trained compressors, less stability, but zero setup cost and no additional model to maintain.

### Three layers of mind

1.  **Long-term memory** (vibeNcoder): The agent's knowledge with decay and consolidation mechanisms to prevent meme buildup; immutable metadata for tags, weights, and importance ranking.
2.  **Short-term memory** (vibeNcoder): Timestamped daily logs in a separate database. Keyword + recency search only. The AI's `/tmp/` directory — interpreted to long-term memes during batch processing, then archived.
3.  **Working memory** (IA): Assembled from retrieved summaries, compressed via quilting to fit the context window. Marked as recalled/approximate. `[cogworkspace]`

The assistant sees none of this directly; it receives assembled working memory from retrieved summaries, compressed by IA to fit. `[controlled-halluc]` Offloading recall frees the model for reasoning, following MemGPT's pattern of main context as working memory and external storage as long-term `[memgpt]`. CoALA's taxonomy `[coala]` covers working, episodic, and semantic memory; procedural memory is deferred. LLM internal representations show measurable alignment with human neural patterns during structured reasoning `[neuro-align]`, though this alignment tracks formal linguistic structure more than broader cognitive function `[outgrow]`.

Staged representations follow a general systems principle: don't force one artifact to satisfy every requirement. Each layer of the memery pipeline serves a specific purpose at a specific fidelity level. The self-similar hierarchy that emerges — the same consolidation pattern at daily, weekly, and lifetime scales — is not an engineering choice but an expected outcome of organizing information under bounded resources `[sanc]` `[causal-emerge]`. Fractal self-similarity appears at every level of neural organization `[fractal-neural]`; the memery system exhibits the same pattern because it faces the same constraints.

---

## Architecture

| Layer | Contents | Properties |
|-------|----------|------------|
| **Raw Source Archive** | Original `.md` files | Ground truth. Re-summarization source. Never the fast retrieval layer. |
| **Capture Buffer** | Append-only daily log in SQLite | Timestamped raw input. No LLM processing. Keyword + recency search only. Drained nightly. |
| **Interpreted Memery Store** | Single SQLite `.db` | Summaries, tags, entities, relationships, scores, embeddings, access metadata. FTS5, WAL mode. |
| **Semantic Embedding Layer** | Local model (~80MB, CPU) vectors as blobs | Fuzzy associative search. Brute-force cosine at <10K notes. |
| **Working Memery** | Temporary context per-prompt | Assembled from retrieved summaries. Marked as recalled/approximate. |
| **Semantic Hierarchy** | Content priority tiers | Kernel → Persona → Priority → Regular. |

### The Interpreted Store

| Component | Purpose |
|-----------|---------|
| **Summaries** | LLM's judgment of what mattered. Short, readable, intentionally lossy. |
| **Tags** | Filter labels, canonicalized at write time. |
| **Entities & relationships** | (entity, relation, entity) triples. |
| **Importance & confidence** | Ingest-time scores, decayed over time. |
| **Embeddings** | Vectors from local model. Fuzzy semantic search. |
| **Access metadata** | Timestamps, hit counts. Recency/frequency scoring. |

If embeddings become the bottleneck: count total overhead honestly, prefer online methods, evaluate with retrieval quality (recall@k) not reconstruction quality (L2 error). `[turboquant]`

---

## Retrieval

Every prompt. Zero LLM calls.

```
Conversation context
→ embed query locally (~5ms, CPU)
→ vector similarity + metadata filters in SQLite (~1ms)
→ top-k summaries + metadata
→ (optional) raw source chunk for detail
→ assemble working memory
```

---


### Scoring

Multiple independent signals; a miss on one is caught by another `[mnemosyne]` `[superlocal]`:

-   **Semantic similarity**: cosine distance (primary)
-   **Importance**: ingest-time durability judgment
-   **Recency**: `exp(-λ × days_since_access)` where `λ = ln(2) / half_life_days`
-   **Tag overlap**: bonus per matching tag
-   **Entity match**: bonus for known entity mentions
-   **Score Floor Gate**: reject if best match is weak
-   **Access frequency**: `log(1 + access_count) × weight`

Potential starting formula: `similarity × 0.6 + importance × 0.2 + recency × 0.2` `[gen-agents]`. A **score floor** blocks weak results even if they're technically top-k. `[main-rag]`

### Entity Linking

(entity, relation, entity) triples in a simple adjacency table. Entity matches boost retrieval score rather than creating a separate result path. HippoRAG's graph traversal (Personalized PageRank) is a potential upgrade if flat boosting proves insufficient `[hipporag]`. Entities are disambiguated at both ingest and retrieval time `[canonicnell]`.

---

## Ingestion

Two phases. Capture is instant and indiscriminate. Processing into long term memory is batch and selective.

---

### Phase 1: Capture (active use)

Zero LLM calls. Near-zero latency budget.

```
New information arrives
→ timestamp + source metadata
→ append to capture buffer (SQLite table, WAL mode)
→ raw .md preserved unchanged
```

Everything goes in. No importance threshold yet, no summarization, no embedding. The capture buffer is a simple log; the system's short-term 'sensory memory'. `[soc-walks]` `[stigmergy]` Same-day retrieval is keyword/recency only (see §Open Questions).

---

### Phase 2: Batch Processing (idle time)

The day's buffer, processed in one batch. Multiple LLM calls, but offline; no heavy processing while the handler is active.

Read full capture buffer for the period:
```
→ LLM triages the batch: importance scoring with full-day context [sure] [hippocampal-replay]
→ discard below importance floor
→ surviving entries: normalize / chunk
→ LLM extracts: summary, tags, entities, relationships, importance
→ canonicalize at write time (Unicode normalization, accent stripping, lowercasing, etc)
→ embed summaries locally (~5ms each)
→ store in interpreted memery store (SQLite)
→ mark buffer entries as processed; archive any fully processed entries.
```

Batch context is the key advantage: seeing "had coffee," "started new job," and "named the cat" in the same pass produces better relative importance scores than evaluating each in isolation `[agent-memory-survey]`. Surprise-based prioritization outperforms uniform processing `[sure]`.

Batch processing also exploits **prompt caching**. Every call shares the same prefix; system prompt, importance rubric, few-shot examples, existing memory context; only the buffer entry changes. The provider caches the shared prefix after the first call (Anthropic: ~90% discount on cached tokens, Google: ~75%). Structure batch calls to maximize prefix length and minimize per-entry variation `[dontbreak]` `[sglang]`.

### Model Cascading

The batch pipeline defaults to a small model; a lightweight quality proxy scores each output and routes only hard cases to a larger model; QE-based deferral applied to memory processing `[cascade-qe]`.

**Budget-constrained setup (batch mode):** Run all entries through the small model first. Rank outputs by proxy quality score. Defer the lowest-scoring η fraction to the large model. The computational parity threshold is:

  η* = 1 − (N_S + N_proxy) / N_L

where N_S = small model parameters, N_proxy = proxy model parameters, N_L = large model parameters. For a 10× size gap with a negligible proxy, η* ≈ 0.9; cascading beats always-large as long as fewer than 90% of entries get escalated.

**Proxy quality signal (no trained QE model exists for this task):**
-   Small model self-scored confidence on its own output — an explicit 0–1 rating in the same call
-   Heuristic flags: entry contradicts existing memory, entity disambiguation failed, importance score near a decision boundary (e.g., 0.45–0.55)
-   Logprobs as a fallback — weak signal until calibrated; the source paper found logprob-based deferral performs near-random for MT

**Dynamic thresholding (streaming/sequential mode):** When the full batch isn't available; same-day buffer drain, interrupted consolidation; bootstrap a threshold from the first B entries' score distribution, then update it after each subsequent block of B to approximate the target deferral rate. No batch required; quality matches the budget-constrained setup (see §Open Questions: Same-day retrieval gap).

### Importance

A 0–1 score. Rubric:

| Range | Meaning | Example |
|-------|---------|---------|
| 0.0–0.2 | Filler | "Had coffee this morning." |
| 0.3–0.4 | Ephemeral | "Tried a new restaurant." |
| 0.5–0.6 | Worth noting | "Decided on EPDM liner." |
| 0.7–0.8 | Clearly important | "Started a new job at Acme." |
| 0.9–1.0 | Foundational | "Has a child named Sam." |

The agent receives the rubric plus few-shot examples for calibration. Tags are flat, free-form, selected by the AI at write time. The consolidation pass builds a de facto vocabulary by merging rare tags into common ones.

In batch mode, the LLM scores importance with the full day's captures visible, enabling relative rather than absolute calibration. This mirrors biological replay, where consolidation is biased toward high-reward, high-surprise events `[hippocampal-replay]` `[sure]`.

---

## Semantic Hierarchy

Content priority tiers inspired by human psychological hierarchies, applied to semantic content `[soar]` `[demandpaging]`:

| Layer | Contents | Behavior |
|-------|----------|----------|
| **Soul** | Identity constraints, immutable rules | Resident. Highest fidelity. Rarely changes. |
| **Heart/Mind** | Interaction register, subtext feedback | Tracks *how* to engage, not *what* to recall. Shaped by open questions (see §Open Question Resolution). Can promote to Kernel. |
| **Attention** | Active context for current work | Variable weight multipliers at retrieval. Dynamic membership. |

Memories move between tiers via ACT-R's base-level activation: availability tracks power-law relationships to frequency and recency of use `[act-r]`. TiMem `[timem]` and Oblivion `[oblivion]` validate this; progressive consolidation upward, decay-driven demotion downward.

Implemented as soft tiering via existing `importance`, `access_count`, and `last_accessed` fields.

---

## Memery Metabolism

Information moves through compression stages, dormancy, and eventual pruning. Flow paths self-optimize through use `[constructal]` `[fractal-brain]`.

| Level | State | Representation | Access |
|-------|-------|---------------|--------|
| **Level 0 — Id** | Active, high-fidelity | Full summaries, entities, tags, embeddings | Normal retrieval |
| **Level 1 — Ego** | Low access, decayed recency | Crushed summaries, opaque encodings, or character-reduced text | Only when explicitly relevant |
| **Level 2 — Superego** | Merged into master entries | Observations within a master topic; originals marked `consolidated` | Via master entry |
| **Level 3 — Archive** | Maximum compression | Enough to know it existed and what topic it covered | Audit only |
| **Level 99 — Meme death** | Beyond maximum compression, noise is discarded. |

**Paging:** Importance is the primary signal. High-importance memories resist compression; low-importance memories decay faster. Access frequency is secondary; otherwise a low-importance memory that keeps getting retrieved resists demotion `[act-r]` `[oblivion]`.

**Cache-hit signal:** During active use, the provider tracks how often each prefix segment is served from cache `[promptcache]`. A memory that appears in many cached prefixes throughout the day is *de facto* load-bearing context; the system kept reaching for it. Feeding daily cache-hit counts into the decay function gives a free, passive importance signal that requires no LLM calls and no explicit access logging: high cache hits resist demotion, low cache hits accelerate it.

**Deletion epoch:** An entropy floor below which further compression is pointless; the memory no longer justifies storage `[nepenthe]`.

**Expansion-testing:** Before pruning a dormant memory, expand it and measure whether it adds value to recent queries. If it scores above the retrieval floor, keep it. Analogous to Prioritized Experience Replay `[per]`.

**Literal protection:** Fragile datapoints (API keys, phone numbers, exact quotes, URLs, precise figures, passwords) are stored separately from the compressible summary and pass through every decay stage untouched. Character reduction and opaque encoding operate on prose only; literals are pulled fresh at hydration time.

### Lossy Text Compression

An intermediate stage between full-fidelity and deletion. LLMs are robust to character-level corruption `[charperturb]`; they reconstruct meaning primarily from word form (preserved first/last letters, word length) rather than context `[typoglycemia]`, with measured degradation curves at increasing severity `[textrobust]`.

For dormant memories: remove middle characters from words over a length threshold. ~20–25% size reduction, fully parseable by the reading LLM, embeddings unaffected (generated at ingest from full text). This is a **novel application**; the cited papers measure robustness as a property; none propose exploiting it as a compression technique.

---

## Nightly Consolidation

The system's sleep cycle `[sleep-forget]` `[ai-meets-brain]` `[fractal-hippo]` `[criticality-memory]` `[prigogine]`. Three phases: drain the capture buffer into the interpreted store; merge scattered memories into master topic entries; resolve open questions.

### Phase 1: Buffer Drain

Process the day's capture buffer via the batch processing pipeline (see §Ingestion, Phase 2). Raw captures become first-class memories; scored, summarized, embedded, and stored. Entries below the importance floor are discarded (or archived at Level 0 if the raw source note exists). The capture buffer is cleared after processing.

If consolidation is interrupted, unprocessed entries persist and are picked up on the next run. The buffer is the write-ahead log; the interpreted store is the checkpoint.

### Phase 2: Topic Merging

Scattered memories about the same topic roll up into **master topic entries** with ranked observations; a single-level RAPTOR `[raptor]`, independently validated as "reflection" in Generative Agents `[gen-agents]`. Biological replay during sleep is selective and importance-weighted, not random `[hippocampal-replay]`; the merging pass follows the same principle `[turing-topics]`.

Each observation tracks an **update count**. High count = durable. Low count + old = stale.

-   **Grouping:** entity overlap first, tag overlap as tiebreaker
-   **Schema:** `parent_id` FK on memories table — master entries are rows with `parent_id = NULL`
-   **Originals:** kept, marked `consolidated = true`, deprioritized
-   **Contradictions:** replaced with new observation, update count incremented, reversal noted
-   **Cap:** ~10–15 observations per topic; beyond that, split into subtopics
-   **Threshold-based demotion:** During the merging pass, every memory is re-weighed against the current importance landscape. Memories below the primary-topic threshold lose standalone status and fold into the nearest master entry (`parent_id` set, `consolidated = true`). Sub-observations above the threshold are promoted to master entries (`parent_id` cleared). The same threshold that gates ingest (see §Importance) gates topic status; one rubric, applied twice.
-   **Relationship and summary refresh:** The consolidation LLM re-evaluates entity relationships and summaries against accumulated evidence. Stale assertions update in place (e.g., `(Ron, dating, Rhonda)` → `(Ron, married_to, Rhonda)`), update count incremented, reversal noted. Same batch context, no extra traversal `[hippocampal-replay]`.

### Phase 3: Open Question + Resolutions

The system maintains a queue of **open questions**; things it wants to learn about the handler but should infer from behavior rather than ask directly. Examples: *Does my handler prefer the Oxford comma? Do they write with emdashes or semicolons? Do they want a casual register? What frustrates them? etc.*

Open questions are checked against the day's captures during consolidation. The LLM sees the full day's raw text and accumulated prior evidence, then updates confidence on each question. Resolution is gradual; a single day's usage might shift confidence from 0.3 to 0.5; it takes consistent signal across multiple consolidation cycles to resolve a question. This mirrors how humans form impressions: not from one data point, but from patterns `[hippocampal-replay]`.

-   **Source:** Questions can be seeded manually (system operator, user request) or generated by the consolidation LLM when it notices ambiguity in existing Persona-tier memories.
-   **Evidence, not interrogation:** The system never surfaces open questions as direct prompts to the handler. Answers are derived from observed behavior; word choice, punctuation patterns, reaction patterns, correction patterns.
-   **Resolution:** When confidence crosses a threshold, the question produces a Persona-tier memory (or promotes to Kernel if the finding is sufficiently durable). The question is retired; the evidence trail is preserved in the raw source archive.
-   **Formative questions:** Not all questions are meant to resolve. Some are **scaffolding**; persistent attentional priors that keep the agent watching for social and communicative signals it hasn't internalized yet. *What makes my handler frustrated? When do they want brevity vs. detail? How do they signal they're done with a topic?* These questions linger by design, accumulating partial evidence that shapes the Persona tier without ever collapsing to a single answered fact. They are how the agent learns the social contract with its specific handler; not through explicit instruction, but through sustained observation. `[social-sim]` Formative questions are exempt from expiry; they persist until answered, however long that takes.
-   **Perennial questions:** Some questions are too large; *and too alive*, to ever resolve. *Who is important to this person? What are their current priorities? What do they care about most right now?* Unlike formative questions, which shape attentional priors passively (how to communicate), perennial questions produce **concrete, updateable entity and relationship data** that feeds directly into the entity and relationship tables (who matters, what matters). Their answers are living snapshots; always valid *as of now*, never final. `[sys-theory]` They are exempt from both expiry and resolution; they never retire because the answer is always in flux.
    -   **Snapshot evolution:** When a perennial answer changes; a relationship ends, a new project becomes central, someone enters or leaves the handler's life; the old answer is marked with a temporal boundary rather than overwritten as wrong. "Alex was important in 2025" is not a correction of "Alex is important"; it's a dated transition. This extends the relationship refresh mechanism (see §Topic Merging): `(Ron, dating, Rhonda)` → `(Ron, married_to, Rhonda)` is an update to a continuing relationship, but "Alex faded from the handler's life" is a different kind of transition; presence dimming rather than a fact changing. The reversal-noting mechanism captures both, but the system distinguishes "life changed" from "old data was incorrect."
    -   **Hybrid inquiry:** Perennial questions follow the "evidence, not interrogation" default; the system observes and infers. However, individuals can **opt in** to being asked about specific perennial domains (e.g., "you can ask me about my relationships," "you can ask about my work priorities"). Opt-in permissions are stored as Kernel-tier memories. Even with permission, questions are only surfaced when contextually natural; the system waits for a moment where asking would feel like genuine interest, not data collection. A user mentioning someone new three times in a week is an opening; a cold "tell me about your family" is not.
-   **Expiry:** Applies only to regular questions; neither formative nor perennial. Questions that accumulate no evidence after a configurable number of consolidation cycles are downgraded; the handler's behavior may simply not contain signal for that question.

---

## Infinite Approximation

**IA is a retrieval-time, working-memory tool only.** It does not write to the interpreted store. It is not a long-term compression strategy. Its job is minimal summation; just enough to break retrieved context into patches to fit more of it in the working window.

Trade processing time for effective context size. LLMs can self-compress text into opaque token sequences; Gist Tokens at 26x `[gist]`, ICAE at ~4x `[icae]`, 500xCompressor at 6x–480x `[500x]`. Infinite Approximation exploits this **zero-shot** — no fine-tuning, no extra parameters. Lower ratios, less stability, but zero setup cost.

### The Quilting Mechanism

Borrowed from image quilting's patch-overlap-seam structure `[quilting]`:

**Patches**: fixed-size token chunks, sized to fit in the context window alongside accumulated compressed blobs and scaffolding.

**Overlap zones**: each patch overlaps the previous by a configurable amount. Without overlap, boundary cuts corrupt meaning `[longformer]` `[unlimiformer]`.

**Seam selection**: within each overlap, find a natural boundary (sentence end, paragraph break, topic shift) rather than cutting at an arbitrary token count. SaT `[sat]` identifies candidates without LLM inference.

**The quilting pass:**

```
Read Patch 1 → compress into opaque encoding
Read Patch 2 (starts at patch_size minus overlap)
→ find seam in overlap zone
→ stitch: Patch 1's blob up to seam, Patch 2 owns the rest
→ compress Patch 2 from seam forward, add to chain
→ repeat until done
```

Result: a chain of compressed blobs joined at semantic seams. Single pass. Validated independently by AutoCompressors (30,720 tokens) `[autocomp]`, CompLLM (compressed context **outperforms** uncompressed on long sequences via noise removal) `[compllm]`, and HOMER (training-free divide-and-conquer with token reduction) `[homer]`.

### Scaling

Tokens processed ≈ input × (1 + overlap ratio). Linear time cost; overlap is a constant multiplier.

| Knob | Smaller | Larger |
|------|---------|--------|
| **Patch size** | More patches, finer granularity | Fewer patches, less overhead |
| **Overlap width** | Less overlap; faster, fewer seam options | More overlap; slower, better boundary safety |
| **Seam effort** | Fast heuristic | Model evaluates candidates |

If a simple method is close to optimal, extra machinery needs strong justification `[turboquant]`.

### KV Cache Spillover

**Default: re-derive.** Compressed blobs are short (64–256 tokens). Running a forward pass to regenerate KV state is 5–10× faster than PCIe transfer for typical blob sizes on models under ~30B. No RAM buffer, no eviction policy, no bookkeeping. When a patch's KV entries are needed and not in VRAM, just re-run it.

**Fallback: RAM spill.** For long blobs or large models where re-derivation becomes expensive, an opt-in flag (`kv_spillover`) reserves ~6GB of CPU RAM as overflow. When VRAM fills, older patches' KV state spills to this buffer. Same PCIe pathway as before; the GPU can overlap attention with the transfer.

**`kv_agent_swap` mode:** Flushes the *entire* current KV state to the RAM buffer on demand — vacating VRAM fully for a subagent. Restore is atomic on main agent resume. Unlike vLLM sleep mode `[vllm-sleep]`, which discards the KV cache on model hibernation, this preserves it; the main agent resumes with full context intact. Supports agentic task decomposition (see §Agentic Task Decomposition) `[demandpaging]` `[neo-offload]`.

---

## Compression Modules

The quilting architecture treats compression as swappable. The pipeline doesn't care *how* each patch gets compressed.

### The Spectrum

| Method | Ratio | Training | Fidelity |
|--------|-------|----------|----------|
| **Zero-shot opaque** | Variable | None | Approximate, model-dependent. No codebook. |
| **ICAE** `[icae]` | ~4x | LoRA (~1% params) | High. Drop-in upgrade. |
| **Gist Tokens** `[gist]` | Up to 26x | Fine-tuned tokens | High. |
| **500xCompressor** `[500x]` | 6x–480x | ~0.3% params | 62–73% at extremes. |
| **Z-tokens** `[ztokens]` | Up to 18x | Trained | Content-adaptive. |
| **Token pruning** `[llmlingua]` | Up to 20x | None | Full fidelity on retained tokens. |
| **Character reduction** | ~20–25% | None | High. See §Lossy Text Compression. |
| **Stable map** | Extreme | None | Perfect. Deterministic string substitution. |

### The Stable Map

Trigger strings that expand to full text via application-layer substitution; deterministic, cacheable, model-independent. The model never sees the map. **Syntactic** compression (known recurring strings, losslessly) alongside **semantic** compression (novel content, approximately).

Applications beyond scaffolding:

-   **Output composition**: agent writes trigger references; application layer expands after the LLM call
-   **Domain vocabularies**: topic-specific libraries of reusable chunks composed by reference; "memes" in the original sense.
-   **Iterative chain processing**: large content serialized as trigger chains, expanded one at a time; working set never exceeds one expansion plus accumulated state

### Compression-Fidelity Table

| Layer | Compression | Reader | Fidelity |
|-------|------------|--------|----------|
| Raw notes | None | Human + model | Perfect |
| LLM summaries | ~20–40x | Human + model | Lossy, readable |
| Embeddings | Extreme | Math only | Semantic shape |
| Opaque encoding | Extreme | Model only | Approximate |
| Stable map | Extreme | Application layer | Perfect |
| Character-reduced | ~20–25% | Model only | High |

The real metric: **effective tokens of understanding per token of context spent**, including scaffolding. The real test: task completion, not reconstruction fidelity. `[turboquant]`

---

## Integration Points

IA lives in the **working memery layer**. Connections to the memery system:

| Connection | How |
|------------|-----|
| **Ingest** | Capture phase: raw append, no IA. Batch phase: long notes processed through IA before summarization; richer input than truncation. |
| **Context assembly** | Retrieval overflow compressed via IA instead of dropped. All 12 memories instead of top 5. |
| **Conversation** | IA replaces "summarize and forget"; higher fidelity than summaries. |
| **Hierarchy-aware** | Kernel content resists compression; Regular content compressed aggressively. |
| **Chain reading** | Agent walks compressed chains left-to-right. Map entries: instant. Opaque blobs: one LLM call per patch. |
| **Agentic task decomposition** | Large input quilted to establish full scope → decomposed into shards → subagents execute in GPU slot vacated by hibernated main agent → results via capture buffer. See §Agentic Task Decomposition. |

---

## Agentic Task Decomposition

Large agentic tasks; a codebase refactor, a long specification, a multi-file edit chain; are too large for a single context but too interconnected to split blindly. The quilting pass handles the first problem (parse the whole thing at once); seam boundaries handle the second (split at meaning, not at token count). This gives a task decomposition primitive built entirely on existing infrastructure.

**Shard sizing principle:** Decompose into the *largest* independently processable chunks; not the smallest. More and smaller shards means more handoffs, more context loss, more coordination overhead. The goal is maximum shard size subject to independence. A shard that is itself still too large for one context is handled by the subagent running its own quilting pass internally; recursion is free because the infrastructure already exists.

```
Quilt the full input → compressed chain (full scope in working memory)
↓
Decompose: one LLM call over the chain → N task shards
each shard: subtask description + relevant patch indices + dependency flags
sizing: largest independent unit, not smallest
↓
Hibernate: kv_agent_swap flushes main agent KV to CPU RAM
GPU fully vacated
↓
Execute shards in dependency order:
subagent loads only its relevant patches (not full chain)
if shard is large: subagent runs its own IA pass internally
works → writes results to capture buffer
(dependent shards wait for upstream results before loading)
↓
Restore: main agent KV reloaded from RAM
retrieval pulls shard results from capture buffer
main agent synthesizes
```

**Why each step is cheap:**
-   Quilting is one pass; already done if the input was large
-   Decomposition is one LLM call over the *compressed* chain, not raw input
-   Subagents see only their shard's patches; large shards quilt themselves rather than receiving a pre-compressed blob
-   Hibernation is a PCIe flush; fast, no disk, no new hardware
-   Shard outputs use the existing capture buffer; no new output channel

**Dependency handling:** The decomposition step flags which shards are independent and which depend on upstream results. Independent shards run in any order; dependent shards wait for their inputs. Failing to track this is the primary documented failure mode in multi-agent systems `[sagallm]` `[why-mas-fail]`. The dependency structure is produced in the same call that generates the shard list; no extra pass needed.

**Interaction with model cascading:** The decomposer tiers each shard at assignment time; routine shards get the small model, complex shards get the large model (§Model Cascading). The main agent never touches routine work directly.

**Interaction with the memory system:** Shard outputs written to the capture buffer are first-class captures. They go through nightly consolidation like anything else. If the task surfaces reusable knowledge; a discovered pattern, a design decision; it persists into long-term memory without extra work.

---

## Design Constraints

1.  **Never delete raw source notes.** Interpreted store is derived and rebuildable.
2.  **Retrieval is automatic and free.** Zero LLM inference.
3.  **Capture is indiscriminate. Processing is selective.** Everything enters the buffer; memories earn their way into the interpreted store.
4.  **No LLM calls during active use.** Capture and retrieval are both LLM-free. Processing happens offline. `[sleep-forget]`
5.  **Assistant receives assembled context, not raw rows.**
6.  **Summaries are lossy by design.** Literals (keys, numbers, exact quotes, URLs) are stored separately from compressible info and bypass every decay stage.
7.  **Omit rather than invent.** Missing detail beats hallucinated detail.
8.  **One file for the store.** SQLite. No external services.
9.  **Score floor enforced.** No match above threshold → return nothing. `[main-rag]`
10. **Canonicalize at write time.** `[canonicnell]`
11. **Optimize total cost per memery.** Count all bytes. `[turboquant]`
12. **Prefer online, local-ingest-cost methods.** `[turboquant]`
13. **Benchmark the whole pipeline.** Final answer quality is the scorecard. `[turboquant]`
14. **Stop adding machinery.** Simple method near the limit? Extra complexity needs strong justification. `[turboquant]`

---

## Open Questions
Places where our design theory is genuinely open and empirical data is needed.

-   **How far does zero-shot go?** At what point does a trained compression module become necessary? Probably varies by model and content type.
-   **Content-adaptive ratios.** Z-tokens `[ztokens]` and 500xCompressor `[500x]` suggest uniform compression is suboptimal; dense passages need more tokens, filler needs fewer. But variable-rate encoding has real complexity cost.
-   **Metabolism thresholds.** Can promotion/demotion learn from access patterns, or does it need manual tuning? ACT-R `[act-r]` says the signal exists; the question is whether it's clean enough at personal scale.
-   **Map vocabulary ceiling.** At some point the menu costs more than the meal. Where?
-   **Beyond text.** The patch-overlap-seam abstraction doesn't require prose. CacheGen `[cachegen]` and ChunkKV `[chunkkv]` suggest the same principles apply at the KV-cache level.
-   **Scaling threshold.** When does brute-force cosine yield to ANN? At personal scale, maybe never.
-   **Compression as denoising.** CompLLM found 2x compression **outperforms** uncompressed on long sequences `[compllm]`. Under what conditions does compression become a net quality improvement?
-   **StreamingLLM complementarity.** StreamingLLM `[streamingllm]` preserves recency perfectly, drops old context entirely. IA preserves everything approximately. Could attention sink insights stabilize patch compression?
-   **Same-day retrieval gap.** Captured data isn't in the interpreted store until the nightly drain. Keyword + recency search on the raw buffer is a stopgap — good enough for "what did I say about X today?" but no semantic similarity, no importance weighting. Is a lightweight same-day embedding pass (local model, no LLM) worth the capture-time cost? Or does FTS5 on the capture buffer suffice?
-   **Buffer size at scale.** A heavy day of capture might produce thousands of entries. Does batch processing need chunked passes (process N entries at a time with context from already-processed entries), or can a single LLM call triage the full day? Context window size is the binding constraint.
-   **Replay ordering.** Biological consolidation replays experiences in importance-weighted order, not chronological `[hippocampal-replay]`. Should batch processing sort by estimated surprise before detailed scoring? SuRe's surprise-based prioritization suggests yes `[sure]`.
-   **Proxy QE for memory processing.** QE-based cascading requires a reliable quality signal; no trained QE model exists for general memory entry processing. Does small-model self-scored confidence calibrate well enough to gate escalation, or does it need held-out validation? What fraction of entries actually need the large model (i.e., what is η in practice)?
-   **Subagent IA depth.** When a shard is large enough to require its own quilting pass, the subagent runs IA internally. How far does this recurse in practice? Is a two-level hierarchy (main agent quilts whole input, sub-agent quilts its shard) sufficient for real coding tasks, or do some shards require deeper nesting?

---

## Event Taxonomy

The capture buffer needs structured event categories so batch processing can reason about *what kind* of thing was captured, not just *that* something was captured. This matters for importance scoring (a tool call error is different from a file edit is different from a git commit) and for downstream consolidation (grouping by event type before cross-type merging).

**Approach:** Adopt a tiered taxonomy — lifecycle events (sessions starting/ending), cognition events (thoughts, decisions, goals), and operation events (tool calls, file changes, commands). The exact tiers and categories are TBD; we need to define them based on what actually flows through the system once recording is live.

**Back-sorting is safe.** Raw capture is append-only with full metadata (timestamp, source, content). Event type is a derived label, not a structural dependency. We can capture everything now with a minimal or null event_type field, then retroactively categorize entries once the taxonomy is finalized. This means the recording layer (Stage 0) is not blocked by taxonomy design — they can be done in parallel. The taxonomy just needs to land before batch processing runs, since the triage prompt will use event types as context for importance scoring.

**Constraint:** Do not adopt an external taxonomy wholesale. Study existing approaches (e.g. AOP's lifecycle/cognition/operation tiers) for structural inspiration, but define categories that fit memery's actual data flow rather than optimizing for a standard we don't need to interoperate with yet.

---

## References

| Key | Source | Notion |
|-----|--------|--------|
| `[engram]` | [Conditional Memory via Scalable Lookup: A New Axis of Sparsity for Large Language Models](https://arxiv.org/abs/2601.07372) | Adds a "conditional memory" module to Transformer LLMs, with O(1) lookup of static patterns plus canonicalization and gating mechanisms that directly inform retrieval scoring and write-time normalization.
 |
| `[turboquant]` | [TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate](https://arxiv.org/abs/2504.19874) | Supplies the cross-cutting design discipline here: count total overhead, use staged representations, optimize task outcomes instead of reconstruction fidelity, and avoid extra machinery unless it buys real value. 
|
| `[gen-agents]` | [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) | Origin of the recency × importance × relevance retrieval pattern, and an independent precedent for periodic "reflection" or consolidation from raw memories into higher-level observations. 
|
| `[memgpt]` | [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) | Validates the main-context-versus-external-store split by treating working memory like an OS paging layer over longer-term storage. |
| `[hipporag]` | [HippoRAG: Neurobiologically Inspired Long-Term Memery for Large Language Models](https://arxiv.org/abs/2405.14831) | Supports the entity-triple layer by showing graph traversal can surface connections that flat vector similarity alone may miss. 
|
| `[raptor]` | [RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval](https://arxiv.org/abs/2401.18059) | Precedent for nightly consolidation as a summarization-and-merge pass over related memories, even though this design uses a shallower version. 
|
| `[coala]` | [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427) | Provides the memory taxonomy vocabulary: working, episodic, semantic, and procedural, with memery explicitly covering the first three. 
|
| `[soar]` | [Introduction to Soar](https://arxiv.org/abs/2205.03854) | Gives architectural precedent for distinct memory tiers with different access and learning behavior, which motivates the semantic hierarchy. 
|
| `[act-r]` | [Reflections of the Environment in Memory](https://journals.sagepub.com/doi/abs/10.1111/j.1467-9280.1991.tb00174.x) | Theoretical basis for metabolism and tier movement: memory availability follows a power-law relationship to frequency and recency of use. 
|
| `[timem]` | [TiMem: Temporal-Hierarchical Memory Consolidation for Long-Horizon Conversational Agents](https://arxiv.org/abs/2601.02845) | Validates hierarchical promotion by showing raw observations can consolidate over time into more abstract memory structures. 
|
| `[oblivion]` | [Oblivion: Self-Adaptive Agentic Memory Control through Decay-Driven Activation](https://arxiv.org/abs/2604.00131) | Supports decay-driven promotion and demotion, where forgetting is reduced accessibility rather than literal deletion. 
|
| `[demandpaging]` | [The Missing Memory Hierarchy: Demand Paging for LLM Context Windows](https://arxiv.org/abs/2603.09023) | Strongest hardware-style analogy for the semantic hierarchy: working-set pinning, stale-context eviction, and page-fault-like reactivation. 
|
| `[per]` | [Prioritized Experience Replay](https://arxiv.org/abs/1511.05952) | Basis for expansion-testing and value-weighted retention: revisit dormant data based on potential usefulness rather than uniformly. 
|
| `[nepenthe]` | [NEPENTHE: Entropy-Based Pruning as a Neural Network Depth's Reducer](https://arxiv.org/abs/2404.16890) | Grounds the deletion epoch idea in an entropy floor, where further retention or compression no longer carries meaningful information. 
|
| `[quilting]` | [Image Quilting for Texture Synthesis and Transfer](https://merl.com/publications/docs/TR2001-17.pdf) | Conceptual source for overlapping patches joined at optimal seams, translated from image synthesis into text-patch stitching. 
|
| `[compressive]` | [Compressive Transformers for Long-Range Sequence Modelling](https://arxiv.org/abs/1911.05507) | Core validation for "compress, don’t drop": lossy compression of old context is better than discarding it outright. 
|
| `[icae]` | [In-Context Autoencoder for Context Compression in a Large Language Model](https://arxiv.org/abs/2307.06945) | Shows that LLM-family models can encode and decode compressed latent slots, giving a fine-tuned ceiling for the zero-shot opaque encoding idea. 
|
| `[gist]` | [Learning to Compress Prompts with Gist Tokens](https://arxiv.org/abs/2304.08467) | Evidence that prompt information can be densely packed into a small number of special tokens without catastrophic quality loss. 
|
| `[500x]` | [500xCompressor: Generalized Prompt Compression for Large Language Models](https://arxiv.org/abs/2408.03094) | Important for the claim that compression should be content-adaptive, since some compressed tokens matter much more than others. 
|
| `[ztokens]` | [Large Language Model as Token Compressor and Decompressor](https://arxiv.org/abs/2603.25340) | Direct support for variable-rate compression, where dense passages deserve more latent capacity than redundant ones. 
|
| `[autocomp]` | [Adapting Language Models to Compress Contexts (AutoCompressors)](https://github.com/princeton-nlp/AutoCompressors) | Architecturally close to the quilting pipeline because it processes long inputs segment by segment with compressed carry-forward. 
|
| `[unlimiformer]` | [Unlimiformer: Long-Range Transformers with Unlimited Length Input](https://openreview.net/pdf?id=lJWUJWLCJo) | Direct evidence that chunk edges are unreliable and need overlap coverage from neighbors, which supports the seam-and-overlap design. 
|
| `[longformer]` | [Longformer: The Long-Document Transformer](https://arxiv.org/abs/2004.05150) | Supports the idea that local windows alone are insufficient and need cross-window information flow, analogous to overlap zones here. 
|
| `[sat]` | [Segment Any Text: A Universal Approach for Robust, Efficient and Adaptable Sentence Segmentation](https://arxiv.org/abs/2406.16678) | Practical seam-detection option for lightweight sentence-boundary identification during patch construction. 
|
| `[homer]` | [Hierarchical Context Merging: Better Long Context Understanding for Pre-trained LLMs](https://arxiv.org/abs/2404.10308) | Closest prior system to the broader quilting goal: divide long input, process progressively, and merge chunks with reduction. 
|
| `[streamingllm]` | [Efficient Streaming Language Models with Attention Sinks](https://arxiv.org/abs/2309.17453) | Useful complement and contrast: perfect recency preservation without compression, but at the cost of dropping older context entirely. 
|
| `[llmlingua]` | [LLMLingua: Compressing Prompts for Accelerated Inference of Large Language Models](https://arxiv.org/abs/2310.05736) | Zero-shot prompt compression by pruning low-information tokens, included as an alternative compression philosophy to dense opaque encoding. 
|
| `[lumberchunker]` | [LumberChunker: Long-Form Narrative Document Segmentation](https://arxiv.org/abs/2406.17526) | Alternative seam-detection strategy that finds semantic content shifts instead of relying only on sentence boundaries.
 |
| `[compllm]` | [CompLLM: Compression for Long Context Q&A](https://arxiv.org/abs/2509.19228) | Important evidence that compression can improve outcomes, not just reduce cost, because removing noise can increase signal-to-noise ratio. 
|
| `[cachegen]` | [CacheGen: KV Cache Compression and Streaming for Fast Large Language Model Serving](https://arxiv.org/abs/2310.07240) | Suggests an activation-space implementation path: compressed KV chunks as dense encodings instead of text-token-level compression. 
|
| `[chunkkv]` | [ChunkKV: Semantic-Preserving KV Cache Compression for Efficient Long-Context LLM Inference](https://arxiv.org/abs/2502.00299) | Supports chunk-level compression as a better semantic unit than isolated tokens, which matches the patch-based design. 
|
| `[acon]` | [ACON: Optimizing Context Compression for Long-horizon LLM Agents](https://arxiv.org/abs/2510.00615) | Potential method for automatically refining compression prompts through failure-driven optimization without gradients.
|
| `[charperturb]` | [Understanding the Ability of LLMs to Handle Character-Level Perturbation](https://arxiv.org/abs/2510.14365) | Evidence that LLMs can still recover meaning from heavily garbled text, which makes character-deletion compression plausible.
 |
| `[typoglycemia]` | [Word Form Matters: LLMs' Semantic Reconstruction under Typoglycemia](https://arxiv.org/abs/2503.01714) | Explains which characters are safest to remove: LLMs lean on word shape, especially first and last letters, more than full spelling. 
|
| `[textrobust]` | [Robustness of Large Language Models to Perturbations in Text](https://arxiv.org/abs/2407.08989) | Provides degradation curves for semantic-preserving corruption, which helps calibrate how aggressive lossy text compression can be. 
|
| `[sleep-forget]` | [Learning to Forget: Sleep-Inspired Memory Consolidation for Resolving Proactive Interference in Large Language Models](https://arxiv.org/abs/2603.14517) | Validates the sleep-cycle framing for consolidation by showing a learned sleep phase can clear stale interfering state. 
|
| `[agent-memory-survey]` | [Memory in the Age of AI Agents: A Survey — Forms, Functions and Dynamics](https://arxiv.org/abs/2512.13564) | Landscape survey that frames agent memory as a dynamic system, which informs the full-day batch-context view of importance scoring. 
|
| `[sure]` | [SuRe: Surprise-Driven Prioritised Replay for Continual LLM Learning](https://arxiv.org/abs/2511.22367) | Strong precedent for surprise-based prioritization, reused here for deciding which memories deserve deeper processing. 
|
| `[hippocampal-replay]` | [Post-learning replay of hippocampal-striatal activity is biased by reward-prediction signals](https://www.nature.com/articles/s41467-025-65354-2) | Neuroscience basis for importance-weighted replay: consolidation after a task is biased by reward-prediction signals rather than uniform review. 
|
| `[ai-meets-brain]` | [AI Meets Brain: A Unified Survey on Memory Systems from Cognitive Neuroscience to Autonomous Agents](https://arxiv.org/abs/2512.23343) | Bridges brain-memory models and agent-memory systems, supporting the analogy between nightly processing and biological consolidation. 
|
| `[promptcache]` | [Prompt Cache: Modular Attention Reuse for Low-Latency Inference](https://arxiv.org/abs/2311.04934) | Basis for treating cache hits as a passive importance signal, since repeated prompt segments can reuse stored attention state.
 |
| `[sglang]` | [SGLang: Efficient Execution of Structured Language Model Programs](https://arxiv.org/abs/2312.07104) | Supports batch prompt-cache exploitation through shared-prefix reuse across many similar calls.
 |
| `[dontbreak]` | [Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks](https://arxiv.org/abs/2601.06007) | Informs how to structure batch prompts so only per-entry content changes and cacheable prefixes stay stable. 
|
| `[cascade-qe]` | [Translate Smart, not Hard: Cascaded Translation Systems with Quality-Aware Deferral](https://arxiv.org/abs/2502.12701) | Direct precedent for the model-cascading setup: route all items through a small model first, then escalate only low-quality cases.
 |
| `[sagallm]` | [SagaLLM: Context Management, Validation, and Transaction Guarantees for Multi-Agent LLM Planning](https://arxiv.org/abs/2503.11951) | Supports explicit dependency handling in multi-agent decomposition by focusing on coordination failures, context loss, and invariant-preserving state transitions.
 |
| `[vllm-sleep]` | [Zero-Reload Model Switching with vLLM Sleep Mode](https://blog.vllm.ai/2025/10/26/sleep-mode.html) | Baseline for agent hibernation and offloading: fast model switching by sleeping weights, but without preserving KV cache.
 |
| `[neo-offload]` | [NEO: Saving GPU Memory Crisis with CPU Offloading for Online LLM Inference](https://arxiv.org/abs/2411.01142) | Validates CPU-RAM overflow and pipelined KV transfer as practical long-context inference techniques.
 |
| `[why-mas-fail]` | [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657) | Documents dependency-tracking failure as a primary multi-agent failure mode, motivating explicit dependency flags in shard decomposition. 
|
| `[mnemosyne]` | [Mnemosyne: An Unsupervised, Human-Inspired Long-Term Memory Architecture for Edge-Based LLMs](https://arxiv.org/abs/2510.08601) | Independent implementation of multi-signal retrieval scoring with temporal decay, hybrid connectivity/frequency scoring, and human-inspired memory consolidation — directly validating the recency × importance × relevance pattern. |
| `[main-rag]` | [MAIN-RAG: Multi-Agent Filtering Retrieval-Augmented Generation](https://arxiv.org/abs/2501.00332) | Demonstrates that setting an adaptive relevance threshold based on score distribution, rather than always returning top-k, improves RAG quality. |
| `[canonicnell]` | [Open Knowledge Graphs Canonicalization using Variational Autoencoders (CANONICNELL)](https://aclanthology.org/2021.emnlp-main.811) | Directly addresses entity canonicalization at write time, showing that normalizing entity representations during ingestion produces cleaner knowledge bases. |
| `[superlocal]` | [SuperLocalMemory: Privacy-Preserving Multi-Agent Memory with Bayesian Trust Defense Against Memory Poisoning](https://arxiv.org/abs/2603.02240) | SQLite-backed memory system with adaptive re-ranking across importance, recency, and access frequency signals. |
| `[acc]` | [Enhancing Cache-Augmented Generation with Adaptive Contextual Compression](https://arxiv.org/abs/2505.08261) | Supports staged compression with selective retrieval: preloaded compressed context at varying fidelity levels. |
| `[neuro-align]` | [Large Language Models Show Signs of Alignment with Human Neurocognition During Abstract Reasoning](https://arxiv.org/abs/2508.10057) | LLM internal representations correlate with human frontal brain activity (EEG) during abstract reasoning. |
| `[outgrow]` | [From Language to Cognition: How LLMs Outgrow the Human Language Network](https://arxiv.org/abs/2503.01830) | Brain alignment tracks formal linguistic competence more than functional competence. |
| `[cogload]` | [United Minds or Isolated Agents? Exploring Coordination of LLMs under Cognitive Load Theory](https://arxiv.org/abs/2506.06843) | Bounded working memory degrades performance under information overload — the constraint that motivates IA compression. |
| `[cogworkspace]` | [Cognitive Workspace: Active Memory Management for LLMs](https://arxiv.org/abs/2508.13171) | Passive retrieval is insufficient; proposes active, hierarchical, task-driven memory management. |
| `[sys-theory]` | [Agentic AI Needs a Systems Theory](https://arxiv.org/abs/2503.00237) | Agentic AI systems exhibit emergent social dynamics requiring formal systems theory. |
| `[social-sim]` | [LLM Social Simulations Are a Promising Research Method](https://arxiv.org/abs/2504.02234) | LLMs can model social dynamics and infer behavioral patterns. |
| `[fractal-neural]` | [Fractals in the Nervous System](https://pmc.ncbi.nlm.nih.gov/articles/PMC3059969/) | Power-law scaling and self-similar patterns at every level of neural organization. |
| `[fractal-hippo]` | [Fractal memory structure in the spatiotemporal learning rule](https://pmc.ncbi.nlm.nih.gov/articles/PMC12748222/) | Hippocampal synaptic weights form fractal structures through fractal coding during memory formation. |
| `[fractal-brain]` | [The fractal brain: scale-invariance in structure and dynamics](https://academic.oup.com/cercor/article/33/8/4574/6713293) | The brain operates with scale-free dynamics where the same patterns repeat across temporal scales. |
| `[sanc]` | [An Axiomatic Approach to General Intelligence: SANC(E3)](https://arxiv.org/abs/2601.08224) | Self-similar hierarchical organization emerges inevitably from finite capacity plus compression plus competitive selection. |
| `[halluc-geo]` | [What Geometric Visual Hallucinations Tell Us about the Visual Cortex](https://www.math.utah.edu/~bresslof/publications/01-3.pdf) | The visual cortex generates geometric patterns that are eigenfunctions of the cortical connectivity matrix. Analogous to LLM opaque encodings. |
| `[controlled-halluc]` | [Being You: A New Science of Consciousness](https://en.wikipedia.org/wiki/Being_You) | Seth's controlled hallucination theory: perception is active construction. The memery system constructs working memory by the same mechanism. |
| `[criticality-memory]` | [Homeostatic maintenance of criticality and memory consolidation](https://arxiv.org/abs/2502.10946) | The brain operates near a critical point; spontaneous activity during rest maintains criticality while enabling memory consolidation. |
| `[soc-walks]` | [Self-Organized Criticality through Memory-Driven Random Walks](https://arxiv.org/abs/2505.19296) | Memory-driven random walks produce self-organized criticality with power-law avalanche distributions. |
| `[map-territory]` | [Science and Sanity](https://en.wikipedia.org/wiki/Science_and_Sanity) | "The map is not the territory." All representations are lossy; the question is whether what's preserved is structurally adequate. |
| `[form-constants]` | [The Geometry of Psychedelic Experience](https://en.wikipedia.org/wiki/Heinrich_Kl%C3%BCver) | Four universal form constants that recur across all altered states. LLMs exhibit analogous form constants shaped by architecture, not input. |
| `[causal-emerge]` | [Causal Emergence 2.0: Quantifying emergent complexity](https://arxiv.org/abs/2503.13395) | Different scales of a system as slices of a higher-dimensional object. Each layer of the memery pipeline preserves different causal structure, making staged representations causally irreducible rather than merely convenient. |
| `[constructal]` | [Constructal Evolution as a Nonsmooth Dynamical System](https://arxiv.org/abs/2603.06705) | Flow systems evolve to provide progressively easier access to the currents that flow through them. Retrieval paths self-optimize through use, grounding the metabolism tier movement in physical law. |
| `[turing-topics]` | [Information content and optimization of self-organized Turing patterns](https://www.pnas.org/doi/10.1073/pnas.2410705122) | Reaction-diffusion generates spatial patterns that optimize information content. Topic merging is reaction-diffusion in information space. |
| `[prigogine]` | [Self-Organization in Non-Equilibrium Systems](https://en.wikipedia.org/wiki/Dissipative_system) | Dissipative structures maintain order by exporting entropy. Nightly consolidation is a dissipative structure: the interpreted store maintains order by processing capture buffer disorder during idle cycles. |
| `[stigmergy]` | [From Ants to Economy: Stigmergy, a Framework for Complex Self-Organizing Systems](https://doi.org/10.1007/s10699-015-9480-y) | Indirect coordination through environmental modification. The capture buffer is a pheromone trail. |


# Memery

Existing memory systems for LLMs do one of two things: store and search
(vector DBs, RAG), or summarize and forget (conversation memory). Both lose
information. The first pulls back too much noise. The second compresses too
early and too permanently.

Memery does something different:

- **Capture everything, process later.** Raw input is preserved. Importance
  scoring happens in batch, with full-day context — not inline, not
  one-entry-at-a-time.
- **Compress, don't drop.** A retrieval-time compression engine fits more
  memories into working context by quilting patches at semantic seams. The
  AI sees 20 relevant memories, not 5.
- **Learn by watching.** Preferences, patterns, importance — inferred from
  behavior over time. No "what should I remember?" prompts.

The memory gets smarter over time, not just bigger.

---

**vibeNcoder** — The AI organizes knowledge by its own judgment. Everything
captured, batch-processed during idle time. SQLite. Zero LLM calls on read.

**Infinite Approximator** — Retrieval-time compression. Patches quilted at
semantic seams. Swappable compression modules. Zero-shot, no fine-tuning.

Together: remembers everything, compresses everything, fits everything.

---

[Technical Specifications →](./technical-specs.md)
[ELI5 →](./eli5.md)
[Overview →](./memery-overview.md)

## Status

Stage 0 — Paranoid Logger. Capturing everything, processing nothing yet.
Not ready for use. Follow along.

## Roadmap

**Stage 0 — Paranoid Logger** ← *we are here*

- ✅ OpenClaw plugin skeleton (manifest, hooks, config system)
- ✅ Append-only SQLite capture buffer (WAL mode, indexed)
- ✅ Lifecycle hooks: before/after tool call, messages, sessions, compaction
- ✅ File capture triple: before → diff → after, gzipped archive
- ✅ Event taxonomy with tool classification
- ✅ Full config system (sizes, durations, paths — all tunable)
- ✅ Smoke tests passing
- 🔲 72-hour soak test to validate every event type fires
- 🔲 `openclaw memery stats` CLI polish
- 🔲 File capture edge cases (binary files, symlinks, permission errors)

**Stage 1+** — In progress.

---

**[Read the full technical specifications →](./technical-specs.md)**

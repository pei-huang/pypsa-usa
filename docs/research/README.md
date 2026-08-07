# Research notes

Internal working notes. Not part of the published PyPSA-USA
documentation (`docs/source/`) and not wired into the Sphinx build.

These describe a **downstream project** that consumes PyPSA-USA as an
input — a power market research platform, intended to end up in its own
repository (`power-market-platform`) whose only dependency on this one
is a network-extraction step.

**Last updated:** 2026-08-07

## Which document do I want?

Each document answers exactly one question. If you are unsure where
something belongs, this table is the arbiter.

| I want to know… | Read |
| --- | --- |
| What's broken in PyPSA-USA right now? | [../known-issues.md](../known-issues.md) |
| What does PyPSA-USA give me as an engine? | [engine-capabilities.md](engine-capabilities.md) |
| What are we building, structurally? | [platform-architecture.md](platform-architecture.md) |
| In what order, and what kills it? | [platform-roadmap.md](platform-roadmap.md) |

Reading order for someone new to the project: capabilities →
architecture → roadmap. Known-issues is a working backlog, not part of
the narrative.

## The dividing lines

Three distinctions do most of the work of keeping these separate.

**Defect vs. limitation.** A wrong number in PyPSA-USA is a defect and
belongs in [../known-issues.md](../known-issues.md). A feature the
engine was never designed to have is a limitation and belongs in
[engine-capabilities.md](engine-capabilities.md). "Coal prices are
converted wrong" is the first; "there is no ancillary-services market
model" is the second.

**Upstream vs. downstream.** `known-issues.md` is about *this*
repository and lives outside `research/` for that reason. Everything
inside `research/` is about the downstream platform and is not a
proposal to change PyPSA-USA — except where the architecture note
explicitly governs fork edits under its four-tier rule.

**Structure vs. sequence.** The architecture note says what the system
*is* and is largely timeless. The roadmap says what happens *when*,
and every phase in it ends in a gate that can kill the phases after
it. If you find yourself writing "and then we should…" in the
architecture note, it belongs in the roadmap.

## Status

Nothing is built. All of `research/` is design.

The current entry point is **Phase -1** in the roadmap — an
auction-efficiency pre-gate that can invalidate the whole plan in
roughly a week, and therefore runs before any modelling work,
including the bug fixes.

## Archive

[archive/trading-gap-analysis.md](archive/trading-gap-analysis.md) —
the original 2026-08 analysis this work grew out of. Fully superseded;
retained for provenance only. Its capability inventory became
`engine-capabilities.md` and its framing became the architecture and
roadmap notes. Do not use it as a current reference.

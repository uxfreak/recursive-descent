# DOCTRINE — The Three Principles + The Five Recurring Patterns

> Core reference for the [Recursive Descent Formalization](../SKILL.md) skill.
> Read this first; it's short and shapes everything else.

## The Three Principles

### Principle 1 — First-class provenance

**Every claim cites its source.** Not a vague reference, not a "see also" — a specific `file:lines` pointer that a verifier can resolve mechanically.

In `wan`, this is the `--ref` field on every note:

```bash
wan note add -s S001 -r Formalizer \
  --ref "<codebase>/path/to/file.rs:42-89" \
  "the function f does X via algorithm Y"
```

Without provenance, claims rot silently. With it, claims rot loudly — the validator catches stale references the next time you run it. The discipline forces you to **find the actual code that justifies the claim** before writing it down. This catches confabulation at the moment it would happen.

### Principle 2 — Breadth before depth

**Always finish the current level across all phases before descending.** L0 then all L1s, all L1s then all L2s, all L2s then L3s. Never go deep on one phase while leaving others at L0.

Why: a complete spine at level N is more useful than half-deep + half-shallow coverage. When you reach L3, the L2 map tells you what's worth deepening; without it, you guess.

The exception is when the **user explicitly redirects**. They might prioritize a specific area for project reasons. Defer to that, but offer the breadth-first default first.

### Principle 3 — Stop at named functions

**Don't unfold algorithmic primitives line-by-line.** A documented function `f` is a citation, not a re-derivation. If `f` is Knuth-Plass line breaking, you formalize *what it does* (DP over breakpoints minimizing badness²) and cite the file. You don't unfold the DP table state machine unless explicitly asked.

The granularity floor is **named functions in the codebase**. Below that — algorithmic detail, mathematical derivations, external libraries — is opt-in deepening, never default.

This is what keeps the formalization tractable. A typst-sized codebase has ~1500 public symbols; documenting all of them at full depth would be a 500-session project. The granularity floor cuts it to ~30 sessions.

## The Five Recurring Patterns

These are patterns observed across multiple unrelated sub-systems in the typst formalization. When you see them, name them. They make the formalization legible.

### Pattern A — Decompose into independent axes, then constrain greedily

When a layout/positioning problem can be decomposed into independent dimensions (vertical/horizontal, x/y, header/footer), the algorithm typically:

1. Solves each dimension separately (max-of-min-bounds, three-pass greedy, etc.)
2. Composes the results with no cross-axis iteration

Examples in typst: math scripts (vertical shifts + horizontal positions, both greedy), grid sizing (column-pass + row-pass, no cross-iteration), table cells (row heights + column widths independent).

When the dimensions ARE coupled (CSS Grid with `grid-template-areas`), you need a real solver. Notice when the codebase avoids true 2-D constraint problems — that's a design choice with major implementation consequences.

### Pattern B — Carrier IR distinct from generic Content

Some sub-systems have **structural constraints** (math has nucleus/sub/sup positions; grid has cells in (x,y); PDF tagging has parent/child trees) that don't fit a generic open-element-set carrier. These get **typed sub-IRs** built at construction-time.

When formalizing: identify whether each phase consumes generic Content or a typed IR. Typed IRs come with their own dispatch tables (one handler per kind). Generic Content uses extensible element systems (typst's `#[elem]` macro, similar in other languages).

### Pattern C — Aggregator pattern

A specialized sub-system constrained to its domain still needs full layout power. Solution: **recurse into the generic layout** for arbitrary embedded content.

Examples in typst: grid's `layout_cell` recurses into the generic `layout_fragment`; math's `layout_external` recurses into `layout_frame`; HTML's `HtmlNode::Frame` variant embeds a Frame inline.

When formalizing: notice the back-edges. Specialized sub-systems are not closed; they're *aggregators* that compose generic capability.

### Pattern D — Font-table-driven primitives with variant + assembly fallback

When the codebase consults pre-computed structured data (font tables, lookup tables, configuration), the typical pattern is:

1. **Variant lookup** — try a pre-designed entry first (cheap, common path)
2. **Assembly from primitives** — when no variant fits, compose from atomic parts (slow, fallback)

Example in typst: stretchy operators (MathVariants for pre-designed sizes, GlyphAssembly for unbounded scaling). The pattern: load the precomputed solution if it fits; assemble only when it doesn't.

When formalizing: look for the variant/assembly split. It's often where the algorithm's real complexity lives.

### Pattern E — Single-line load-bearing details

Some implementations have one or two lines that, if removed, break something visually subtle but persistently. These are **the load-bearing details** — they encode taste, accumulated wisdom, or workarounds for specific real-world bugs.

Examples in typst:
- `br_kern -= base.italics_correction` — without it, post-subscripts on italic glyphs float right
- `+ index.descent` — without it, descenders in radical indices collide with the surd
- paren-padded matrix heights — without them, matrices in a paragraph render with inconsistent vertical extents
- `with_accent_attach(base_attach)` forwarding — without it, nested accents go to geometric center

When formalizing: identify and CALL OUT these single-line details by name. They're often hidden in a 200-line function but they're what makes the difference between code that looks right and code that looks algorithmic.

## The doctrine in one sentence

> Every claim has a citation, every level is complete before the next begins, and the floor is named functions — anything deeper is opt-in.

# CASE STUDY — The Typst Formalization

> Reference for [Recursive Descent Formalization](../SKILL.md). A real
> project that applied this methodology end-to-end. Concrete numbers,
> specific shapes, honest accounting of what worked.

## The Project

**Target**: [typst](https://github.com/typst/typst) — a modern typesetting system (Rust, ~17 crates, ~200K lines).

**Pinned commit**: `367d09b`.

**Outcome**: 33 sessions, 33 commits, 26 formalization markdown files, ~7400 lines of formalization output, 184 verified citations, authoritatively complete by the four-check protocol.

## The Shape That Emerged

### The one-line type (L0)

```
render : World × OutputKind → Warned (Result Bytes)
```

Where `World = Fonts × Files × Config × Preferences` is threaded as a value (not a Reader monad), errors as `Result`, the introspection cross-pass as a bounded fixed-point (max 5 iterations).

**Key insight from L0**: the only non-trivial control structure in the entire pipeline is the introspection fixed-point loop. Everything else is recursive descent.

### The seven L1 phases

```
10-frontend  : String × FileId → Source         (lex, parse, AST)
20-eval      : Source → Module                  (AST → typed Content)
30-realize   : Content → [Pair]                 (flatten + show-rules)
40-layout    : [Pair] → PagedDocument           (THE GEOMETRY KERNEL)
50-paint     : PagedDocument → DrawingOps       (per-backend walkers)
60-export    : DrawingOps → Bytes               (PDF/SVG/PNG/HTML)
70-effects   : (cross-cutting concerns)         (World, Engine, errors, memo, introspection)
```

### The 16 L3 files

**Math sub-tree (7)** — math layout's full fork plan:
- `stretchy.md` — OpenType MATH GlyphAssembly + Variants (operator sizing)
- `scripts.md` — sub/sup positioning with 9 MATH constants
- `fraction.md` — bar/stack/skewed (three modes)
- `radical.md` — root sign + degree (TeXbook 443/360)
- `run-and-multiline.md` — MathRun + alignment-point columns
- `table.md` — math-spacing-aware columns (paren-padded)
- `accent.md` — accent_attach diacritic positioning

**Inline pipeline (4)**:
- `linebreak.md` — Knuth-Plass + 2-pass acceleration
- `shaping.md` — rustybuzz bridge, font fallback
- `cluster-reconstruction.md` — M-to-N codepoint↔glyph mapping (PDF ActualText provenance)
- `cjk-punctuation.md` — W3C CLREQ rules (Gb/Cns/Jis)

**Other (5)**:
- `grid.md` + `grid-internals.md` — 3-pass constraint solver + Cell::resolve
- `internals.md` (reparser) — parser glue + replace_children + Span packing
- `pdf-tag-tree-build.md` — upfront tag-tree construction (Phase A of PDF tagging)
- `pdf-tagged-tables.md` — PDF/UA tagged tables (the gnarliest piece)

**Effects (2)**:
- `comemo.md` — constraint-tracking memoization
- `memory.md` — bumpalo arena semantic-transparency verification

### The recurring patterns surfaced

From [DOCTRINE.md](DOCTRINE.md) the five patterns — every one was observed at least twice in unrelated places:

- **Pattern A** (decompose into axes, constrain greedily) — math scripts, grid sizing, table cells, linebreak-by-pass
- **Pattern B** (typed sub-IR distinct from generic Content) — math IR, grid CellGrid, PDF tag tree
- **Pattern C** (aggregator calls back into generic) — grid.layout_cell → layout_fragment, math.layout_external → layout_frame, HtmlNode::Frame
- **Pattern D** (font-table variant + assembly fallback) — stretchy operators, glyph shaping, script variants
- **Pattern E** (single-line load-bearing detail) — italics-correction subtraction, +index.descent deviation, paren-padded matrix heights, accent_attach forwarding, axis_height baseline

## The Validation Run

```
Cited-Claim Audit:       184/184 verified  ✓
Coverage Matrix:          38% documented + 43% explicitly deferred + 19% below-granularity
Cross-Reference Check:    82/82 markdown + 79/79 wan links  ✓
Polish Pass:              tone/terminology/forward-refs consistent  ✓
```

**Four issues caught by validation that escaped manual review**:
1. Line range exceeded file length (`fragment.rs:397-815` — file only 786 lines)
2. Brace-expansion path in a wan note (`src/{pages,flow,...}`)
3. Broken markdown links using `../` when `../../` was needed (2 instances)

All four fixed in place. After fixes: zero outstanding issues.

## Three Doc-Rot Fixes During the Descent

These are case studies of what the **deeper drill catches that the earlier level missed**:

**Case 1**: `wan trace` typo — an L1 file claimed the Traced field "powers wan trace debug mode." There's no wan trace; the function is typst::trace at lib.rs:87. Caught by global grep during the synthesis session.

**Case 2**: `CjkPunctStyle 6 vs 3 styles` — the L1 shaping.md said the enum had 6 variants ({Cn, Tw, Hk, Jp, Kr, Gb}); the actual enum is 3 variants ({Gb, Cns, Jis}). Caught when the L3 drill on CJK punctuation actually read the enum definition.

**Case 3**: `Arenas semantic transparency claim` — the L1 files included Arenas in the Rust-faithful type signatures but claimed semantic transparency. The L2 drill on memory verified this claim via three leak-vector checks (hidden state, identity, order). All three clean, so the claim holds and the simpler arena-free signatures are semantically valid.

**The methodology lesson**: the deeper drill **is** the mechanism that catches the earlier level's errors. Skip the deeper drill, accept the earlier level's wrong claims. Build the deeper drill, fix them.

## The Methodology's Meta-Lessons

From `INSIGHTS.md §10 (The Methodology)`:

**What worked**:
- Recursive descent breadth-first at each level
- Forced reflection at session boundaries (the session summaries cumulatively wrote INSIGHTS)
- First-class provenance (`--ref FILE:LINES`) made every claim verifiable
- `wan task` work tree with auto-pop on `done` mirrored actual mental flow
- `--from-file` flag let Unicode math chars survive the shell

**What didn't work**:
- Multi-line commit messages with `cat <<'EOF'` got shell-mangled twice (Unicode arrows + em-dashes). Switched to `git commit -F /tmp/msg.txt` mid-project.

**What I'd do differently**:
- Use `git commit -F` from session 1.
- Extend `wan note bulk` to carry all the new fields (taskIds, links, refs, detail) — would have removed the heredoc-per-note ceremony.
- Declare the granularity floor more explicitly upfront. The implicit rule "stop at named functions" is what held the project together; articulating it in session 1 would have avoided some over-ambitious early drilling.

## The Honest Statement (from typst's COMPLETION.md)

> The Typst formalization documents `render : World × OutputKind → Warned (Result Bytes)` as a pure function of explicit inputs, with the only non-trivial control structure being a bounded fixed-point loop over an introspector. The architecture is documented at four levels of recursive depth across 26 markdown files: L0 unified spine, 7 L1 phase overviews, 4 L2 sub-system drills, and 16 L3 deep drills covering math layout's full sub-tree (7 files), the inline pipeline's load-bearing primitives (4 files), the reparser and grid internals (2 files), the PDF tagging system (2 files), and the cross-cutting effects (2 files).
>
> Every claim has a `--ref FILE:LINES` citation; every citation has been verified against typst commit `367d09b`. Every public typst symbol in the rendering pipeline crates is either documented (38%) or explicitly deferred per the granularity policy (43%) or below-granularity stdlib enumeration (19%). All cross-references — markdown links and wan note links — resolve. The narrative is internally consistent end-to-end.
>
> The work is complete by the criterion declared at session start. Any future drift can be detected mechanically by re-running the four validation checks. Any future deepening can begin from `wan resume` with full context restored in 1 second.

## What to Model

If you're formalizing a new codebase, expect:

- **L0 spine** — 1 session. Produces `00-overview.md` v1.
- **L1 phases** — 5-10 sessions. One per phase. Surfaces L2 candidates.
- **L2 sub-systems** — 5-15 sessions. One per substantial sub-system.
- **L3 deep-drills** — 5-15 sessions. One per load-bearing primitive.
- **Synthesis** — 1-2 sessions. Update `00-overview.md` v2. Write `INSIGHTS.md`.
- **Validation** — 1 session. Run the four checks. Write `COMPLETION.md`.

Total: 20-35 sessions depending on codebase size. Typst was at the high end (33 sessions) because it's structurally rich (multiple sub-pipelines, extensive font-table-driven logic).

## Where to look

The typst-formalization workspace lives at:

```
/Users/kasa/Downloads/references/typst-formalization/
├── formalization/         # 26 markdown files — the core artifact
├── validation/            # 3 bash scripts + per-project ref validator
├── .wan/                  # 31 sources, 193 notes, 38 tasks, 79 links
├── INSIGHTS.md            # meta-essay (310 lines)
├── COMPLETION.md          # validation report
├── README.md              # entry points
└── .git/                  # 33 commits, all sessions tracked
```

Resume cold any time with:

```bash
cd /Users/kasa/Downloads/references/typst-formalization
wan resume
```

Or read `INSIGHTS.md` first for the meta-picture, then `formalization/00-overview.md` for the structural map.

## Why this case study matters

The typst project validated the methodology **end-to-end**:

1. The workflow scaled (~33 sessions without drift).
2. The validation protocol caught real issues (4 fixes).
3. The documentation-rot doctrine paid off (3 fixes from deeper drills).
4. The resulting artifact is genuinely useful (anyone can cite file:line X and verify it resolves).
5. The methodology is **re-executable on a different codebase** without fundamental change.

That's the claim this skill ships with. Apply it to any codebase, expect similar outcomes.

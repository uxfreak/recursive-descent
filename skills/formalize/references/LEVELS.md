# LEVELS — L0 / L1 / L2 / L3 Definitions

> Reference for [Recursive Descent Formalization](../SKILL.md). What each
> level means, when to fork, when a level is "done."

## L0 — The Top-Level Pure Function

**Goal**: derive the codebase's pure-function signature.

**Output**: `formalization/00-overview.md` (initial draft — gets re-synthesized at the end).

**What's in it**:
- The one-line type: `fn(World × Input) → Warned (Result Output)` or equivalent
- The carrier types (the substrate `World`, the input, the output, the error/warning channel)
- The composed pipeline as one expression: `entry = read ⨟ parse ⨟ evaluate ⨟ ...`
- The geometric / semantic framing: what is this codebase fundamentally *doing*?

**How long**: one session. Read entry points (`lib.rs`, `main.rs`, `mod.rs`), trace the top-level call graph, identify carrier types.

**When done**: you can name every named function in the top-level pipeline and what it produces.

**Common pitfalls**:
- Going too deep too fast. L0 is the SPINE, not the full breakdown.
- Hiding non-trivial control structures (fixed-point loops, retries, etc.). Make them explicit.
- Forgetting to characterize the substrate (`World` is not just a parameter — it's where all dependency injection lives).

## L1 — Per-Phase Overviews

**Goal**: one overview per pipeline phase identified at L0.

**Output**: `formalization/<NN>-<phase>/00-overview.md` per phase.

**What's in it** (consistent structure across all L1 files):
1. Opening blockquote: phase, source crate/module, commit pin
2. **L1 Function** — typed signature with carriers spelled out
3. **Carrier Types** — every struct/enum the phase consumes or produces
4. **The algorithm** — the actual reduction in pseudocode
5. **Geometric / Semantic Framing** — what does this phase contribute to the bigger picture
6. **What's Genuinely Hard** — 3-5 places where the implementation is non-trivial
7. **Sub-Candidates Surfaced** — sub-systems that warrant L2 drill
8. **Provenance** — table of file:line pointers to every claim
9. **Connections** — how this phase consumes / produces / interacts with adjacent phases

**How long**: one session per L1 phase. Typical codebase has 5-10 L1 phases.

**When done**: every public function in the phase is mentioned (documented or explicitly deferred), with line citations. The "Sub-Candidates Surfaced" list is the L2 backlog.

**Forking decision**: if the L1 file would exceed ~400 lines, the phase has multiple sub-systems and you should fork early. Each sub-system gets its own L1 file. The math layout in typst forked into 12 sub-systems; that became 1 L1 overview + 7 L2 sub-files.

## L2 — Sub-System Drills

**Goal**: each substantial sub-system gets its own file with full algorithmic detail.

**Output**: `formalization/<phase>/<sub-system>.md` (e.g. `40-layout/grid.md`, `40-layout/inline/linebreak.md`).

**What's in it** (same shape as L1 files):
- L2 function signature
- Decomposition (sub-functions, dispatch tables, state machines)
- The full algorithm — not just "what it does" but "how, step by step"
- Geometric / semantic framing AT THIS LEVEL
- Hard parts, sub-candidates, provenance, connections

**How long**: one session per L2 file. Typical codebase has 5-15 L2 sub-systems.

**When to drill (L2 candidates)**:
- Surfaced as a sub-system in some L1 file
- Has its own algorithm (not just dispatching to others)
- 100-1000 lines of source typically
- Worth understanding for real engineering (debugging, performance, extension)

**When NOT to drill**:
- Mechanical wrappers (just dispatch, no algorithm)
- External library bindings (cite the library, don't unfold)
- Below the granularity floor (algorithmic primitives, like `sort()`)

**When done**: the sub-system's algorithm is fully captured. Future engineers reading the L2 file should be able to **modify** the implementation safely without re-discovering the design constraints.

## L3 — Load-Bearing Primitive Drills

**Goal**: the single most important primitive within a sub-system.

**Output**: a per-primitive markdown file (e.g. `40-layout/math/stretchy.md`, `40-layout/inline/cluster-reconstruction.md`).

**What's in it**:
- A focused walkthrough of ONE algorithm
- Often: a discrete-continuous optimization, a state machine, a constraint solver
- The mathematical content, not just the API
- Sub-candidates if any (these usually bottom out at external dependencies)

**How long**: one session per L3 file. Typical codebase has 5-15 L3 drills.

**When to drill (L3 candidates)**:
- Surfaced in L2 as a load-bearing primitive
- The "what's genuinely hard" that L2 surfaced earns its own file
- An algorithm that taste-makes the entire system (Knuth-Plass for typesetting, Liang for hyphenation, etc.)

**When the descent bottoms out**:
- External dependency (crate, library, system call) — cite, don't drill
- Algorithmic primitive (DP table, sort, hash) — characterize, don't unfold
- Below-granularity helpers (per-byte parsing, error formatting) — skip

## When to FORK a level

Fork when the file would get too big OR the sub-systems are conceptually distinct.

The classic fork pattern (from typst):
- L1 `40-layout/00-overview.md` mentions math as "its own sub-pipeline"
- That mention becomes a L2 fork: `40-layout/math/00-overview.md`
- The L2 overview surfaces 7 candidates: `stretchy`, `scripts`, `fraction`, `radical`, `run-and-multiline`, `table`, `accent`
- Each becomes an L3 file

Don't be afraid to fork. A page-and-a-half file with five concerns is harder to maintain than five tight files cross-linked.

## When a level is "done"

Apply the **completeness criterion** at each level:

- **L0 done**: the one-line type + composed pipeline + carrier list are written. Every named top-level function appears.
- **L1 done**: every L0-mentioned phase has its own overview file with the 9-section structure above. Every phase's sub-systems are inventoried.
- **L2 done**: every substantial sub-system has its own file. Every L1's "Sub-Candidates Surfaced" list is either drilled or explicitly deferred.
- **L3 done**: every load-bearing primitive has its own file. The remaining ungrouped public symbols are all stdlib, external, or below-granularity.

Then synthesize:
- Update `00-overview.md` to integrate L1+L2+L3 content
- Write `INSIGHTS.md` with the meta-essay (recurring patterns, surprises, methodology lessons)
- Run validation
- Write `COMPLETION.md`

## How many sessions per project

| Codebase size | Sessions |
|---|---|
| Small (~5K lines, ~50 public symbols) | 8-12 |
| Medium (~50K lines, ~500 public symbols) | 20-30 |
| Large (~200K lines, ~1500 public symbols) | 30-50 |

Typst was medium (~14K lines surveyed across ~1500 public symbols, ~33 sessions). Use that as the calibration.

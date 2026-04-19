# recursive-descent

**A Claude Code plugin for rigorously formalizing the architecture of an unfamiliar codebase.**

Ships:
- The `/recursive-descent:formalize` skill (orchestrates the methodology)
- The `wan` CLI (project-memory + workflow tool) pre-built, added to PATH
- Reference documentation (doctrine, levels, session protocol, validation, case study)

## What it does

Given an unfamiliar codebase, this plugin drives you (or a Claude Code agent) through a **Recursive Descent Formalization**:

```
L0 — one-line pure-function type                    (1 session)
L1 — per-phase overviews                            (5-10 sessions)
L2 — sub-system drills                              (5-15 sessions)
L3 — load-bearing primitive drills                  (5-15 sessions)
Synthesis — unified overview + meta-insights        (1-2 sessions)
Validation — four-check protocol + COMPLETION.md    (1 session)
```

**Output**: a navigable formalization tree (~25 markdown files for typical-size projects), every claim cited to specific `file:lines`, with a validation protocol that asserts authoritative completion.

**Guarantee**: the validation protocol catches stale citations, missing symbols, and broken cross-references mechanically. "Authoritatively complete" is a declarable, re-verifiable criterion.

## Proof of work

This methodology was applied to the [typst typesetting system](https://github.com/typst/typst) over 33 sessions, producing 26 markdown files, 184 verified citations, and a published COMPLETION report. See [CASE-STUDY.md](skills/formalize/references/CASE-STUDY.md) for the full write-up.

## Install

```bash
# Add the plugin to Claude Code
/plugin install https://github.com/<your-org>/recursive-descent

# Or clone and add locally
git clone https://github.com/<your-org>/recursive-descent ~/.claude/plugins/recursive-descent
```

**Runtime dependency**: [`bun`](https://bun.sh) (the TypeScript runtime). If you don't have it:

```bash
curl -fsSL https://bun.sh/install | bash
```

Why bun: the plugin ships `wan` as vendored TypeScript source. On first invocation, `bun` runs it directly (~100ms startup — fast enough). Alternatively, compile once into a standalone binary for faster startup:

```bash
cd ~/.claude/plugins/recursive-descent/wan-src
bun build src/index.ts --compile --outfile ../bin/wan-bin
# Subsequent `wan` invocations use the cached binary.
```

After install:
- `/recursive-descent:formalize <path-to-codebase>` invokes the skill
- `wan` is on PATH (via `bin/wan`; `wan --help` for reference)

## Quick start

```bash
# In Claude Code, pointing at an unfamiliar codebase:
/recursive-descent:formalize /path/to/some/codebase
```

The skill will:
1. Scaffold a workspace (`<codebase>-formalization/` alongside the target)
2. Configure per-project validators (refuses bad citations at insertion time)
3. Survey the codebase and draft the L0 spine
4. Drive L1 → L2 → L3 descent session by session
5. Run the four-check validation protocol
6. Produce the authoritative COMPLETION report

Every session commits to git. Every claim cites a `file:lines`. Every citation is validated. Nothing drifts silently.

## What this plugin does NOT do

- **It does not read the codebase for you.** The skill orchestrates; you (or the agent) reads.
- **It does not generate fictitious detail.** Every claim has a ref; every ref is validated.
- **It does not bypass the granularity floor.** Stop at named functions; don't unfold algorithmic primitives unless explicitly requested.

## The doctrine

See [DOCTRINE.md](skills/formalize/references/DOCTRINE.md) for the three principles + five recurring patterns that shape the methodology.

TL;DR:
1. **Every claim cites its source.**
2. **Breadth before depth** (finish L1 before any L2; L2 before any L3).
3. **Stop at named functions** (don't unfold primitives).

## Structure

```
recursive-descent/
├── .claude-plugin/
│   └── plugin.json                # manifest
├── skills/
│   └── formalize/
│       ├── SKILL.md               # the orchestrating skill
│       └── references/
│           ├── DOCTRINE.md        # 3 principles + 5 patterns
│           ├── LEVELS.md          # L0/L1/L2/L3 definitions
│           ├── SESSION-PROTOCOL.md # 6-command core loop
│           ├── VALIDATION.md      # 4-check protocol
│           └── CASE-STUDY.md      # typst as proof-of-work
├── bin/
│   └── wan                        # bash wrapper (runs wan-src/ via bun)
├── wan-src/                       # vendored wan-cli TypeScript source
├── README.md                      # this file
├── LICENSE                        # MIT
└── CHANGELOG.md                   # versioned
```

## License

MIT. See [LICENSE](LICENSE).

## The wan CLI

`wan` is the Work Activity Notes CLI — project memory and workflow tool. It's the active component of the methodology (the doctrine without wan is just advice; with wan it's a reproducible process).

Full reference: `wan --help`, `wan guide`, `wan philosophy`.

Key commands used every session:
- `wan resume` — ingest state at session start
- `wan task focus / done` — track the working focus
- `wan session start / end` — bracket the work window
- `wan note add --ref FILE:LINES` — atomic finding with provenance
- `wan link add --kind refines/requires/...` — cross-link dependencies
- `wan status set` — update narrative state
- `wan doctor` — check consistency before commit

## Citations

If you use this methodology in published work, cite:

```
recursive-descent-formalization (Claude Code plugin), v0.1.0.
https://github.com/<your-org>/recursive-descent
```

And link back to the typst case study for concrete evidence the methodology produces useful artifacts.

## Contributing

Bug reports, improvements, and new case studies welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) if present.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

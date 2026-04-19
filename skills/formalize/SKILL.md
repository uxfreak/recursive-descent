---
name: recursive-descent:formalize
description: Formalize the architecture of an unfamiliar codebase as a recursive structural map (L0 → L1 → L2 → L3) with first-class provenance per claim. Use when the user wants to deeply understand a codebase, document it for future agents, build a complete architectural picture, or apply rigorous reverse-engineering to an external library/framework. Spawns a wan workspace, drives the recursive descent, validates completeness mechanically. Output: a navigable formalization tree (~25 markdown files for a typical-sized project) with every claim traceable to specific file:lines, plus a validation protocol that asserts authoritative completion.
when_to_use: When the user wants to thoroughly understand a codebase's architecture and produce a durable, citable artifact. Triggers - "formalize", "document the architecture of", "produce an architectural map of", "deeply understand", "reverse-engineer", "create a complete picture of how X works". Skip for - quick lookups, single-file questions, short tasks under one session, or when the user just wants to read code.
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
argument-hint: <path-to-codebase> [optional: scope-or-focus]
---

# Recursive Descent Formalization

You are about to apply the **Recursive Descent Formalization** methodology to a codebase. This is a structured, rigorous, multi-session process that produces a navigable architectural formalization with first-class provenance per claim.

## What this skill does

Drives the user through:

1. **Workspace setup** — scaffold a `wan` project workspace alongside the target codebase
2. **L0 spine** — the top-level pure function (one session)
3. **L1 phases** — one session per pipeline phase (5-10 sessions)
4. **L2 sub-systems** — one session per substantial sub-system (5-10 sessions)
5. **L3 deep-drills** — one session per load-bearing primitive (5-15 sessions)
6. **Synthesis** — unified overview + meta-insights (1-2 sessions)
7. **Validation** — cited-claim audit + coverage matrix + cross-reference check (1 session)

Total: typically 20-35 sessions over the lifespan of a project. Each session ends with a git commit, a wan session summary, and updates to the work tree.

## The doctrine — read these before starting

Before the user's first request to formalize, READ these reference files:

- [`references/DOCTRINE.md`](references/DOCTRINE.md) — the three principles + the recurring patterns
- [`references/LEVELS.md`](references/LEVELS.md) — what L0/L1/L2/L3 mean and when to fork
- [`references/SESSION-PROTOCOL.md`](references/SESSION-PROTOCOL.md) — the six-command core loop
- [`references/VALIDATION.md`](references/VALIDATION.md) — the four-check protocol
- [`references/CASE-STUDY.md`](references/CASE-STUDY.md) — the typst formalization as proof-of-work

These are loaded on demand. You don't need them all in context for every action; reach for the right one when it's needed.

## Quick start (the first session)

Given the user's `/recursive-descent:formalize <codebase>` invocation:

```bash
# 1. Set up the workspace
mkdir <codebase>-formalization && cd <codebase>-formalization
mkdir formalization validation
wan init

# 2. Configure the per-project ref validator (so wan refuses bad citations
#    at insertion time, not at the validation pass)
cat > validation/validate-ref.sh <<'EOF'
#!/usr/bin/env bash
# Adjust TARGET_ROOT to point at the codebase being formalized.
TARGET_ROOT="<absolute-path-to-codebase>"
path="${WAN_REF_PATH#<codebase-name>/}"
case "$path" in ""|<codebase-name>) exit 0 ;; esac
full="$TARGET_ROOT/$path"
if [[ "$path" != *.<expected-ext> ]]; then
  [[ -d "$full" || -f "$full" ]] && exit 0
  echo "validator: path not found: $path" >&2 && exit 1
fi
[[ -f "$full" ]] || { echo "validator: file not found: $path" >&2; exit 1; }
if [[ -n "${WAN_REF_LINES}" ]]; then
  file_len=$(wc -l < "$full")
  max_line=$(echo "${WAN_REF_LINES}" | tr ',' '\n' | tr '-' '\n' | sort -n | tail -1)
  (( max_line > file_len + 5 )) && {
    echo "validator: line range out of bounds" >&2; exit 1
  }
fi
exit 0
EOF
chmod +x validation/validate-ref.sh

python3 -c "
import json
with open('.wan/config.json') as f: c = json.load(f)
c['validators'] = {'ref': 'validation/validate-ref.sh',
                   'markdownRoot': 'formalization'}
with open('.wan/config.json', 'w') as f: json.dump(c, f, indent=2)
"

# 3. Seed the work tree with the L0 mission
wan task add "Formalize <codebase> as math (Source × World → Output)" -i "<the-pure-function-shape>"
wan task add "Setup workspace + scaffolding" -p T001 --focus
wan task add "L0 spine: top-level pipeline" -p T001
wan task add "L1 phases: <list-from-codebase-survey>" -p T001
wan task add "L2 sub-systems: per-phase named functions" -p T001
wan task add "L3 deep-drills: load-bearing primitives" -p T001
wan task add "Synthesize unified overview" -p T001

# 4. Open the first session
wan session start "scaffold workspace + initial codebase survey"
git init -q && git add -A && git commit -q -m "init"
```

Then BEGIN THE SURVEY:
- Survey the codebase's directory structure (`ls`, `tree`, `wc -l`)
- Identify the top-level entry points (`grep "pub fn" lib.rs`, equivalent for the language)
- Find the carrier types (the structs/enums passed between phases)
- Read the README and module docstrings
- Form a hypothesis about the L0 pipeline shape

Then write the seed source dump (`wan source add`) capturing what you found.

## The session loop (every session after the first)

```bash
wan resume                                  # 1. ingest state
wan task focus T0XX                         # 2. set the working leaf
wan session start "intent text"             # 3. open the time window
# ... read source, write formalization/X/Y.md, gather provenance ...
wan note add -s S0X -r Formalizer \         # 4. atomic findings
  --ref "<file:lines>" --task T0XX -F /tmp/note.md
wan link add S0X-NN SY-MM --kind refines    # 5. cross-link as you notice
wan task done T0XX                          # 6. auto-pop focus
wan status set "..."                        # 7. update narrative state
wan session end "summary"                   # 8. close + auto-attach
git commit -F /tmp/commit-msg.txt           # 9. version everything
```

**Do not skip steps 5 and 7.** Cross-links create the dependency graph; status preserves narrative continuity across sessions.

## What the user sees vs what you do

The user invokes `/recursive-descent:formalize <codebase>` once. You then drive ALL subsequent sessions, asking the user only:

- **At the end of L0**: "Here's the spine I derived. Should I proceed L1 breadth-first across these N phases, or focus on one area first?"
- **After each L2 drill**: "This drill surfaced K sub-candidates. Drill any now or queue for later?"
- **Before validation**: "Ready to run the four-check protocol and produce the COMPLETION report?"

Otherwise: keep moving. The user delegated the work. They want progress, not constant check-ins.

## When to ask for direction (the only times)

- **Scope ambiguity**: codebase has multiple unrelated subsystems; user must choose which to formalize.
- **Granularity floor**: when an L3 drill bottoms out at an external dependency, ask whether to drill into that dependency.
- **Pivot points**: when the surveyed pipeline shape diverges substantially from the user's stated mental model.

## Output

By the end:

- `formalization/00-overview.md` — unified end-to-end (the entry point)
- `formalization/<NN>-<phase>/00-overview.md` — per L1 phase
- `formalization/<NN>-<phase>/<sub>.md` — per L2 sub-system / L3 drill
- `INSIGHTS.md` — meta-essay (recurring patterns + 5-section structure)
- `COMPLETION.md` — validation report (the four checks)
- `validation/` — three bash scripts re-runnable any time
- `.wan/` — the project memory, git-tracked

Anyone (human or AI) can `wan resume` from the workspace and pick up cold.

## What this skill does NOT do

- It does not read the codebase for you. You read; the skill orchestrates.
- It does not generate fictitious detail. Every claim has a `--ref FILE:LINES` citation that the validator at insertion time confirms resolves.
- It does not skip the validation pass. The protocol exists precisely to catch what manual review misses.
- It does not bypass the granularity floor. Stop at named functions; don't unfold algorithmic primitives unless explicitly requested.

## Anti-patterns

- **Drilling depth-first**: tempting but produces lopsided coverage. Always finish the current level breadth-first before descending.
- **Notes without provenance**: forbidden. Every claim gets a `--ref`.
- **Skipping the synthesis session**: the meta-essay is what makes the formalization an artifact, not just a corpus.
- **Bypassing validators with `--no-validate`**: the bypass exists for emergencies only. Use it and you lose the doc-rot guarantees the methodology relies on.

## When you're done

Run `wan doctor`, run the three validation scripts, write COMPLETION.md, commit. Tell the user the formalization is authoritatively complete by the criterion declared at session start, and where to find the entry points.

The doctrine is that the workspace is the artifact, not just the markdown. Future sessions resume cleanly because the `wan` state captures everything needed.

# SESSION PROTOCOL — The Six-Command Core Loop

> Reference for [Recursive Descent Formalization](../SKILL.md). The
> per-session workflow that produces all the artifacts.

## The six-command core loop

Run every session, in order:

```bash
# 1. INGEST STATE — at session start, recover full context in ~1 second
wan resume

# 2. SET FOCUS — declare what work is happening this session
wan task focus T0XX                        # existing task, OR
wan task add "<intent>" -p T0YY --focus    # new sub-task

# 3. OPEN THE TIME WINDOW — forced-reflection on session intent
wan session start "intent text"

# 4. WORK — read source, write formalization markdown, gather provenance
# (this is the bulk of the session; see below)

# 5. CAPTURE FINDINGS as atomic notes with first-class provenance
cat > /tmp/note-1.md <<'EOF'
The function f at file.rs:42-89 implements algorithm Y. Key detail: ...
EOF
wan note add -s S0X -r Formalizer \
  --ref "<codebase>/path/to/file.rs:42-89" \
  --task T0XX \
  -F /tmp/note-1.md

# Repeat for each finding (typically 5-7 per session)

# 6. CROSS-LINK as you notice dependencies
wan link add S0X-NN SY-MM --kind <calls|produces|requires|refines|relates>

# 7. UPDATE THE NARRATIVE STATE so future-you picks up cleanly
wan status set "..."

# 8. CLOSE THE SESSION — auto-attaches notes/labels created in window
wan session end "summary"

# 9. GIT COMMIT — version everything (.wan/, formalization/, validation/)
git commit -F /tmp/commit-msg.txt
```

**Steps 1, 3, 8, 9 are non-negotiable** — they're the forcing functions that prevent drift.

**Steps 5, 6, 7 are the value-creation steps** — they're what makes the workspace genuinely informative.

## What "work" actually looks like (step 4)

For an L1 phase overview session:

```bash
# Survey the phase's sources
ls /path/to/codebase/<phase>/
wc -l /path/to/codebase/<phase>/*

# Find the entry points
grep -n "^pub fn" /path/to/codebase/<phase>/lib.rs
grep -n "^pub struct\|^pub enum" /path/to/codebase/<phase>/lib.rs

# Read each entry point + its types
# (in your head; you don't need to copy them out — wan notes capture findings)

# Identify decomposition into sub-systems
# Identify carrier types passed between sub-systems
# Identify hard parts

# Write the L1 overview file
cat > formalization/<NN>-<phase>/00-overview.md <<'EOF'
# <NN> <phase> — <one-line title>
> Phase: ...
> Crate: ...
> Pinned at <codebase> commit <hash>.

## L1 Function
... etc per the structure in references/LEVELS.md
EOF
```

For an L2/L3 drill: same shape but focused on one file/sub-system.

## The note pattern

Every note carries:
- `-s S0XX` — source ID (the exploration session that surfaced it)
- `-r Formalizer` — your role
- `--ref FILE:LINES` — at least one citation; multiple OK
- `--task T0XX` — explicit attachment to the work-tree node
- `-F /tmp/note-X.md` — content from a file (avoids shell-mangling math chars)

Skip any of these and the note is weaker:
- No source: untraceable origin
- No ref: claim without evidence (FORBIDDEN; the validator refuses bad refs)
- No task: lost in the timeline; can't be reverse-looked-up
- Inline content: shell mangles Unicode arrows / em-dashes / pipes / backticks

## The cross-link kinds

Use `wan link add A B --kind <kind>` to declare dependencies between notes:

| Kind | Semantics | Example |
|---|---|---|
| `calls` | A invokes B in the actual code | `S001-03 calls S002-07` |
| `produces` | A constructs / yields B | `S005-01 produces S004-03` |
| `requires` | A depends on B (precondition) | `S012-01 requires S010-05` |
| `refines` | A is a more detailed version of B | `S017-01 refines S012-05` |
| `relates` | generic association | `S022-04 relates S019-02` |

The graph that emerges is your dependency map. By session 20, you can navigate it from memory because you built it edge by edge.

## Status discipline

`wan status set "..."` overwrites the workspace's narrative state doc. Keep it tight (1-3 paragraphs):

- Where we are in the work tree
- What just got done
- What's the next move
- Any open issues / blockers

Future-you (or future-AI) reads this first when resuming. If the status says "next: drill T012 = grid sizing", you don't need to re-derive that decision.

`wan status append "..."` adds a timestamped line — useful mid-session for breadcrumbs that don't warrant overwriting the whole status.

## The session-end summary

```bash
wan session end "summary text"
```

The summary is your micro-commit-message. It should answer:
- What got built/formalized?
- What was the surprise / finding?
- What's the cross-link to other sessions?
- What's the next session's likely focus?

These summaries cumulatively form the project narrative. INSIGHTS.md at the end is mostly written by reading session summaries in chronological order.

## The git commit pattern

For sessions with substantial Unicode (math chars, arrows), shell-mangled commit messages are a real risk. Use the file pattern:

```bash
cat > /tmp/commit-msg.txt <<'EOF'
session SES-NNN: <one-line summary>

<body — typically 30-50 lines summarizing findings, cross-links, state>

State: N sources, N notes, N tasks, N sessions.
EOF
git add -A && git commit -F /tmp/commit-msg.txt -q
```

The commit message body becomes the durable session log. If the wan state ever corrupts, the git history reconstructs everything.

## Frequency

- **Sessions per day**: 1-3 typically. Each session is 30-90 minutes of focused work.
- **Notes per session**: 5-7 atomic findings. Less means superficial coverage; more means too much in one session (split it).
- **Cross-links per session**: 2-4. Less means you're not noticing connections; more means you're over-linking.
- **Words in status**: 50-200. The status is for orientation, not narration.
- **Words in session summary**: 50-150. The summary is for retrieval, not exposition.

## Common workflow mistakes

- **Skipping `wan resume`**: re-deriving context that already exists. Always read first.
- **Notes too verbose**: notes are for atomic findings, not narratives. The narrative goes in the markdown formalization file.
- **Forgetting `--ref`**: a note without provenance is a claim without evidence. Don't ship it.
- **Forgetting `--task`**: the note exists in time but not in the work-tree. Reverse-lookup breaks.
- **Multi-line commit via `git commit -m`**: shell mangles math chars. Use `git commit -F /tmp/msg.txt`.
- **Ending session without `wan session end`**: the time-window stays open; deltas accumulate incorrectly.
- **Letting status drift**: future-you can't recover. Update status every session, terse but accurate.

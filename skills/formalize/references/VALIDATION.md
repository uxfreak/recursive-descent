# VALIDATION — The Four-Check Protocol

> Reference for [Recursive Descent Formalization](../SKILL.md). What
> "authoritatively complete" means and how to verify it mechanically.

## The criterion

A formalization is **authoritatively complete** when:

> Every claim has been verified against the current source.
> Every public symbol is either documented or explicitly deferred per the granularity policy.
> All cross-references are valid.
> The narrative is internally consistent end-to-end.

This is a **declarable criterion** that other people can re-verify. It's not "I think it's done"; it's "the protocol passed."

## The four checks

### Check 1 — Cited-Claim Audit

**What it does**: extracts every `--ref FILE:LINES` from `.wan/notes.json` and every cited line range in formalization markdown. Verifies each citation resolves.

**When it fails**:
- File doesn't exist (typo, file moved, file deleted upstream)
- Line range exceeds file length (stale citation against newer source)
- Path is malformed (brace-expansion shorthand, etc.)

**Implementation**: `validation/validate-refs.sh` (template provided below)

**Eager prevention**: configure `validators.ref` in `.wan/config.json` to refuse bad refs at insertion time (see `wan-cli` docs). The audit becomes a backstop rather than a discovery mechanism.

### Check 2 — Coverage Matrix

**What it does**: enumerates every public symbol (`pub fn`, `pub struct`, `pub enum`, `pub trait`) in the codebase's source. Classifies each as:
- **documented** — name appears in formalization markdown
- **explicitly deferred** — mentioned in the "What's Deferred" section of `00-overview.md` or `INSIGHTS.md §9`
- **gap** — neither

**When it surfaces gaps**:
- Real coverage gap: substantial function/struct that the formalization missed
- False gap: stdlib enumeration that's categorically deferred but the validator can't pattern-match
- Pattern-match miss: e.g., `*Elem` types should be deferred but aren't recognized

**Honest read**: a typst-sized codebase ends with ~38% documented + 43% explicitly deferred + 19% categorically below-granularity. The "gap" rate is meaningful only if the remaining symbols are substantive (load-bearing functions, key data types) not enumerations (stdlib element types, accessor methods).

**Implementation**: `validation/validate-coverage.sh` (template below). Customize the deferral patterns for your codebase.

### Check 3 — Cross-Reference Consistency

**What it does**:
- Every `[text](path)` in formalization markdown must resolve.
- Every `wan link` target note must exist.

**When it fails**:
- Off-by-one in `../` paths (linking from a 3-level-deep file to a 2-level-deep file)
- Renamed file but the link wasn't updated
- Note ID typo in `wan link add`

**Implementation**: `validation/validate-links.sh` (template below).

**Eager prevention**: `validators.markdownRoot` in `.wan/config.json` makes `wan doctor` scan the formalization tree at every doctor run.

### Check 4 — Polish Pass (human-judgment)

**What it does**: read all formalization files top-to-bottom *as a reader*, not as the writer.

**Catch**:
- Tone / header style inconsistency (one file uses "Phase 3"; another uses "Step 3")
- Terminology drift (one file says "Adjustability"; another says "adjustability values")
- Forward references that turned out wrong
- Missing forward pointers (an L1 file mentions a sub-system that's now drilled at L3 but doesn't link)
- Redundancy that should be folded

**Time**: 30-60 minutes for a typst-sized formalization (26 markdown files, ~7000 lines). Skim, don't deep-read; you wrote it, you're checking flow not content.

## When all four pass

Write `COMPLETION.md` declaring authoritative completion. Sample structure (see typst's `COMPLETION.md` for full reference):

```markdown
# COMPLETION REPORT

## The Criterion (declared upfront)
<paste the four-line criterion>

## Validation Results

### ✅ Check 1 — Cited-Claim Audit
<paste the report>

### ✅ Check 2 — Coverage Matrix
<paste the report + interpretation>

### ✅ Check 3 — Cross-Reference Consistency
<paste the report>

### ✅ Check 4 — Polish Pass
<bullet list of what was checked>

## What Got Caught and Fixed
<table of issues caught by the validation that escaped manual review>

## Final Inventory
<counts: sessions, commits, sources, notes, tasks, links, files>

## How to Re-Validate
<the bash commands>

## The Honest Statement
<one paragraph claiming completion with evidence>
```

## Validation script templates

Place these in `validation/` of your workspace, customize the codebase-specific bits.

### `validate-refs.sh`

Validates every `--ref FILE:LINES` resolves. Customize `TARGET_ROOT` and the path-stripping logic.

```bash
#!/usr/bin/env bash
set -uo pipefail
WORKSPACE="$(cd "$(dirname "$0")/.." && pwd)"
TARGET_ROOT="<absolute-path-to-target-codebase>"
REPORT="$WORKSPACE/validation/refs-report.txt"
ok=0; bad=0
> "$REPORT"

extract_refs() {
  grep -roh -E '<codebase-prefix>/[a-z0-9_/-]+\.<ext>:[0-9]+(-[0-9]+)?(,[0-9]+(-[0-9]+)?)*' \
       "$WORKSPACE/formalization" 2>/dev/null | sort -u
  python3 -c "
import json
with open('$WORKSPACE/.wan/notes.json') as f: d = json.load(f)
for n in d.get('notes', []):
    for r in n.get('refs') or []:
        if r.get('lines'): print(f\"{r['path']}:{r['lines']}\")
        else: print(r['path'])
" | sort -u
}

verify() {
  local ref="$1" path lines
  if [[ "$ref" == *:* ]]; then path="${ref%:*}"; lines="${ref##*:}"
  else path="$ref"; lines=""; fi
  path="${path#<codebase-prefix>/}"
  case "$path" in ""|<codebase-prefix>) return 0 ;; esac
  local full="$TARGET_ROOT/$path"
  if [[ "$path" != *.<ext> ]]; then
    [[ -d "$full" || -f "$full" ]] && return 0
    echo "MISSING_PATH  $ref" >> "$REPORT"; return 1
  fi
  [[ -f "$full" ]] || { echo "MISSING_FILE  $ref" >> "$REPORT"; return 1; }
  [[ -z "$lines" ]] && return 0
  local file_len max
  file_len=$(wc -l < "$full")
  max=$(echo "$lines" | tr ',' '\n' | tr '-' '\n' | sort -n | tail -1)
  (( max > file_len + 5 )) && {
    echo "OUT_OF_BOUNDS  $ref (file has $file_len, ref cites $max)" >> "$REPORT"
    return 2
  }
  return 0
}

while IFS= read -r ref; do
  [[ -z "$ref" ]] && continue
  verify "$ref" && ok=$((ok+1)) || bad=$((bad+1))
done < <(extract_refs)

echo "Verified: $ok / $((ok+bad))"
[[ $bad -eq 0 ]]
```

### `validate-coverage.sh`

Enumerates public symbols and classifies. Customize crate list, deferral patterns, and the source-tree path.

```bash
#!/usr/bin/env bash
set -uo pipefail
WORKSPACE="$(cd "$(dirname "$0")/.." && pwd)"
TARGET_ROOT="<absolute-path-to-target-codebase>"
HAYSTACK="$WORKSPACE/validation/.haystack"
GAPS="$WORKSPACE/validation/coverage-gaps.txt"

CRATES=( <list-the-crates-or-modules-here> )

find "$WORKSPACE/formalization" -name "*.md" -print0 | xargs -0 cat > "$HAYSTACK"
[[ -f "$WORKSPACE/INSIGHTS.md" ]] && cat "$WORKSPACE/INSIGHTS.md" >> "$HAYSTACK"

> "$GAPS"
total=0; doc=0; def=0; gap=0
for crate in "${CRATES[@]}"; do
  src="$TARGET_ROOT/<crate-path>/$crate/src"
  [[ -d "$src" ]] || continue
  while IFS= read -r line; do
    sym=$(echo "$line" | sed -E 's/.*pub (fn|struct|enum|trait|type|const) +([A-Za-z_][A-Za-z0-9_]*).*/\2/')
    [[ -n "$sym" && "$sym" != "$line" ]] || continue
    case "$sym" in
      new|get|len|is_empty|default|from|into|clone|drop|fmt|hash|eq) continue ;;
    esac
    total=$((total+1))
    if grep -q -F -w "$sym" "$HAYSTACK"; then doc=$((doc+1)); continue; fi
    # Categorical deferrals — customize these patterns per codebase
    case "$sym" in
      *Elem|*Item|*Style|from_*|to_*|is_*|as_*|with_*|set_*|get_*) def=$((def+1)); continue ;;
      [A-Z][A-Z]*) def=$((def+1)); continue ;;
    esac
    gap=$((gap+1))
    echo "$crate  $sym" >> "$GAPS"
  done < <(grep -rh -E "^pub (fn|struct|enum|trait|type|const) " "$src" | sort -u)
done

rm -f "$HAYSTACK"
echo "Total: $total | Documented: $doc | Deferred: $def | Gaps: $gap"
echo "Gap symbols in: $GAPS"
```

### `validate-links.sh`

Verifies all markdown links + wan note links resolve.

```bash
#!/usr/bin/env bash
set -uo pipefail
WORKSPACE="$(cd "$(dirname "$0")/.." && pwd)"
ok_md=0; bad_md=0
for md in $(find "$WORKSPACE/formalization" -name "*.md") "$WORKSPACE/INSIGHTS.md"; do
  [[ -f "$md" ]] || continue
  base="$(dirname "$md")"
  while IFS= read -r path; do
    file_part="${path%%#*}"; [[ -z "$file_part" ]] && continue
    case "$file_part" in http*|mailto:*) continue ;; esac
    if [[ "$file_part" == /* ]]; then target="$file_part"
    else target="$base/$file_part"; fi
    if [[ -f "$target" || -d "$target" ]]; then ok_md=$((ok_md+1))
    else bad_md=$((bad_md+1)); echo "BROKEN $(basename "$md"): $path"; fi
  done < <(grep -oE '\]\([^)]+\)' "$md" 2>/dev/null | sed -E 's/^\]\(([^)]+)\)$/\1/' | sort -u)
done

ok_wan=$(python3 -c "
import json
with open('$WORKSPACE/.wan/notes.json') as f: d = json.load(f)
ids = {n['id'] for n in d.get('notes', [])}
print(sum(1 for n in d.get('notes', []) for l in (n.get('links') or []) if l.get('to') in ids))
")
bad_wan=$(python3 -c "
import json
with open('$WORKSPACE/.wan/notes.json') as f: d = json.load(f)
ids = {n['id'] for n in d.get('notes', [])}
print(sum(1 for n in d.get('notes', []) for l in (n.get('links') or []) if l.get('to') not in ids))
")
echo "Markdown links: $ok_md ok / $bad_md broken"
echo "Wan links: $ok_wan ok / $bad_wan broken"
[[ $bad_md -eq 0 && $bad_wan -eq 0 ]]
```

## When to run validation

- **Before each session-end commit** (eager validators are doing this for you)
- **Before declaring completion** (the four-check protocol)
- **After typst (or your target) updates upstream** (catch upstream drift)
- **Weekly during the active formalization** (catch slow rot)

## When validation finds issues

- **0-3 issues**: fix in place, re-run, no protocol break.
- **4-10 issues**: pause, fix all, re-run, document the findings (which patterns escaped which checks).
- **>10 issues**: there's a systemic problem. Either the methodology drifted (add validation patterns) or the target codebase changed substantially (re-pin the commit and re-validate).

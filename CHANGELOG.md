# Changelog

All notable changes to `recursive-descent` will be documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — Initial release

### Added

- **`recursive-descent:formalize` skill** — orchestrates the four-level recursive descent formalization methodology
- **Reference documentation**:
  - `DOCTRINE.md` — three principles + five recurring patterns
  - `LEVELS.md` — L0/L1/L2/L3 definitions + fork criteria
  - `SESSION-PROTOCOL.md` — six-command core loop
  - `VALIDATION.md` — four-check completion protocol + script templates
  - `CASE-STUDY.md` — typst formalization as proof-of-work
- **`wan` CLI** — vendored TypeScript source (`wan-src/`) with a bash wrapper (`bin/wan`) that runs it via `bun` on first invocation. Can be pre-compiled into `bin/wan-bin` for cached startup.
- **Plugin manifest** (`.claude-plugin/plugin.json`) — v0.1.0

### Methodology features

- Per-session workflow: `wan resume` → `task focus` → `session start` → work → `note add --ref` → `link add` → `status set` → `session end` → git commit
- First-class provenance: every claim carries a `--ref FILE:LINES` citation
- Pluggable per-project validators (via `.wan/config.json`) that refuse bad citations at insertion time
- Four-check validation protocol: cited-claim audit + coverage matrix + cross-reference consistency + polish pass
- Authoritative completion criterion: declarable, re-verifiable

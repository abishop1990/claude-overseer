# claude-overseer

Autonomous development overseer for [Claude Code](https://claude.ai/code). Triages your codebase, picks the highest-value action, executes it, commits it, and loops.

## What it does

```
1. Triage    [WORKING]     — build, test, git state (parallel, <2 min)
2. Analyze   [WORKING]     — 2-3 scans max, rotated per cycle
3. Plan      [PRESENTING]  — concrete plan for the top finding
4. Execute   [WORKING]     — do the work, verify build+tests pass
5. Commit    [WORKING]     — structured [overseer/cycle-N] commit, loop back
```

**Core principle: Act, Don't Catalog.** 60% of each cycle is execution, not analysis. Every cycle should produce a commit.

## Features

- **Language agnostic** — auto-detects Rust, Node/TS, Python, Go, Java, .NET
- **CLAUDE.md aware** — uses project-specific build/test commands
- **Smart rotation** — scans 2-3 dimensions per cycle, not all 6 every time
- **Skips analysis when unnecessary** — build broken? just fix it. Open backlog? execute it.
- **Attention states** — `[WORKING]` / `[PRESENTING]` / `[BLOCKED]` so you know when you're needed
- **Notifications** — OS toast, Slack, ntfy.sh, or webhook when input is required
- **Autopilot** — say "autopilot" and it picks + executes the best action with guardrails
- **Interruptible** — message at any time; it handles your request then resumes
- **Cost optimized** — Haiku for searches/builds, Sonnet for code, Opus only for coordination

## Install

**As a plugin:**
```bash
/plugin marketplace add abishop1990/claude-overseer
/plugin install overseer
```

**As a project skill:**
```bash
mkdir -p .claude/skills/overseer
curl -o .claude/skills/overseer/SKILL.md \
  https://raw.githubusercontent.com/abishop1990/claude-overseer/main/skills/overseer/SKILL.md
```

**As a personal skill (all projects):**
```bash
mkdir -p ~/.claude/skills/overseer
curl -o ~/.claude/skills/overseer/SKILL.md \
  https://raw.githubusercontent.com/abishop1990/claude-overseer/main/skills/overseer/SKILL.md
```

## Usage

```bash
/overseer              # Full triage
/overseer tests        # Test coverage only
/overseer bugs         # Bug detection + security
/overseer perf         # Performance optimization
/overseer security     # Security + dep audit
/overseer roadmap      # Plan next features
/overseer quality      # Code quality + dead code + doc drift
/overseer deps         # Dependency audit
/overseer smoke        # Functional smoke test
```

During a session: "autopilot", "skip", "stop", or interrupt with any message.

## How it stays efficient

| Optimization | How |
|-------------|-----|
| Scan rotation | Max 2-3 analysis dimensions per cycle instead of all 6 |
| Skip on failure | Build broken → fix it immediately, no analysis |
| Skip on backlog | Open findings from last cycle → execute, don't re-scan |
| Targeted re-scan | After a fix, only re-check affected dimensions |
| Model delegation | Haiku for searches/builds, Sonnet for code, Opus for decisions |
| No subagents for small projects | Inline Grep/Glob for <20 files |
| Git log = audit trail | No separate iteration log file; `[overseer/cycle-N]` commits track everything |

## License

MIT

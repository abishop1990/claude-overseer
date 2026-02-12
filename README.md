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
- **Autonomy tiers** — cautious, balanced, or aggressive; you choose how much control to keep
- **Smart rotation** — scans 2-3 dimensions per cycle, not all 6 every time
- **Skips analysis when unnecessary** — build broken? just fix it. Open backlog? execute it.
- **User queue** — say "add [feature]" anytime to steer what gets worked on next
- **Attention states** — `[WORKING]` / `[PRESENTING]` / `[BLOCKED]` so you know when you're needed
- **Notifications** — OS toast, Slack, ntfy.sh, or webhook when input is required
- **Adaptive learning** — tracks your approve/skip patterns to auto-adjust over time
- **Session resume** — picks up where the last session left off
- **Stuck detection** — defers findings after 3 failed attempts, asks for guidance
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

During a session:
- **"autopilot"** — pick + execute best action with guardrails
- **"skip"** — skip to next finding
- **"add [description]"** — queue a feature/task for next cycle
- **"show queue"** — see pending user-queued items
- **"stop"** — stop and show session summary
- Or interrupt with any message

## Autonomy tiers

| Tier | When things auto-execute | When you're asked |
|------|-------------------------|-------------------|
| **Cautious** | Never | Always |
| **Balanced** (default) | Build/test fixes, dead imports, lint | Features, API changes, architecture |
| **Aggressive** | Everything except guardrail items | API changes, auth/security, file deletion |

Guardrail items (always ask): public API changes, file deletion, auth/security code, ambiguous test failures, CI/CD changes.

## How it stays efficient

| Optimization | How |
|-------------|-----|
| Scan rotation | Max 2-3 analysis dimensions per cycle instead of all 6 |
| Skip on failure | Build broken → fix it immediately, no analysis |
| Skip on backlog | Open findings or user queue → execute, don't re-scan |
| Diff-scoped re-analysis | After a commit, prioritize changed files over full rescan |
| Pattern batching | 3+ similar issues → one systematic fix, not N separate passes |
| Adaptive learning | Auto-promotes categories you always approve, deprioritizes skipped ones |
| Model delegation | Haiku for searches/builds, Sonnet for code, Opus for decisions |
| No subagents for small projects | Inline Grep/Glob for <20 files |
| Git log = audit trail | No separate log file; `[overseer/cycle-N]` commits track everything |

## License

MIT

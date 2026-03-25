# claude-overseer

Autonomous development overseer for [Claude Code](https://claude.ai/code). Triages your codebase, picks the highest-value action, executes it in an isolated worktree, opens a PR, reviews it with a code review subagent, fixes concerns, and loops.

## What it does

```
1. Triage    [WORKING]     — build, test, git state (parallel, <2 min)
2. Analyze   [WORKING]     — 2-3 scans max, knowledge-driven, rotated per cycle
3. Plan      [PRESENTING]  — concrete plan for the top finding
4. Execute   [WORKING]     — do the work in an isolated worktree
5. PR+Review [WORKING]     — open PR, code review subagent reviews it, fix concerns
6. Merge     [WORKING]     — merge after approval, clean up, loop back
```

**Core principle: Act, Don't Catalog.** 50% of each cycle is execution, not analysis. Every cycle should produce a reviewed PR.

## Features

- **Language agnostic** — auto-detects Rust, Node/TS, Python, Go, Java, .NET
- **CLAUDE.md aware** — uses project-specific build/test commands
- **Worktree isolation** — each cycle works in its own git worktree, keeping main clean
- **PR workflow** — every change goes through a pull request for traceability
- **Automated code review** — a separate subagent reviews every PR before merge
- **Review fix loops** — review concerns get addressed automatically (up to 2 rounds)
- **Knowledge base** — persistent codebase understanding survives context drops and sessions
- **Intent analysis** — understands what code is *trying to do*, not just what patterns exist
- **Feature discovery** — identifies missing capabilities from architecture and intent analysis
- **Autonomy tiers** — cautious, balanced, or aggressive; you choose how much control to keep
- **Smart rotation** — scans 2-3 dimensions per cycle, not all 6 every time
- **Knowledge-first scanning** — only analyzes files that changed or aren't yet understood
- **Skips analysis when unnecessary** — build broken? just fix it. Open backlog? execute it.
- **User queue** — say "add [feature]" anytime to steer what gets worked on next
- **Attention states** — `[WORKING]` / `[PRESENTING]` / `[BLOCKED]` so you know when you're needed
- **Notifications** — OS toast, Slack, ntfy.sh, or webhook when input is required
- **Adaptive learning** — tracks your approve/skip patterns to auto-adjust over time
- **Session resume** — picks up where the last session left off via state + knowledge base
- **Stuck detection** — defers findings after 3 failed attempts, asks for guidance
- **Interruptible** — message at any time; it handles your request then resumes
- **Cost optimized** — Haiku for scans/builds, Sonnet only when reasoning needed, inline for small fixes

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
/overseer suggest      # "What should we build next?"
/overseer stabilize    # Zero-tolerance: fix all warnings, flaky tests
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
| Knowledge base | Persistent codebase understanding — never re-scans what's already known and unchanged |
| Knowledge-first scanning | Check knowledge base before scanning; only analyze new/changed/unknown files |
| Worktree isolation | Each cycle in its own worktree — main stays clean, no merge conflicts |
| PR + code review | Every change reviewed by independent subagent before merge |
| Scan rotation | Max 2-3 analysis dimensions per cycle instead of all 6 |
| Skip on failure | Build broken → fix it immediately, no analysis |
| Skip on backlog | Open findings or user queue → execute, don't re-scan |
| Diff-scoped re-analysis | After a commit, prioritize changed files over full rescan |
| Pattern batching | 3+ similar issues → one systematic fix, not N separate passes |
| Adaptive learning | Auto-promotes categories you always approve, deprioritizes skipped ones |
| Haiku-first analysis | Haiku scans everything; Sonnet only escalates for control/data flow |
| Smart delegation | ≤2 file fixes inline, no subagent overhead; single subagent up to 5 files |
| Intent analysis | Understands purpose to find real issues, not just pattern matches |
| Context management | State file + knowledge base = durable memory across context drops |

## The cycle in detail

```
┌─────────────────────────────────────────────────────┐
│  1. TRIAGE — build, test, git state                 │
│     Knowledge base loaded, stale components flagged  │
├──────────────┬──────────────────────────────────────┤
│ Build fails? │ → Fix it, skip to Phase 4            │
│ Tests fail?  │ → Fix it, skip to Phase 4            │
│ Backlog?     │ → Top item to Phase 3                │
│ All green    │ → Phase 2                            │
├──────────────┴──────────────────────────────────────┤
│  2. ANALYZE — 2-3 dimensions, knowledge-first       │
│     Intent analysis: what is the code trying to do? │
│     Update knowledge base with new understanding    │
├─────────────────────────────────────────────────────┤
│  3. PLAN — present finding + approach               │
│     (skipped for high-confidence auto-executes)     │
├─────────────────────────────────────────────────────┤
│  4. EXECUTE — work in isolated worktree             │
│     Build + test in worktree to verify              │
├─────────────────────────────────────────────────────┤
│  5. PR + REVIEW — open PR, code review subagent     │
│     Fix concerns → re-review (up to 2 rounds)       │
├─────────────────────────────────────────────────────┤
│  6. MERGE — squash merge, clean up worktree         │
│     Update state + knowledge base → loop to 1       │
└─────────────────────────────────────────────────────┘
```

## License

MIT

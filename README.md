# claude-overseer

An autonomous development overseer skill for [Claude Code](https://claude.ai/code). Continuously analyzes any codebase to find bugs, test gaps, security issues, performance wins, and the next best thing to build.

## What it does

When you invoke `/overseer`, Claude enters a continuous development loop:

```
Phase 0: Situational Awareness   — git state, dependency audit, open issues/PRs
Phase 1: Health Check             — build, test, lint (all in parallel)
Phase 2: Deep Analysis            — test gaps, code quality, security, perf, dead code, doc drift
Phase 3: Roadmap Review           — parse roadmap artifacts, reason about dependencies
Phase 3b: Functional Smoke Test   — start the server, hit routes, check responses
Phase 4: Findings Report          — prioritized report with concrete file:line references
Phase 5: Execute                  — implement the chosen fix/feature/improvement
Phase 6: Checkpoint               — commit the work, loop back to Phase 0
```

Each cycle produces a prioritized report and asks what to work on next. Say **"autopilot"** and it picks the top recommendation itself.

## Features

- **Language agnostic** — auto-detects Rust, Node/TS, Python, Go, Java, .NET from project files
- **CLAUDE.md aware** — respects project-specific build commands, conventions, and contracts
- **Cost optimized** — uses Haiku for searches and builds, Sonnet for fixes, Opus only for coordination
- **Persistent state** — tracks findings, warning trends, and session progress across context compactions
- **Focus modes** — `/overseer tests`, `/overseer security`, `/overseer roadmap`, etc.
- **Autopilot guardrails** — won't modify public APIs, delete files, or touch auth code without asking
- **Commit checkpoints** — offers to commit after each piece of work to prevent diff accumulation

## Install

### As a Claude Code plugin (shareable)

```bash
# In Claude Code, add the plugin marketplace:
/plugin marketplace add abishop1990/claude-overseer

# Then install the overseer skill:
/plugin install overseer
```

### As a project-local skill (single project)

Copy the skill into your project:

```bash
mkdir -p .claude/skills/overseer
curl -o .claude/skills/overseer/SKILL.md \
  https://raw.githubusercontent.com/abishop1990/claude-overseer/main/skills/overseer/SKILL.md
```

### As a personal skill (all your projects)

```bash
mkdir -p ~/.claude/skills/overseer
curl -o ~/.claude/skills/overseer/SKILL.md \
  https://raw.githubusercontent.com/abishop1990/claude-overseer/main/skills/overseer/SKILL.md
```

## Usage

```bash
# Full analysis across all dimensions
/overseer

# Focus on a specific area
/overseer tests      # Test coverage gaps
/overseer bugs       # Bug detection + security scan
/overseer perf       # Performance optimization
/overseer security   # Security hardening + dependency audit
/overseer roadmap    # Plan next features from roadmap
/overseer quality    # Code quality + dead code + doc drift
/overseer deps       # Dependency health and updates
/overseer smoke      # Functional smoke testing
```

### During a session

| Command | Effect |
|---------|--------|
| Pick from the report | Work on the selected item |
| "next cycle" | Re-analyze without executing anything |
| "auto" / "autopilot" | Let it pick and execute the top recommendation |
| "stop" / "exit" | End with a session summary |

## How it saves you money

The overseer is designed for long-running sessions. Without optimization, it would burn expensive Opus tokens on work that smaller models handle fine. The cost strategy:

| Work | Model | Why |
|------|-------|-----|
| Coordination, reports, prioritization | Opus | Needs judgment |
| Build/test/lint commands | Haiku subagents | Just running commands |
| Code searching (grep, glob) | Haiku subagents | Pattern matching |
| Writing tests, simple bug fixes | Sonnet subagents | Competent but cheaper |
| Feature architecture | Opus | Deep reasoning |
| Code review | Sonnet (CodeRabbit) | Structured review |

A typical analysis-only cycle costs ~3-5 Opus messages + 8-12 Haiku subagent calls.

## Session persistence

The overseer writes `.claude/overseer-state.json` after each cycle. This file:

- Tracks warning count trends across cycles ("SSR warnings: 12 -> 8")
- Remembers findings you deferred (won't re-report them)
- Caches project detection (skips re-detection on subsequent cycles)
- Preserves session progress through context compaction

## Autopilot guardrails

In autopilot mode, the overseer still pauses for confirmation when:

- Modifying a public API or exported interface
- Deleting files or removing features
- Touching security-sensitive code (auth, encryption)
- Ambiguous test failures (fix code vs update test?)
- CI/CD pipeline changes

Everything else (adding tests, quality fixes, doc updates) proceeds automatically.

## License

MIT

# claude-overseer

An autonomous development overseer skill for [Claude Code](https://claude.ai/code). Continuously analyzes any codebase to find bugs, test gaps, security issues, performance wins, and the next best thing to build.

## What it does

When you invoke `/overseer`, Claude enters a continuous development loop:

```
Phase 0:  Situational Awareness  [WORKING]     — git state, dep audit, open issues/PRs
Phase 1:  Health Check           [WORKING]     — build, test, lint (all in parallel)
Phase 2:  Deep Analysis          [WORKING]     — test gaps, quality, security, perf, dead code, docs
Phase 3:  Roadmap Review         [WORKING]     — parse roadmap artifacts, reason about dependencies
Phase 3b: Smoke Test             [WORKING]     — start server, hit routes, check responses
Phase 4:  Findings Report        [PRESENTING]  — prioritized report with file:line references
Phase 4b: Brainstorm & Plan      [PRESENTING]  — concrete plans with approaches, risks, alternatives
         → user decision          [BLOCKED]     — notification sent, pick an action
Phase 5:  Execute                [WORKING]     — implement the chosen fix/feature/improvement
Phase 6:  Checkpoint             [WORKING]     — structured commit, iteration log, loop back
```

Each cycle produces a prioritized report, brainstorms concrete plans, and asks what to work on next. Say **"autopilot"** and it picks the top recommendation itself.

## Features

- **Language agnostic** — auto-detects Rust, Node/TS, Python, Go, Java, .NET from project files
- **CLAUDE.md aware** — respects project-specific build commands, conventions, and contracts
- **Attention states** — clear `[WORKING]` / `[PRESENTING]` / `[BLOCKED]` indicators so you know when you're needed
- **Notifications** — OS toast, Slack webhook, ntfy.sh (mobile push), or generic webhook when input is needed
- **Brainstorming** — doesn't just list problems; proposes 2-3 concrete approaches with effort, risks, and scope variants
- **Cost optimized** — Haiku for searches/builds, Sonnet for fixes, Opus only for coordination
- **Persistent state** — tracks findings, warning trends, and progress across context compactions
- **Iteration log** — `.claude/overseer-log.md` records every cycle for auditability and team visibility
- **Structured commits** — `[overseer/cycle-N]` prefixed, machine-filterable commit history
- **Time-boxed sessions** — "run for 15 minutes" with graceful wind-down
- **Focus modes** — `/overseer tests`, `/overseer security`, `/overseer roadmap`, etc.
- **Autopilot guardrails** — won't modify public APIs, delete files, or touch auth code without asking

## Install

### As a Claude Code plugin (shareable)

```bash
# In Claude Code, add the plugin marketplace:
/plugin marketplace add abishop1990/claude-overseer

# Then install the overseer skill:
/plugin install overseer
```

### As a project-local skill (single project)

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
| "brainstorm more" | Generate different options for the current finding |
| "stop" / "exit" | End with a session summary |

### Notifications

On first run, the overseer asks how you'd like to be notified when it needs input:

| Channel | How it works | Setup |
|---------|-------------|-------|
| **OS** (default) | System toast notification + terminal bell | Zero config |
| **Slack** | Posts to a Slack channel via webhook | Provide webhook URL |
| **ntfy** | Push notification to your phone via [ntfy.sh](https://ntfy.sh) | Provide topic name |
| **Webhook** | POST JSON to any URL | Provide endpoint |
| **None** | No notification — watch the terminal | — |

Preferences are saved to `.claude/overseer-state.json` and persist across sessions.

## How it saves you money

The overseer is designed for long-running sessions. Without optimization, it would burn expensive Opus tokens on work that smaller models handle fine:

| Work | Model | Why |
|------|-------|-----|
| Coordination, reports, prioritization | Opus | Needs judgment |
| Build/test/lint commands | Haiku subagents | Just running commands |
| Code searching (grep, glob) | Haiku subagents | Pattern matching |
| Writing tests, simple bug fixes | Sonnet subagents | Competent but cheaper |
| Feature architecture | Opus | Deep reasoning |
| Code review | Sonnet (CodeRabbit) | Structured review |

A typical analysis-only cycle costs ~3-5 Opus messages + 8-12 Haiku subagent calls.

## Artifacts

The overseer creates these files in your project's `.claude/` directory:

| File | Purpose |
|------|---------|
| `overseer-state.json` | Session state, warning baselines, notification config |
| `overseer-log.md` | Human-readable history of every cycle and action taken |
| `overseer-plan-cycle-N.md` | Detailed plan for cycle N (approaches, risks, files) |

Add these to `.gitignore` if you don't want them tracked, or commit them for team visibility.

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

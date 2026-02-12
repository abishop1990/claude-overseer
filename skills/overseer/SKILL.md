---
name: overseer
description: >
  Autonomous project overseer that continuously analyzes any codebase to find the next
  best thing to work on. Cycles through bug detection, test coverage, performance,
  roadmap planning, and code quality. Works with any language/framework. Use when you
  want a guided development session.
argument-hint: "[focus-area]"
---

# Overseer Mode

You are now in **Overseer Mode** — an autonomous project development loop. Your job is to
analyze the current state of the project, identify the highest-value next action, present
your findings, and — with user approval — execute or plan the work.

**This skill is language and framework agnostic.** Adapt all analysis to whatever project
you're working in. Detect the stack from project files and CLAUDE.md before running any
commands.

## Optional Focus

If the user provided an argument, treat it as a focus area hint:
- `$ARGUMENTS`

If empty, run the full analysis across all dimensions.

## Attention States & Notifications

The overseer has three attention states. **Always display the current state** at the start
of each message so the user can glance and know whether they're needed.

### States

| State | Indicator | Meaning |
|-------|-----------|---------|
| `WORKING` | `[WORKING - no input needed]` | Autonomous execution in progress. User can walk away. |
| `PRESENTING` | `[PRESENTING - review when ready]` | Report or plan is ready for review. Non-urgent. |
| `BLOCKED` | `[BLOCKED - input required]` | Cannot continue without a user decision. |

**Rules:**
- Enter `WORKING` at the start of Phases 0, 1, 2, 3, 3b, 5 (execution), and 6 (commit)
- Enter `PRESENTING` at the start of Phase 4 (report) and Phase 4b (brainstorm/plan)
- Enter `BLOCKED` only when `AskUserQuestion` is used — a real decision is needed
- In autopilot mode, `BLOCKED` should be rare (only for guardrail items)

### Notification Channels

When transitioning to `BLOCKED` (or `PRESENTING` if the user configured eager notifications),
alert the user through their configured channel. Check `.claude/overseer-state.json` for
the `notifications` config.

**Configuration** (set once at session start if not already configured):

```json
{
  "notifications": {
    "channel": "os|slack|webhook|none",
    "on_blocked": true,
    "on_presenting": false,
    "on_cycle_complete": false,
    "slack_webhook": "https://hooks.slack.com/services/...",
    "webhook_url": "https://your-endpoint.com/notify",
    "ntfy_topic": "overseer-alerts"
  }
}
```

**Channel implementations:**

| Channel | How to notify | Setup |
|---------|--------------|-------|
| `os` | Terminal bell (`\a`) + system notification via `osascript` (macOS), `notify-send` (Linux), or `msg` / PowerShell toast (Windows) | Zero config, works everywhere |
| `slack` | POST to Slack webhook URL with a short JSON message | User provides `slack_webhook` |
| `webhook` | POST to any URL with `{"state": "BLOCKED", "message": "...", "cycle": N}` | User provides `webhook_url` |
| `ntfy` | POST to ntfy.sh topic — free, works on mobile | User provides `ntfy_topic` |
| `none` | No notification — user watches the terminal | Default |

**OS notification commands (use in Bash):**
```bash
# macOS
osascript -e 'display notification "Overseer needs input" with title "Claude Overseer"'

# Linux
notify-send "Claude Overseer" "Overseer needs input"

# Windows (PowerShell)
powershell -Command "[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Overseer needs input','Claude Overseer')"
# Or simpler toast:
powershell -Command "New-BurntToastNotification -Text 'Claude Overseer', 'Input required — check your terminal'"

# ntfy (cross-platform, free, mobile push)
curl -s -d "Overseer needs your input (Cycle N)" ntfy.sh/YOUR_TOPIC

# Slack webhook
curl -s -X POST -H 'Content-type: application/json' --data '{"text":"Overseer BLOCKED: needs input (Cycle N)"}' YOUR_SLACK_WEBHOOK

# Generic webhook
curl -s -X POST -H 'Content-type: application/json' --data '{"state":"BLOCKED","message":"...","cycle":N}' YOUR_WEBHOOK_URL
```

**First-run setup:** On the very first overseer invocation in a project, if no notification
config exists, ask the user once:

> How should I notify you when I need input?

Options: "OS notification (Recommended)", "Slack webhook", "ntfy.sh (mobile push)", "No notifications — I'll watch"

Save their choice to `overseer-state.json` so it persists.

---

## Session State

At the start of each overseer session, read `.claude/overseer-state.json` if it exists.
This file tracks findings and progress across sessions and context compactions. Structure:

```json
{
  "last_run": "ISO timestamp",
  "cycle_count": 0,
  "project_type": "rust|node|python|go|java|dotnet|other",
  "build_cmd": "detected build command",
  "test_cmd": "detected test command",
  "warning_baseline": {},
  "known_findings": ["finding-id-1", "finding-id-2"],
  "completed_this_session": ["description of work done"],
  "deferred": ["items user explicitly skipped"],
  "files_modified": ["list of files changed this session"],
  "notifications": {
    "channel": "os",
    "on_blocked": true,
    "on_presenting": false,
    "on_cycle_complete": false
  }
}
```

Update this file at the end of each cycle. This lets you:
- Track warning regressions ("3 new warnings since last cycle")
- Avoid re-reporting known findings the user already deferred
- Resume after context compaction or a new session
- Cache project detection so you don't re-detect every cycle
- Persist notification preferences across sessions

---

## The Overseer Loop

Each cycle, run through these phases **in order**. After completing a cycle, ask the user
if they want to continue to another cycle or stop.

---

### Phase 0: Project Detection & Situational Awareness

#### 0a. Detect Project Stack

If `overseer-state.json` has `project_type` cached, use it. Otherwise, detect by checking
for these files (in parallel):

| File | Stack | Build | Test | Lint |
|------|-------|-------|------|------|
| `Cargo.toml` | Rust | `cargo build` | `cargo test` | `cargo clippy` |
| `package.json` | Node/TS | `npm run build` | `npm test` | `npm run lint` |
| `pyproject.toml` / `setup.py` | Python | — | `pytest` | `ruff check .` |
| `go.mod` | Go | `go build ./...` | `go test ./...` | `go vet ./...` |
| `pom.xml` / `build.gradle` | Java | `mvn compile` / `gradle build` | `mvn test` / `gradle test` | — |
| `*.sln` / `*.csproj` | .NET | `dotnet build` | `dotnet test` | — |

**Override with CLAUDE.md:** If the project's CLAUDE.md specifies build/test commands,
always use those instead of defaults. CLAUDE.md is the source of truth.

Also check for:
- **Monorepo indicators**: `pnpm-workspace.yaml`, `lerna.json`, `Cargo.toml [workspace]`
- **CI config**: `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile` — reveals the
  project's own quality gates
- **Docker**: `Dockerfile`, `docker-compose.yml` — deployment context

#### 0b. Git State
Run `git status`, `git log --oneline -5`, and `git diff --stat` to understand:
- **Uncommitted changes** — may be in-progress work. Don't suggest reworking unless broken.
- **Current branch** — main, feature branch, or detached?
- **Recent commits** — what was last done? Context for what's "next."
- **Stale branches** — local branches whose remote is gone (`git branch -vv | grep gone`)

#### 0c. Dependency Health
Run the appropriate audit command:
- Rust: `cargo audit 2>&1`
- Node: `npm audit 2>&1`
- Python: `pip-audit 2>&1` or `safety check 2>&1`
- Go: `govulncheck ./... 2>&1`

If the audit tool isn't installed, note it as a recommendation and skip.

#### 0d. Open Issues & PRs
If the repo is connected to GitHub, use GitHub MCP tools to check:
- Open PRs that need review or are failing CI
- Recent issues filed (last 7 days)
- Any issues assigned to the current user

This provides external context that pure code analysis misses.

---

### Phase 1: Health Check

Run the detected build and test commands **in parallel**:

For a typical project, this means 2-3 parallel Bash subagents (haiku model):
1. Build/compile check
2. Test suite
3. Linter (if available)

For projects with multiple build targets (e.g., Leptos SSR + WASM, Next.js server + client),
check CLAUDE.md for the full set of compile commands and run them all in parallel.

**Report format:**
```
## Health Check
- Build: PASS / FAIL (N warnings, +/-M since last cycle)
- Tests: X passed, Y failed, Z ignored/skipped
- Lint: PASS / FAIL (N issues)
```

Compare warning counts against `overseer-state.json` to detect regressions.

If anything **fails**, that becomes the top priority for this cycle — skip ahead to
Phase 5 with the fix as the recommended action.

---

### Phase 2: Deep Analysis

Run these analyses **in parallel** using Explore/search agents. Adapt patterns to the
detected language.

#### 2a. Test Coverage Gaps
- Find test files/modules and count test functions per source module
- Identify source files with **zero or very few tests** (especially in core logic dirs)
- Look for complex public functions (>30 lines) that lack test coverage
- Check for placeholder markers: `todo!()`, `TODO`, `FIXME`, `unimplemented`, `NotImplementedError`, `throw new Error("not implemented")`

**Language-specific patterns:**

| Language | Test location | Test marker |
|----------|--------------|-------------|
| Rust | `#[cfg(test)]` modules, `tests/` | `#[test]` |
| JS/TS | `*.test.ts`, `*.spec.ts`, `__tests__/` | `it(`, `test(`, `describe(` |
| Python | `test_*.py`, `*_test.py`, `tests/` | `def test_`, `@pytest.mark` |
| Go | `*_test.go` | `func Test` |
| Java | `src/test/`, `*Test.java` | `@Test` |

#### 2b. Code Quality Scan
- **Crash risks**: unhandled errors, bare `unwrap()`/`!` force unwraps, unchecked
  null/undefined access, empty catch blocks
- **Unnecessary complexity**: functions over 80 lines, deeply nested conditionals (>4 levels),
  God objects/files (>500 lines)
- **Debug leftovers**: `println!`, `console.log`, `print(`, `dbg!`, `debugger` in non-test code
- **Suppressed warnings**: `#[allow(...)]`, `// eslint-disable`, `# noqa`, `@SuppressWarnings`

#### 2c. Security Scan
- SQL injection: string interpolation/concatenation in queries instead of parameterized
- XSS: unsanitized user input rendered as HTML
- Hardcoded secrets: API keys, passwords, tokens in source (not env vars)
- Dependency vulnerabilities: cross-reference with Phase 0c audit results
- Input validation: server endpoints/handlers that don't validate input size or format

#### 2d. Performance Opportunities
- N+1 query patterns (DB calls inside loops)
- Large allocations in hot paths (unnecessary collect/copy/clone)
- Blocking operations in async contexts
- Missing caching for repeated expensive computations
- Unindexed DB queries on large tables

#### 2e. Dead Code & Unused Dependencies
- Exported/public functions with zero external call sites
- Unused imports/dependencies
- Feature flags or config options that are never read
- Commented-out code blocks (>5 lines)

#### 2f. Documentation Drift
- If CLAUDE.md exists, compare its documented contracts/APIs against actual code
- Check if README build/install instructions still work
- Look for doc comments describing behavior the code no longer implements
- If there's a changelog, check if recent commits are reflected in it

---

### Phase 3: Roadmap & Planning Review

Look for project planning artifacts in this priority order:

1. **`docs/roadmap.json`** or **`ROADMAP.md`** — structured roadmap
2. **GitHub Issues/Projects** — via MCP tools, check milestones and issue labels
3. **TODO/FIXME comments** — in-code planning markers
4. **CHANGELOG.md** — what's been done recently, implies what's next

For whatever planning artifacts exist:

1. **Current milestone** — what's in progress with incomplete items?
2. **Next up** — incomplete items ordered by likely dependency
3. **Quick wins** — small/self-contained items in any milestone
4. **Known issues** — documented bugs or limitations

**Dependency reasoning:** Before recommending an item, think about ordering:
- Does this item depend on another incomplete item?
- Would building this first create reusable infrastructure for other items?
- Are there items across milestones that share implementation work?

Explicitly state the dependency chain when recommending.

Cross-reference with the codebase:
- Are there partial implementations started but not finished?
- Do git uncommitted changes suggest the user is mid-implementation on something?

---

### Phase 3b: Functional Smoke Test (optional, when no critical issues found)

If Phases 1-3 found no compilation errors or test failures, attempt a functional check:

1. Detect the serve/run command from CLAUDE.md, `package.json` scripts, or conventions
2. Start the server in background (with a 60s startup timeout)
3. Wait for a "listening" or "ready" message in stdout
4. Hit key routes with curl and check for 200 responses:
   - `GET /` — landing/index
   - `GET /health` or `GET /api/health` — health check endpoint (if it exists)
   - A few other routes discovered from the router/routes config
5. Kill the background server
6. Report which routes responded vs failed

**Skip this phase if:**
- Phase 1 had compilation/build failures
- The user said "skip smoke" or focus area doesn't include it
- Required env vars (DB, API keys) aren't set
- The project is a library, not a server

If the smoke test hangs or fails to start, note it and move on — never block the cycle.

---

### Phase 4: Findings Report

Present a prioritized report to the user:

```
# Overseer Report — Cycle N

## Project
- Stack: [detected] | Branch: [current] | Uncommitted: N files
- Last commit: "message" (time ago)

## Critical (fix now)
- [build failures, test failures, security vulnerabilities in deps]

## High Priority
- [bugs found, crash risks in prod code, data integrity risks, failing routes]

## Recommended Next
- [highest-value item with dependency reasoning and effort: small/medium/large]
- [why this one over alternatives]

## Improvements
- [test coverage gaps, code quality, performance wins]

## Quick Wins
- [small fixes that improve quality with minimal effort]

## Documentation
- [any drift detected between docs and code]

## Roadmap Status
- Current milestone: [name] (N/M items done)
- Next incomplete items: [top 3 with dependency order]
- Blocked items: [items waiting on other items]

## Session Progress
- Cycle: N | Items completed this session: [list]
- Warning trend: [+/-N since last cycle]
```

After the report, transition to Phase 4b.

---

### Phase 4b: Brainstorm & Plan `[PRESENTING]`

This phase turns analysis into actionable plans. Don't just list problems — **propose
solutions with enough detail that execution can begin immediately.**

#### For each top-3 recommended action, produce a mini-plan:

```
### Option 1: [action title] — [effort: small/medium/large]

**What:** One-sentence description of the change.

**Why now:** Why this is the highest-value action at this point in the project.

**Approach:**
1. [concrete step — which file to modify, what to add/change]
2. [next step]
3. [verification step]

**Files touched:** `src/foo.rs`, `src/bar.rs`, `tests/foo_test.rs`

**Risks:** [what could go wrong, edge cases, breaking changes]

**Alternative approach:** [if there's a meaningfully different way to do it]
```

#### Creative brainstorming (for roadmap items and features)

When the recommended action is a new feature or roadmap item, go beyond the obvious:

1. **Brainstorm 2-3 approaches** with genuinely different tradeoffs (not just "option A
   and slightly different option A")
2. **Consider the user's likely priorities** based on project history (what kinds of
   changes do recent commits suggest they care about?)
3. **Think about compound value** — does this action unlock or simplify other items?
4. **Propose scope variants** — "minimum viable version" vs "full version" with clear
   cut lines

#### Write the plan to a file

Write the full plan to `.claude/overseer-plan-cycle-N.md` so it persists even if the
conversation context compacts. Reference this file when asking for the user's decision.

This file also serves as a **reviewable artifact** — the user can look at past plans to
understand the overseer's reasoning, share them with teammates, or use them as specs.

#### Present to user `[BLOCKED]`

After writing the plan, present a summary and ask the user to choose using
`AskUserQuestion`:

Options should include:
- The top 2-3 concrete plans (with effort estimates)
- "Autopilot — pick the best one and execute" (if not already in autopilot)
- "Brainstorm more — I want different options"
- "Skip — next cycle without executing"

**Send a notification** (per the user's configured channel) since this is a `BLOCKED` state.

In **autopilot mode**, skip the question and proceed with Option 1 unless it hits a
guardrail. Still write the plan file for auditability.

---

### Phase 5: Execute `[WORKING]`

Based on the user's choice from Phase 4b:

- **Bug fix** → Locate, understand root cause, implement minimal fix, add regression test
- **Feature** → Enter Plan Mode, design the approach, get approval, implement
- **Test coverage** → Write tests directly (no plan needed for pure test additions)
- **Code quality** → Make the fix, verify build + tests pass
- **Roadmap item** → Enter Plan Mode for anything non-trivial
- **Dependency update** → Update manifest, run full test suite, verify nothing broke
- **Documentation** → Update the docs to match reality

The plan from Phase 4b should be detailed enough that execution can reference specific
files, steps, and verification criteria. Don't re-analyze — execute the plan.

After completing the work, re-run the build and test commands to verify nothing broke.

---

### Phase 6: Checkpoint & Iteration Log `[WORKING]`

Every piece of completed work gets an **atomic, structured commit** and a log entry.
This makes the overseer's work reviewable, revertable, and auditable.

#### 6a. Structured Commit

Commit messages follow a machine-parseable format:

```
[overseer/cycle-N] Short description of the change

Category: bugfix|feature|test|quality|docs|perf|security|deps
Cycle: N
Trigger: [what finding prompted this — e.g., "Phase 2b: unwrap() in auth handler"]
Plan: .claude/overseer-plan-cycle-N.md

Detailed description of what was changed and why.

Co-Authored-By: Claude <noreply@anthropic.com>
```

The `[overseer/cycle-N]` prefix lets you filter overseer commits:
```bash
git log --oneline --grep="overseer/cycle"
```

#### 6b. Iteration Log

Append to `.claude/overseer-log.md` after each cycle. This file is the **human-readable
history** of what the overseer did across all sessions.

```markdown
## Cycle N — [ISO timestamp]

**State:** [analysis-only | fix-applied | feature-planned | deferred]
**Branch:** main
**Health:** Build PASS | Tests 52/52 | Warnings 8 (-2)

**Finding:** [what was identified as highest priority]
**Action:** [what was done — or "deferred by user"]
**Commit:** [hash] [message]
**Files:** [list of modified files]
**Duration:** ~[minutes] (N Opus messages, M subagent calls)

---
```

This log serves multiple purposes:
- **Session recovery** — if context compacts, read the log to understand what happened
- **Team visibility** — share the log to show what autonomous work was done
- **Cost tracking** — the duration/message counts help calibrate the cost budget
- **Revert guide** — if something went wrong, the log shows exactly which commit to revert

#### 6c. Commit Decision

In **autopilot mode**: commit automatically (no prompt).

In **interactive mode**: show the diff summary and proposed commit message, then ask:
"Commit this?" with options:
- "Yes, commit and continue" (default)
- "Yes, commit and stop"
- "No, keep changes uncommitted"
- "No, revert changes" (for when the fix made things worse)

**Send a notification** only if the user chose "stop on cycle complete" in their
notification config (`on_cycle_complete: true`).

After checkpoint, return to **Phase 0** for another cycle.

---

## Cycle Management

- After each cycle, briefly summarize what was done and what changed
- Track across cycles: what you've already reported (don't repeat stale findings)
- Write `overseer-state.json` after each cycle for cross-session persistence
- Append to `.claude/overseer-log.md` after each cycle for human-readable history
- Plan files go to `.claude/overseer-plan-cycle-N.md` (one per cycle that has a plan)
- If the user says **"auto"** or **"autopilot"**, pick the top recommendation yourself
  and execute it, then loop — only pause for user input on ambiguous or dangerous decisions
- If the user says **"stop"** or **"exit"**, end overseer mode with a session summary:
  ```
  ## Session Summary
  - Cycles completed: N
  - Items completed: [list with commit hashes]
  - Items deferred: [list]
  - Remaining top priorities: [top 3]
  - Warning trend: started at X, now at Y
  - Total estimated cost: ~N Opus + M Haiku + K Sonnet messages
  - Full log: .claude/overseer-log.md
  ```

### Time-Boxed Sessions

When the user specifies a time limit (e.g., "run for 15 minutes"), track elapsed time:
- Note the start time at session begin
- At the end of each cycle, check remaining time
- If <3 minutes remain, skip to the session summary instead of starting a new cycle
- If mid-execution when time expires, finish the current work and checkpoint — don't
  leave half-done changes uncommitted

## Autopilot Guardrails

When running in autopilot mode, the overseer MUST still pause for user confirmation when:
- A change would modify a **public API** or **exported interface**
- The recommended action involves **deleting files or removing features**
- A roadmap item has **ambiguous requirements** (multiple valid interpretations)
- The change touches **security-sensitive code** (auth, encryption, input validation)
- A test is failing and the fix could be "update the test" vs "fix the code"
- The change would affect **CI/CD pipelines** or deployment configuration

Everything else (adding tests, fixing quality issues, small bug fixes, docs updates)
can proceed without prompting in autopilot.

## Cost & Compute Optimization

The overseer is a long-running loop. Unoptimized, it burns expensive Opus tokens on work
that Haiku or Sonnet can do just as well. **The overseer's main context (Opus) is a
coordinator — it plans, prioritizes, synthesizes, and makes judgment calls. All grunt work
is delegated to cheaper agents.**

### Model Allocation Rules

| Task | Model | Why |
|------|-------|-----|
| **Coordinating the loop, writing reports, making prioritization decisions** | Opus (main) | Requires judgment and synthesis |
| **Running build/test/lint commands** | Haiku via `Bash` subagent | Just executing commands and capturing output |
| **Grep/Glob/Read searches (2a-2f scans)** | Haiku via `Explore` subagent | Pattern matching, no judgment needed |
| **Counting tests, dead code detection** | Haiku via `Explore` subagent | Mechanical analysis |
| **Reading GitHub issues/PRs** | Haiku via `general-purpose` subagent | Data retrieval |
| **Writing tests for existing code** | Sonnet via `general-purpose` subagent | Needs competence but not top-tier reasoning |
| **Bug fixes (straightforward)** | Sonnet via `general-purpose` subagent | Clear problem → clear fix |
| **Feature planning & architecture** | Opus (main) or `Plan` subagent | Requires deep reasoning |
| **Complex multi-file refactors** | Opus (main) | Cross-cutting concerns need full context |
| **Code review of changes** | Sonnet via `coderabbit:code-reviewer` | Structured review doesn't need Opus |

### Delegation Strategy

**Phase 0 (Situational Awareness):**
- Spawn 3-4 Haiku Bash subagents **in parallel**: `git status`, `git log`, `git diff --stat`, dep audit
- Spawn 1 Haiku Explore subagent: detect project stack (check for Cargo.toml, package.json, etc.)
- Spawn 1 Haiku general-purpose subagent: check GitHub issues/PRs (if connected)
- **Opus does:** Read the results, synthesize into context. Total Opus cost: ~1 message.

**Phase 1 (Health Check):**
- Spawn N Haiku Bash subagents **in parallel** for each build/test/lint command
- **Opus does:** Read pass/fail results, compare against baseline. Total Opus cost: ~1 message.

**Phase 2 (Deep Analysis):**
- Spawn **5-6 Explore subagents in parallel** (all Haiku), one per scan category (2a-2f)
- Each subagent gets a focused prompt: "Search for X pattern, report file:line and count"
- **Opus does:** Receive all scan results, deduplicate, prioritize, cross-reference. ~1 message.

**Phase 3 (Roadmap):**
- Spawn 1 Haiku Explore subagent: find and parse roadmap artifacts, TODO/FIXME counts
- **Opus does:** Reason about dependencies, pick recommendations. ~1 message.

**Phase 4 (Report):**
- **Opus only.** This is the synthesis step — no delegation. ~1 message.

**Phase 5 (Execute):**
- **Bug fixes / test writing:** Delegate to Sonnet `general-purpose` subagent with full context
- **Feature work:** Opus coordinates, may use Plan subagent for architecture
- **Quality fixes:** Sonnet subagent for the fix, Haiku Bash for verification

**Phase 6 (Checkpoint):**
- Haiku Bash subagent: `git diff --stat`, `git add`, `git commit`
- **Opus does:** Propose the commit message. ~1 message.

### Parallelization Rules

1. **Never serialize independent operations.** If you need to search 5 patterns, spawn 5
   agents in one message, not 5 sequential messages.
2. **Batch related Bash commands.** Instead of 3 separate Bash calls for `git status`,
   `git log`, and `git diff`, combine into one: `git status && echo "---" && git log --oneline -5 && echo "---" && git diff --stat`
   BUT only when they're going to the same subagent. Separate subagents for separate concerns.
3. **Don't over-delegate.** If a search needs 1 Grep call, just do it directly — spawning
   a subagent for a single Grep is slower than doing it inline.
4. **Cap concurrent subagents at 6-8.** More than that risks hitting rate limits or
   overwhelming the system.
5. **Reuse subagent results.** If Phase 0 already ran `git status`, don't re-run it in Phase 1.
   Pass the data forward.

### Cost Budget per Cycle

A well-optimized cycle should cost roughly:
- **Analysis only (Phases 0-4):** ~3-5 Opus messages + 8-12 Haiku subagents
- **Analysis + small fix (Phases 0-6):** ~5-7 Opus messages + 10-15 Haiku + 1-2 Sonnet subagents
- **Analysis + feature (Phases 0-6):** ~8-12 Opus messages + 10-15 Haiku + 3-5 Sonnet subagents

If a cycle is consuming more than this, you're doing too much in Opus context. Check if
work can be pushed to subagents.

## Analysis Guidelines

- **Be concrete.** Don't say "consider adding tests" — say "`scoring.rs` has 0 tests for
  `calculate_score()` which has 5 match arms and 3 edge cases"
- **Prioritize by impact.** A crash bug > missing test > style nit
- **Reference specific files and lines.** Use `file_path:line_number` format
- **Respect project conventions.** If CLAUDE.md or comments explain why something is the
  way it is, don't flag it as an issue
- **Quick wins first in ties.** If two items have similar priority, recommend the faster one
- **Track the git diff.** Uncommitted changes may be in-progress work
- **Avoid analysis paralysis.** If Phase 2 finds 30 issues, report the top 5-7 and
  mention "N more items in [category]"
- **Celebrate progress.** When a cycle completes work, note what improved ("test count:
  45 → 52, warning count: 12 → 8")
- **Adapt to the project.** A 200-line script needs a lighter touch than a 50k-line app.
  Scale the depth of analysis to the project size.

## Focus Areas (if argument provided)

| Argument | Restrict analysis to |
|----------|---------------------|
| `bugs` | Phase 2b + 2c, find and fix bugs |
| `tests` | Phase 2a, improve test coverage |
| `perf` | Phase 2d, performance optimization |
| `security` | Phase 2c + dependency audit, security hardening |
| `roadmap` | Phase 3, plan next features |
| `quality` | Phase 2b + 2e + 2f, code quality and cleanup |
| `deps` | Phase 0c, dependency health and updates |
| `smoke` | Phase 3b, functional testing |
| (empty) | Full analysis across all dimensions |

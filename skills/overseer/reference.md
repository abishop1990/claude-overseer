# Overseer Reference

Detailed schemas, templates, and examples. Loaded on demand, not at invocation.

## State File Schema

`.claude/overseer-state.json` — read at cycle start, write at cycle end.

```json
{
  "cycle_count": 0,
  "project_type": "rust",
  "build_cmds": ["cargo check"],
  "test_cmd": "cargo test",
  "autonomy": "balanced",
  "has_ci": true,
  "lint_cmd": "npm run lint",
  "last_health": { "status": "green", "lint": "pass", "warnings": 8, "checked_at_cycle": 1 },
  "open_findings": [
    { "id": "f1", "phase": "2b", "summary": "...", "severity": "high", "confidence": "high" }
  ],
  "deferred": [],
  "user_queue": [],
  "patterns": {
    "approved_categories": [],
    "skipped_categories": [],
    "last_dimensions": [],
    "recent_severities": []
  },
  "available_plugins": [],
  "active_worktrees": [],
  "active_prs": [],
  "notifications": { "channel": "os" }
}
```

## Knowledge Base Schema

`.claude/overseer-knowledge.json` — persistent codebase understanding.

```json
{
  "last_updated": "2026-03-24T10:00:00Z",
  "architecture": {
    "summary": "Express.js REST API with PostgreSQL, React frontend",
    "entry_points": ["src/server.ts", "src/app.tsx"],
    "key_patterns": ["repository pattern", "middleware chain"]
  },
  "components": {
    "src/auth/": {
      "purpose": "JWT-based authentication with refresh tokens",
      "contracts": ["validates tokens on every API request"],
      "dependencies": ["src/db/users.ts", "src/config.ts"],
      "last_analyzed": "2026-03-24T10:00:00Z",
      "last_file_hash": "abc123",
      "known_issues": [],
      "intent_notes": "Designed for multi-tenant but only single-tenant implemented"
    }
  },
  "conventions": {
    "error_handling": "Custom AppError class, caught by global middleware",
    "testing": "Jest + supertest for API, React Testing Library for frontend"
  },
  "dependency_map": {},
  "intent_observations": [
    {
      "location": "src/plugins/",
      "observation": "Plugin loader exists but only one plugin registered",
      "potential_work": "Add more plugins or document plugin development",
      "discovered_cycle": 3
    }
  ],
  "scan_cache": {
    "2a": { "last_scanned": "2026-03-24T10:00:00Z", "findings_count": 0, "skip_until_cycle": 5 }
  }
}
```

## PR Body Template

```markdown
## Summary
- What changed and why

## Category
bugfix|feature|test|quality|docs|perf|security|deps

## Trigger
Phase 2b: unwrap() in auth handler

## Verification
- Build: PASS/FAIL
- Tests: X/Y passing
- Targeted tests: [list]

Generated with [Claude Code Overseer](https://github.com/abishop1990/claude-overseer)
```

## Review Prompt

For subagent-based review when no plugin is available:

```
Review this PR diff for the overseer autonomous development loop.

Context: [one-sentence description of what was changed and why]

Review the diff (git diff main...<branch>) for:
- Correctness: does the change do what it claims?
- Security: injection risks, unvalidated input, secret exposure
- Edge cases: error handling gaps, nil/null paths, boundary conditions
- Side effects: unintended changes to behavior elsewhere
- Test coverage: are the changes adequately tested?
- Style: does it match project conventions?

Rate: APPROVE, REQUEST_CHANGES, or COMMENT
If REQUEST_CHANGES: list specific, actionable concerns with file:line references.
Try hard to find real problems — don't rubber-stamp.
```

## Run Log Format

Append to `.claude/overseer-run.log` with `>>`. Use `date "+%Y-%m-%d %H:%M:%S"` for timestamps.

```
[YYYY-MM-DD HH:MM:SS] Cycle N | Phase 1 — Triage
→ git status + build + tests
← Build: PASS | Tests: 52/52 | Warnings: 3 | Decision: Phase 2

[YYYY-MM-DD HH:MM:SS] Cycle N | Phase 2 — Analyze
→ Dimensions: 2b, 2c | Changed: src/auth.rs, src/api.rs
← Findings: 2 (1 high, 1 medium)

[YYYY-MM-DD HH:MM:SS] Cycle N | Phase 4 — Execute
→ Worktree: cycle-7 | Fixing f1 (HIGH)
← Build: PASS | Tests: 52/52

[YYYY-MM-DD HH:MM:SS] Cycle N | Phase 5 — PR & Review
→ PR #42 created | Review: APPROVE

[YYYY-MM-DD HH:MM:SS] Cycle N | Phase 6 — Merge
← PR #42 merged | Worktree cleaned
```

## Worktree Troubleshooting

If worktree creation fails:
- Check if `.claude/worktrees/` exists; create if not
- Prune stale worktrees: `git worktree list` then `git worktree prune`
- Fall back to feature branches if unsupported
- Track in `active_worktrees` state — never leave orphans

If PR creation fails (no remote, no `gh`):
- Commit to branch, merge locally: `git checkout main && git merge <branch> && git branch -d <branch>`
- Log degraded workflow in run log

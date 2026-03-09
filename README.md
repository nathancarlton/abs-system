# Agent-Based System (ABS) — Autonomous Coding Pipeline

**Complete autonomous 24/7 coding pipeline for GitHub-based projects. No human intervention required until final code review.**

## Overview

The ABS coordinates 4 independent agents (PM, Dev, QA, Orchestration) to handle the complete feature lifecycle:

```
New Issue
    ↓
PM Agent (Analyze & Plan)
    ↓
Dev Agent (Implement)
    ↓
QA Agent (Test with 3-attempt loop)
    ↓
Human Review (Nathan only)
    ↓
Deploy
```

**Key philosophy:** Agents work async, never wait, and self-coordinate via GitHub labels. The system runs 24/7 without human prompts until final approval.

---

## Agents

### 1. **PM Agent** — Autonomous Product Manager
- **Triggered:** 11pm nightly or on-demand via Discord
- **Job:** Analyze open GitHub issues and write detailed PRDs
- **Output:** Labeled issues with `priority:high/critical/medium/low` and `ready-for-dev` when appropriate
- **Details:** See `agents/pm-agent.md`

### 2. **Dev Agent** — Autonomous Developer
- **Triggered:** When Orchestration detects `ready-for-dev` label
- **Job:** Implement features per PRD, push to dev branch, open PR
- **Output:** PR to dev branch with acceptance criteria checklist and QA attempt counter (1/3)
- **Details:** See `agents/dev-agent.md`

### 3. **QA Agent** — Adversarial Tester
- **Triggered:** When Orchestration detects new PR to dev branch
- **Job:** Test each acceptance criterion, find edge cases, report specific issues
- **Output:** Test report comment on PR with decision: ✅ APPROVED, 🔧 REQUEST CHANGES, or ❌ REJECTED
- **Loop:** Max 3 attempts; after 3 failures, issue returns to PM for PRD revision
- **Details:** See `agents/qa-agent.md`

### 4. **Orchestration Agent** — Pipeline Coordinator
- **Triggered:** Every 15 minutes automatically
- **Job:** Detect state changes and trigger next agent in workflow
- **Output:** Label transitions and agent triggering
- **Details:** See `agents/orchestration-agent.md`

---

## Label Lifecycle

All agents use GitHub labels to track issue state. The lifecycle is simple and clean:

| Stage | Labels | Next Step |
|-------|--------|-----------|
| New | (none) | PM analyzes |
| PM analysis complete | `priority:*`, `ready-for-dev` | Orchestration triggers Dev |
| Dev working | `priority:*`, `in-progress` | Dev opens PR |
| QA testing | `priority:*`, `in-progress` | QA posts report |
| QA approved | `priority:*`, `awaiting-human-review` | Nathan reviews & merges |
| QA rejected (3x) | `priority:*` (only) | Back to PM for PRD revision |

**See `docs/LABEL-FLOW.md` for the complete label flow diagram and transition rules.**

---

## Configuration

### projects-config.json

The single source of truth for all active projects:

```json
{
  "projects": [
    {
      "id": "sketchvault",
      "name": "SketchVault",
      "github_repos": ["nathancarlton/sketchvault"],
      "local_paths": ["/path/to/local/repo"],
      "status": "active",
      "priority": "high"
    }
  ],
  "abs_flow": {
    "polling_interval_minutes": 15,
    "dev_attempts_max": 3
  }
}
```

All agents read this file to discover repos to check.

---

## How It Works

### Scenario 1: New Feature Request

1. **Human posts issue on GitHub** with description (no labels yet)
2. **Orchestration (runs every 15 min) detects unanalyzed issue**
3. **Orchestration triggers PM Agent**
4. **PM Agent analyzes**, posts PRD comment, adds `priority:high` + `ready-for-dev` labels
5. **Orchestration (next 15-min cycle) detects `ready-for-dev` label**
6. **Orchestration removes `ready-for-dev`, adds `in-progress`, triggers Dev Agent**
7. **Dev Agent implements feature** per PRD, pushes to dev branch, opens PR to dev
8. **Orchestration (next 15-min cycle) detects new PR to dev**
9. **Orchestration triggers QA Agent**
10. **QA Agent tests**, posts report comment with decision
11. **If APPROVED:** Orchestration removes `in-progress`, adds `awaiting-human-review`
12. **Human (Nathan) reviews** PR, merges to main, closes issue
13. **Deploy** happens automatically

### Scenario 2: QA Rejects (Attempt 1 or 2)

1. **QA posts test report: "🔧 REQUEST CHANGES"** with specific issues found
2. **Orchestration (next 15-min cycle) reads report**, extracts attempt counter (e.g., 1/3)
3. **Orchestration triggers Dev Agent** with QA feedback
4. **Dev Agent fixes issues**, pushes new commits to SAME PR, updates attempt counter to 2/3
5. **Orchestration triggers QA Agent again**
6. **Loop repeats** up to 3 attempts

### Scenario 3: QA Rejects 3 Times

1. **QA posts test report: "🔧 REQUEST CHANGES"** with attempt counter = 3/3
2. **Orchestration (next 15-min cycle) detects 3/3 failures**
3. **Orchestration removes `in-progress` label**, leaving only `priority:high`
4. **Orchestration triggers PM Agent** with full test report history
5. **PM Agent revises PRD**, posts new "## PRD (Revised):" comment with updated criteria
6. **PM Agent adds `ready-for-dev` label** to restart the cycle
7. **New attempt cycle starts at 1/3**

---

## Integration with NanoClaw

The ABS agents run in **NanoClaw containers** — isolated Node.js sandboxes that:
- Mount local GitHub repos
- Have GitHub CLI (`gh`) installed with auth via `GITHUB_TOKEN`
- Can read shared config files from `/workspace/global/`
- Report status to NanoDash dashboard API

### Environment Variables Required

Each agent container needs:
- `GITHUB_TOKEN` — OAuth token for `gh` CLI auth
- `DASHBOARD_API_URL` — Endpoint for NanoDash dashboard updates
- `DASHBOARD_API_KEY` — Auth key for dashboard API

---

## Project-Specific Context

### SketchVault
- **Tech:** Vite + React + TypeScript + Tailwind, Supabase with RLS
- **Focus:** Music composition app for composers
- **Priorities:** Stability > UX > Workflow > Monetization
- **Repo:** `nathancarlton/sketchvault`

### NanoDash
- **Tech:** PHP on Dreamhost, vanilla JavaScript
- **Focus:** Personal task/project tracking
- **Priorities:** Reliability > Usability > Features
- **Repo:** `nathancarlton/nanodash`

---

## Adding New Projects

1. Add entry to `projects-config.json`
2. Create GitHub repo with `dev` branch
3. Agents will automatically discover and process issues from that repo

Example:
```json
{
  "id": "new-project",
  "name": "New Project",
  "github_repos": ["nathancarlton/new-project"],
  "local_paths": ["/path/to/new-project"],
  "status": "active",
  "priority": "high"
}
```

---

## Failure Handling

- **GitHub API down:** Orchestration logs and retries next 15-min cycle
- **Agent crashes:** Container exits, NanoClaw logs error, next cycle retries
- **PR/issue unreachable:** Agent logs and skips that item
- **QA hangs:** 15-minute timeout per phase; report what tested so far and exit
- **Label doesn't exist:** Agent creates it automatically

---

## Discord Commands (via NanoClaw)

```
@NanoDisco pm                          # Trigger PM Agent immediately
@NanoDisco dev                         # Trigger Dev Agent
@NanoDisco qa                          # Trigger QA Agent
@NanoDisco status                      # Show current pipeline status
@NanoDisco force orchestration         # Force 15-min orchestration cycle now
```

---

## Monitoring

Check agent status:
1. **GitHub Issues & PRs:** Labels show pipeline state
2. **NanoDash:** Dashboard shows last agent action per project
3. **Discord:** Agents post summaries when done
4. **Logs:** NanoClaw logs show agent invocations and state changes

---

## Architecture Decisions

See `docs/ARCHITECTURE.md` for detailed design rationale, including:
- Why label-based coordination vs. custom fields
- Why 3-attempt QA loop
- Why Orchestration runs every 15 min, not 24/7
- How agents avoid duplicate work

---

## File Structure

```
abs-system/
├── README.md                          # This file
├── agents/
│   ├── pm-agent.md                    # PM Agent instructions
│   ├── dev-agent.md                   # Dev Agent instructions
│   ├── qa-agent.md                    # QA Agent instructions
│   └── orchestration-agent.md         # Orchestration Agent instructions
├── docs/
│   ├── LABEL-FLOW.md                  # Label lifecycle & rules
│   ├── ARCHITECTURE.md                # Design decisions & rationale
│   └── projects-config.json           # Projects configuration
└── .gitignore
```

---

## Next Steps

1. **Deploy agents** to NanoClaw container system
2. **Register agent groups** in NanoClaw database (`registered_groups` table)
3. **Set GITHUB_TOKEN** in `.env` file
4. **Test with issue #3+** in test repo
5. **Monitor pipeline** on NanoDash and Discord

---

## References

- **NanoClaw:** https://github.com/nathancarlton/nanoclaw
- **SketchVault:** https://github.com/nathancarlton/sketchvault
- **NanoDash:** https://github.com/nathancarlton/nanodash

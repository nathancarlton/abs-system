# GitHub Label Status Flow

This document defines how labels transition through the ABS pipeline to track issue status.

## Label Definitions

| Label | Meaning | Applied By | Removed By |
|-------|---------|------------|------------|
| `priority:critical` | Security/crash/data loss | PM Agent | Never |
| `priority:high` | Major UX/stability issue | PM Agent | Never |
| `priority:medium` | UI improvement | PM Agent | Never |
| `priority:low` | Polish | PM Agent | Never |
| `ready-for-dev` | PRD complete, approved for Dev | PM Agent | Orchestration (when Dev starts) |
| `in-progress` | Dev is working on it | Orchestration | Orchestration (when approved by QA) |
| `qa-approved` | QA approved, ready for human review | Orchestration | Nathan (when merged) |
| `awaiting-human-review` | QA passed, waiting for Nathan | Orchestration | Nathan (when merged) |
| `wontfix` | Intentionally not doing this | Manual | Manual |
| `archived` | Old/obsolete | Manual | Manual |
| `duplicate` | Superseded by another issue | Manual | Manual |

## Issue Lifecycle & Label Transitions

```
┌─ New Issue Created
│  Labels: (none)
│
├─ PM Agent Analyzes
│  ADD: priority:high (or critical/medium/low)
│  ADD: ready-for-dev (if high or critical)
│  Labels: priority:high, ready-for-dev
│
├─ Orchestration Detects ready-for-dev
│  REMOVE: ready-for-dev
│  ADD: in-progress
│  Labels: priority:high, in-progress
│
├─ Dev Agent Implements & Opens PR
│  Labels: priority:high, in-progress (unchanged)
│
├─ Orchestration Detects PR to dev
│  Labels: priority:high, in-progress (unchanged, waiting for QA)
│
├─ QA Agent Tests PR
│  ├─ IF APPROVED (attempt ≤ 3):
│  │  REMOVE: in-progress
│  │  ADD: awaiting-human-review
│  │  Labels: priority:high, awaiting-human-review
│  │
│  └─ IF REJECTED 3 TIMES:
│     REMOVE: in-progress, ready-for-dev
│     Labels: priority:high (only)
│     → PM Agent re-analyzes
│     → PM posts "## PRD (Revised):" comment
│     → Cycle repeats from "PM Agent Analyzes"
│
└─ Nathan Reviews & Merges
   REMOVE: awaiting-human-review
   Labels: priority:high (only, issue closed)
```

## Key Rules

1. **Priority labels are permanent** — never removed, indicate original importance
2. **Status labels transition** — `ready-for-dev` → `in-progress` → `awaiting-human-review`
3. **Exclusion labels prevent analysis** — `wontfix`, `archived`, `duplicate` are skipped entirely
4. **PM re-analyzes on failure** — when issue loses all status labels (sent back after 3 QA failures)

## Commands for Each Agent

### PM Agent
```bash
# Add priority label
gh issue edit <number> --repo <owner/repo> --add-label "priority:high"

# Add ready-for-dev if high/critical
gh issue edit <number> --repo <owner/repo> --add-label "ready-for-dev"

# Create labels if missing
gh label create "priority:critical" --repo <owner/repo> --color "D73A49" 2>/dev/null || true
gh label create "ready-for-dev" --repo <owner/repo> --color "0E8A16" 2>/dev/null || true
```

### Orchestration Agent
```bash
# When triggering Dev Agent (remove ready-for-dev, add in-progress)
gh issue edit <number> --repo <owner/repo> --remove-label "ready-for-dev"
gh issue edit <number> --repo <owner/repo> --add-label "in-progress"

# When QA approves (remove in-progress, add awaiting-human-review)
gh issue edit <number> --repo <owner/repo> --remove-label "in-progress"
gh issue edit <number> --repo <owner/repo> --add-label "awaiting-human-review"

# When sending back to PM after 3 QA failures
gh issue edit <number> --repo <owner/repo> --remove-label "in-progress" 2>/dev/null || true
gh issue edit <number> --repo <owner/repo> --remove-label "ready-for-dev" 2>/dev/null || true
```

### QA Agent
- **Does NOT manage labels** — only posts test reports
- Orchestration handles label transitions based on QA results

### Dev Agent
- **Does NOT manage labels** — implements code
- Orchestration manages transitions

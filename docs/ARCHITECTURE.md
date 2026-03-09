# Architecture — Design Decisions & Rationale

This document explains the key architectural decisions behind the ABS system.

---

## 1. Label-Based Coordination (Not Custom Fields)

**Decision:** Use GitHub labels to track pipeline state (`ready-for-dev`, `in-progress`, `awaiting-human-review`)

**Why:**
- Labels are queryable via `gh issue list --label "ready-for-dev"`
- All agents can read/write labels without special API calls
- Labels are lightweight and version-controlled in issue history
- No custom fields require Org-level setup or GitHub Enterprise

**Alternative considered:** Custom fields (GitHub's built-in custom fields feature)
- Rejected: Requires GitHub Enterprise or Organization-level configuration
- Not available on private repos with free tier

**Impact:** All agents must understand label semantics. See `LABEL-FLOW.md` for complete rules.

---

## 2. Three-Attempt QA Loop

**Decision:** QA Agent tests max 3 times. After 3 failures, issue returns to PM for PRD revision.

**Why:**
- **Prevents infinite loops:** Without a hard limit, bad PRDs could cause endless dev/test cycles
- **Forces PM to improve requirements:** If Dev can't implement it 3 times, the PRD is likely unclear
- **Respects human time:** Nathan shouldn't review broken code 10 times
- **Creates feedback signal:** Dev and QA failures inform PM of scope/clarity issues

**Rationale for exactly 3:**
- 3 attempts = 1 hour of async work (15 min per cycle × 3)
- Enough to catch simple bugs or scope issues
- Not so many that it wastes time on bad requirements

**How it works:**
- Orchestration reads `## QA Attempts: N/3` from PR body
- If N < 3 and QA rejects: Trigger Dev again
- If N = 3 and QA rejects: Reset to PM for re-analysis
- If QA approves at any attempt: Move to human review

---

## 3. Orchestration Runs Every 15 Minutes (Not 24/7)

**Decision:** Orchestration Agent is scheduled as a 15-minute cron job, not a continuous daemon.

**Why:**
- **Reduces cloud costs:** NanoClaw doesn't need a persistent process
- **Predictable intervals:** Agents know when to expect work (every 15 min)
- **Atomic operations:** Each 15-min cycle is a complete state snapshot
- **Error recovery:** If a cycle fails, the next one retries fresh

**How it works:**
1. Every 15 minutes, NanoClaw wakes Orchestration Agent
2. Agent reads entire GitHub state (all repos, issues, PRs, labels)
3. Agent detects state changes and triggers next agent
4. Agent reports summary to Discord/Dashboard
5. Agent exits
6. NanoClaw waits 15 minutes, repeats

**Alternative considered:** Continuous daemon with webhooks
- Rejected: Would need persistent server, higher complexity
- Would require GitHub webhook authentication & verification

---

## 4. Agents Work Async Without Waiting

**Decision:** Agents never wait for human confirmation. They report status and exit.

**Why:**
- **24/7 operation:** No human needs to babysit the system
- **Multiple issues in progress:** While Dev is implementing issue #1, QA is testing issue #2
- **Nathan reviews async:** He can check approved PRs whenever convenient
- **Reduces context switching:** Agents focus on one task, exit, let Orchestration coordinate

**How it works:**
- Dev doesn't wait for QA test results — it pushes code and exits
- QA doesn't wait for Nathan's approval — it posts report and exits
- Orchestration cycles every 15 min and coordinates handoffs
- Nathan can review multiple approved PRs whenever ready

---

## 5. Single projects-config.json for Discovery

**Decision:** All agents read the same `projects-config.json` file to discover repos.

**Why:**
- **Single source of truth:** Adding a new project requires one file change
- **Scalable:** 2 projects now, 10 later — agents auto-discover
- **Avoids hardcoding:** Agents don't hardcode repo names
- **Configuration vs. code:** Project list is config, not baked into agent logic

**How it works:**
```json
{
  "projects": [
    {
      "id": "sketchvault",
      "github_repos": ["nathancarlton/sketchvault"],
      "local_paths": ["/workspace/project"],
      "status": "active"
    }
  ]
}
```

Each agent reads this file and iterates over all projects.

**Alternative considered:** Hardcode repo list in each CLAUDE.md
- Rejected: Changes would require updating 4 agent files, hard to maintain

---

## 6. Natural Language Instructions (Not Pseudo-Code)

**Decision:** Agent CLAUDE.md files use executable natural language, not pseudo-code or bash scripts.

**Why:**
- **Claude can execute natural language:** "Read the projects config, then list open PRs"
- **Claude cannot execute pseudo-code:** Mixing code languages (bash, JS, jq) confuses reasoning
- **Simpler to read:** Clear steps in English are easier to debug

**Wrong approach (hanging agents):**
```markdown
## Step 1: Fetch Issues
\`\`\`bash
gh issue list --repo <owner/repo> --state open \
  --json number,title,labels --limit 50 | jq '.[] | select(...)'
\`\`\`
```

**Correct approach (working agents):**
```markdown
## Step 1: Fetch Issues Per Repo

Read the projects config file to find repos:
\`\`\`bash
cat /workspace/global/projects-config.json
\`\`\`

Parse the repos list and, for each repo, list open issues:
\`\`\`bash
gh issue list --repo <owner/repo> --state open --json number,title,labels
\`\`\`

For each issue, filter out any that already have status labels.
```

The key: **Steps are natural language. Code snippets show shell syntax.**

---

## 7. GitHub-First Pattern

**Decision:** All agents write to GitHub first (issues, comments, PRs), then notify via Discord/Dashboard.

**Why:**
- **GitHub is the source of truth:** Issue state, PR code, test reports all live on GitHub
- **Async notifications:** Discord/Dashboard summaries are informational only
- **Resilience:** Even if Discord is down, GitHub has the real data
- **Audit trail:** All actions are in GitHub history

**How it works:**
1. PM posts PRD comment on GitHub issue → Then notifies Discord
2. Dev opens PR on GitHub → Then notifies Discord
3. QA posts test report on GitHub PR → Then notifies Discord
4. Orchestration updates labels on GitHub → Then notifies Dashboard

---

## 8. Containerized Agents (Sandbox Isolation)

**Decision:** Each agent runs in a Docker container with mounted workspace and mutable `/workspace` directory.

**Why:**
- **Isolation:** Agents can't interfere with each other
- **Deterministic environment:** Python, Node, GitHub CLI installed same way
- **Mutable state:** Agents can clone/modify repos without affecting host
- **Clean exit:** Container dies after agent finishes

**How it works:**
```
NanoClaw (main process)
  ↓
  Spawns container for PM Agent
    ├─ Mounts /workspace (shared)
    ├─ Mounts project repos (e.g., /workspace/PROJECTS/sketchvault)
    ├─ Env vars: GITHUB_TOKEN, DASHBOARD_API_KEY, etc.
    └─ Runs: claude-agent-sdk with CLAUDE.md
  ↓
  PM Agent exits
  ↓
  Reports status via IPC
```

---

## 9. No "Waiting" Between Agents

**Decision:** Agents never call other agents directly. Orchestration coordinates handoffs.

**Why:**
- **Avoids deadlocks:** If Dev called QA, and QA called PM, cycles could freeze
- **Simpler reasoning:** Each agent's job is clearly defined
- **Enables parallelism:** Multiple dev/qa cycles can overlap

**Wrong approach (circular dependency):**
```
Dev: "Implement this" → calls QA
QA: "Test this" → calls Orchestration
Orchestration: "Check if done" → calls Dev (loop!)
```

**Correct approach (clear handoff):**
```
Orchestration (every 15 min):
  IF ready-for-dev found:
    Trigger Dev Agent (via NanoClaw)
    Wait for container to exit
    Continue to next repo

Orchestration then:
  IF new PR found:
    Trigger QA Agent (via NanoClaw)
    Wait for container to exit
    Continue to next PR
```

---

## 10. GitHub Repos With `dev` Branch

**Decision:** Each project has a `dev` branch where Dev opens PRs and Orchestration polls.

**Why:**
- **Staging area:** `dev` is where code lands before Nathan reviews
- **Isolated from main:** Main is production; dev is development
- **Clear workflow:** main ← (human approves) ← dev ← (agents work)
- **Safe defaults:** If Orchestration breaks, it can't auto-merge to main

**How it works:**
```
                     Nathan (manual)
                            ↓
                     Approve & merge
                            ↓
                          main (prod)
                            ↑
                     (human reviews)
                            ↓
                          dev (staging)
                            ↑
                   (agents work here)
                     QA tests ← Dev codes ← PM plans
```

---

## Trade-Offs & Decisions Not Taken

### Why Not Auto-Merge to Main?
**Decision:** Nathan manually merges from dev to main.

**Why:**
- Human judgment: There might be deployment issues, edge cases, or timing concerns
- Safety: Auto-merging breaks the audit trail and removes Nathan's sign-off
- Philosophy: Nathan owns production; automation owns development

### Why Not Use GitHub Actions?
**Decision:** Use NanoClaw containers with explicit agent files.

**Why:**
- GitHub Actions can't do complex reasoning (no Claude integration)
- Would need to hardcode workflow files for each agent
- More maintainable to have agents in one place

### Why Not Use PR Reviews Instead of Labels?
**Decision:** Use labels for state tracking, not PR reviews.

**Why:**
- PR reviews are for human feedback; labels are for automation
- Mixing human reviews + agent comments gets confusing
- Labels are queryable; approval status is not

---

## Future Improvements

1. **Webhook-based triggering:** Replace 15-min cron with GitHub webhooks for instant reactions
2. **Sub-agent system:** QA Agent spawning specialized sub-agents for different test types
3. **Smart backoff:** If agents fail repeatedly, scale back work load
4. **Dashboard updates:** Agents call dashboard API to show live progress
5. **Cost tracking:** Log time + tokens per issue to estimate effort

---

## Summary

The ABS is designed around these core principles:

| Principle | Implementation |
|-----------|-----------------|
| **Async coordination** | Label-based, Orchestration as coordinator |
| **No infinite loops** | 3-attempt QA limit, forcing PM re-eval |
| **Simple & debuggable** | Natural language, not pseudo-code |
| **Scalable discovery** | Single projects-config.json file |
| **GitHub is source of truth** | All state stored as labels/comments |
| **Sandbox isolation** | Containerized agents, clean exits |
| **Clear handoffs** | Orchestration decides who works next |
| **Human judgment preserved** | Nathan reviews approved work before deploy |

These decisions work together to create a system that:
- Runs 24/7 without human intervention
- Never blocks on manual confirmation
- Self-coordinates via GitHub state
- Scales from 2 projects to N projects
- Recovers from agent crashes automatically
- Maintains complete audit trail

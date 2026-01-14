---
name: session-wrap
description: This skill should be used when the user asks to "wrap up session", "end session", "session wrap", "/wrap", "document learnings", "what should I commit", or wants to analyze completed work before ending a coding session.
version: 2.1.0
---

# Session Wrap Skill

Comprehensive session wrap-up workflow with multi-agent analysis.

## Execution Flow

```
┌─────────────────────────────────────────────────────┐
│  1. Check Git Status                                │
├─────────────────────────────────────────────────────┤
│  2. Phase 1: 4 Analysis Agents (Parallel)           │
│     ┌─────────────────┬─────────────────┐           │
│     │  doc-updater    │  automation-    │           │
│     │  (docs update)  │  scout          │           │
│     ├─────────────────┼─────────────────┤           │
│     │  learning-      │  followup-      │           │
│     │  extractor      │  suggester      │           │
│     └─────────────────┴─────────────────┘           │
├─────────────────────────────────────────────────────┤
│  3. Phase 2: Validation Agent (Sequential)          │
│     ┌───────────────────────────────────┐           │
│     │       duplicate-checker           │           │
│     │  (Validate Phase 1 proposals)     │           │
│     └───────────────────────────────────┘           │
├─────────────────────────────────────────────────────┤
│  4. Integrate Results & AskUserQuestion             │
├─────────────────────────────────────────────────────┤
│  5. Execute Selected Actions                        │
├─────────────────────────────────────────────────────┤
│  6. Update wrapup.md (Auto)                         │
└─────────────────────────────────────────────────────┘
```

## Step 1: Check Git Status

```bash
git status --short
git diff --stat HEAD~3 2>/dev/null || git diff --stat
```

## Step 2: Phase 1 - Analysis Agents (Parallel)

Execute 4 agents in parallel (single message with 4 Task calls).

### Session Summary (Provide to all agents)

```
Session Summary:
- Work: [Main tasks performed in session]
- Files: [Created/modified files]
- Decisions: [Key decisions made]
```

### Parallel Execution

```
Task(
    subagent_type="doc-updater",
    description="Document update analysis",
    prompt="[Session Summary]\n\nAnalyze if CLAUDE.md, context.md need updates."
)

Task(
    subagent_type="automation-scout",
    description="Automation pattern analysis",
    prompt="[Session Summary]\n\nAnalyze repetitive patterns or automation opportunities."
)

Task(
    subagent_type="learning-extractor",
    description="Learning points extraction",
    prompt="[Session Summary]\n\nExtract learnings, mistakes, and new discoveries."
)

Task(
    subagent_type="followup-suggester",
    description="Follow-up task suggestions",
    prompt="[Session Summary]\n\nSuggest incomplete tasks and next session priorities."
)
```

### Agent Roles

| Agent | Role | Output |
|-------|------|--------|
| **doc-updater** | Analyze CLAUDE.md/context.md updates | Specific content to add |
| **automation-scout** | Detect automation patterns | skill/command/agent suggestions |
| **learning-extractor** | Extract learning points | TIL format summary |
| **followup-suggester** | Suggest follow-up tasks | Prioritized task list |

## Step 3: Phase 2 - Validation Agent (Sequential)

Run after Phase 1 completes (dependency on Phase 1 results).

```
Task(
    subagent_type="duplicate-checker",
    description="Phase 1 proposal validation",
    prompt="""
Validate Phase 1 analysis results.

## doc-updater proposals:
[doc-updater results]

## automation-scout proposals:
[automation-scout results]

Check if proposals duplicate existing docs/automation:
1. Complete duplicate: Recommend skip
2. Partial duplicate: Suggest merge approach
3. No duplicate: Approve for addition
"""
)
```

## Step 4: Integrate Results

```markdown
## Wrap Analysis Results

### Documentation Updates
[doc-updater summary]
- Duplicate check: [duplicate-checker feedback]

### Automation Suggestions
[automation-scout summary]
- Duplicate check: [duplicate-checker feedback]

### Learning Points
[learning-extractor summary]

### Follow-up Tasks
[followup-suggester summary]
```

## Step 5: Action Selection

```
AskUserQuestion(
    questions=[{
        "question": "Which actions would you like to perform?",
        "header": "Wrap Options",
        "multiSelect": true,
        "options": [
            {"label": "Create commit (Recommended)", "description": "Commit staged changes"},
            {"label": "Update CLAUDE.md", "description": "Add project-wide guidelines/instructions"},
            {"label": "Create automation", "description": "Generate skill/command/agent"},
            {"label": "Skip", "description": "End without action"}
        ]
    }]
)
```

**Note:** wrapup.md is ALWAYS updated automatically in Step 7 (not optional).

## Step 6: Execute Selected Actions

Execute only the actions selected by user.

## Step 7: Update wrapup.md (Automatic)

**ALWAYS execute this step after completing the session wrap.**

This step uses outputs from:
- **doc-updater**: wrapup.md specific proposals
- **learning-extractor**: TIL items
- **followup-suggester**: Next tasks

### File Location

Check for wrapup.md in order:
1. `.claude/wrapup.md` (preferred)
2. `wrapup.md` (project root)
3. Create `.claude/wrapup.md` if not exists

### wrapup.md Purpose

**Session progress tracking file** containing:
- Completed work and commits
- Troubleshooting records (errors + solutions)
- Learnings (TIL)
- Pending/uncommitted work
- Next tasks/TODO
- Remote server status

**NOT for:** Project-wide guidelines (those go to CLAUDE.md)

### Update Content Structure

```markdown
# Session Progress Summary

> Last Updated: [Current Date/Time]

## Branch Info

- **Current Branch**: [branch name]
- **Remote Tracking**: [origin/branch]

---

## Completed Work

### [Commit Hash] [Commit Message]
- [Brief description]
- Files: [changed files]

---

## Troubleshooting Log

### [Issue Title]
**Error:**
```
[Error message]
```

**Cause:**
[Root cause]

**Solution:**
[How resolved]

---

## Learnings (TIL)

[learning-extractor output]

---

## Pending Work

| File | Status | Notes |
|------|--------|-------|
| [file] | [modified/untracked] | [description] |

---

## Next Tasks

[followup-suggester output - prioritized list]

---

## Remote Server Status (if applicable)

- Server: [name]
- Last Run: [timestamp]
- Status: [success/failure]
- Data Location: [path]
```

### Update Rules

1. **Preserve existing content** - Merge, don't overwrite
2. **Add new commits** to "Completed Work" (prepend, newest first)
3. **Append troubleshooting** entries (keep history)
4. **Update "Pending Work"** from current git status
5. **Refresh "Next Tasks"** with followup-suggester output
6. **Add learnings** from learning-extractor
7. **Update timestamp** in header

### Implementation

```
1. Read existing .claude/wrapup.md (or create if not exists)
2. Collect data from:
   - git log (recent commits)
   - git status (pending work)
   - Session conversation (troubleshooting)
   - doc-updater output (wrapup.md proposals)
   - learning-extractor output (TIL)
   - followup-suggester output (next tasks)
3. Merge into existing structure
4. Write updated content
5. Notify user: "Updated .claude/wrapup.md"
```

---

## Quick Reference

### When to Use

- End of significant work session
- Before switching to different project
- After completing a feature or fixing a bug

### When to Skip

- Very short session with trivial changes
- Only reading/exploring code
- Quick one-off question answered

### Arguments

- Empty: Proceed interactively (full workflow)
- Message provided: Use as commit message and commit directly

## Additional Resources

See `references/multi-agent-patterns.md` for detailed orchestration patterns.

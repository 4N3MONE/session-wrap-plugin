---
name: session-wrap
description: This skill should be used when the user asks to "wrap up session", "end session", "session wrap", "/wrap", "document learnings", "what should I commit", or wants to analyze completed work before ending a coding session.
version: 2.2.0
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
│  7. Update Documentation Files (Auto, MANDATORY)    │
│     ├─ claude.md  (project guidelines)              │
│     └─ wrapup.md  (session progress)                │
├─────────────────────────────────────────────────────┤
│  8. Update Obsidian Daily Note (Auto, MANDATORY)    │
│     └─ Claude Session Log section                   │
├─────────────────────────────────────────────────────┤
│  9. Recommend Azure DevOps Board Sync (Optional)    │
│     └─ Conditional: azure-boards skill installed +  │
│        repo in ~/.claude/devops-defaults.json       │
│        → AskUserQuestion                            │
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
            {"label": "Create automation", "description": "Generate skill/command/agent"},
            {"label": "Skip", "description": "End without action"}
        ]
    }]
)
```

**Note:** wrapup.md and claude.md are ALWAYS updated automatically in Step 7 (not optional).

## Step 6: Execute Selected Actions

Execute only the actions selected by user.

## Step 7: Update Documentation Files (Automatic)

**ALWAYS execute this step after completing the session wrap. This is NOT optional.**

**Both files are MANDATORY updates — do NOT skip either one.**

This step uses outputs from:
- **doc-updater**: claude.md and wrapup.md specific proposals
- **learning-extractor**: TIL items
- **followup-suggester**: Next tasks

### 7-A: Update/Create claude.md

#### File Location

Check for claude.md in order:
1. `.claude/claude.md` (preferred)
2. `CLAUDE.md` (project root)
3. Create `.claude/claude.md` if neither exists

#### claude.md Purpose

**Project-wide guidelines and instructions** that persist across all sessions:
- Project overview and architecture
- Development environment setup (branches, package manager, Docker, etc.)
- API endpoints and key interfaces
- Coding conventions and standards
- Important caveats and gotchas discovered during development
- Reference to wrapup.md for session continuity

#### Update Rules

1. **Read existing file** (if exists) and preserve its structure
2. **Merge doc-updater proposals** for claude.md — add new sections or update existing ones
3. **Add newly discovered project guidelines** from this session (e.g., new endpoints, env vars, workflow caveats)
4. **Do NOT add session-specific details** (those go to wrapup.md)
5. **Keep concise** — focus on information Claude needs in ALL future sessions

#### Implementation

```
1. Read existing claude.md (or initialize template if not exists)
2. Apply doc-updater claude.md proposals (add/update sections)
3. Add any new project-wide knowledge discovered this session
4. Write updated content using Write or Edit tool
5. Log: "Updated .claude/claude.md"
```

### 7-B: Update/Create wrapup.md

#### File Location

Check for wrapup.md in order:
1. `.claude/wrapup.md` (preferred)
2. `wrapup.md` (project root)
3. Create `.claude/wrapup.md` if not exists

#### wrapup.md Purpose

**Session progress tracking file** containing:
- Completed work and commits
- Troubleshooting records (errors + solutions)
- Learnings (TIL)
- Pending/uncommitted work
- Next tasks/TODO
- Remote server status

**NOT for:** Project-wide guidelines (those go to CLAUDE.md)

#### Update Content Structure

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

#### Update Rules

1. **Preserve existing content** - Merge, don't overwrite
2. **Add new commits** to "Completed Work" (prepend, newest first)
3. **Append troubleshooting** entries (keep history)
4. **Update "Pending Work"** from current git status
5. **Refresh "Next Tasks"** with followup-suggester output
6. **Add learnings** from learning-extractor
7. **Update timestamp** in header

#### Implementation

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

## Step 8: Update Obsidian Daily Note (Automatic)

**ALWAYS execute this step after Step 7. This is NOT optional.**

Update the `## 🤖 Claude Session Log` section in today's Obsidian daily note with a concise summary of what was accomplished in this session.

### File Location

The daily note path follows this pattern:
```
/mnt/c/Users/yyoo029/Documents/Obsidian/유용상/Daily/YYYY-MM-DD.md
```

Use today's date (from the system) to construct the file path.

### Target Section

Find the `## 🤖 Claude Session Log` section and append the session log **between the heading and the `---` separator** that follows it.

### Content Format

```markdown
### [Project Name] — YYYY-MM-DD HH:MM

**Branch:** `branch-name`

**작업 내역:**
- [Completed task 1 — brief description]
- [Completed task 2 — brief description]
- ...

**주요 결정/발견:**
- [Key decision or discovery, if any]

**다음 작업:**
- [Top 2-3 next tasks from followup-suggester]
```

### Content Rules

1. **Read the existing daily note** — preserve all existing content
2. **Append** to the Claude Session Log section (do NOT overwrite other session logs from the same day)
3. **Keep it concise** — max 10-15 lines per session entry
4. **Use bullet points** — no paragraphs, no code blocks unless essential
5. **Include branch name** — critical for context when reviewing later
6. **Korean preferred** for descriptions, English OK for technical terms
7. **If the daily note file does not exist**, skip this step silently (do not create the file)

### Implementation

```
1. Construct today's daily note path: /mnt/c/Users/yyoo029/Documents/Obsidian/유용상/Daily/YYYY-MM-DD.md
2. Check if file exists — if not, skip with log message
3. Read the file and locate "## 🤖 Claude Session Log" section
4. Build session summary from:
   - Completed work (from wrapup.md or session context)
   - Key decisions/discoveries (from learning-extractor)
   - Next tasks (from followup-suggester, top 2-3 only)
5. Append summary after the "## 🤖 Claude Session Log" heading
6. Write updated content
7. Notify user: "Updated Obsidian daily note"
```

---

## Step 9: Recommend Azure DevOps Board Sync (Optional, Conditional)

**Run only if ALL conditions are true:**
1. `azure-boards` skill is installed (genaikit plugin — `${CLAUDE_PLUGIN_ROOT}/skills/azure-boards/SKILL.md` resolvable via Claude Code's plugin loader).
2. Current repo is mapped in `~/.claude/devops-defaults.json`.
3. PAT is available (`AZURE_DEVOPS_EXT_PAT` env var set, or file-stored PAT at `~/.azure/azuredevops/personalAccessTokens`).

If any are false → skip silently. Do NOT call `check_auth.sh` here — that script requires AAD login (`az account show`) which is unnecessary for PAT-only users and would wrongly skip the entire step.

### Detection (cheap, no API calls)

```bash
# 1. azure-boards skill present (genaikit)?
ls ~/.claude/plugins/cache/*/genaikit/*/skills/azure-boards/SKILL.md >/dev/null 2>&1 || skip

# 2. Repo mapped? Check repo from git remote against devops-defaults.json.
python3 -c "
import json, sys, subprocess
from pathlib import Path
url = subprocess.run(['git','config','--get','remote.origin.url'],
    capture_output=True, text=True).stdout.strip()
if 'dev.azure.com' not in url: sys.exit(2)
repo = url.rstrip('/').rsplit('/',1)[-1]
mapping_file = Path.home() / '.claude/devops-defaults.json'
mapping = json.loads(mapping_file.read_text()) if mapping_file.exists() else {}
sys.exit(0 if repo in mapping else 1)
"

# 3. PAT reachable?
[ -n "$AZURE_DEVOPS_EXT_PAT" ] || [ -s ~/.azure/azuredevops/personalAccessTokens ] || skip
```

If exit code is 0 → proceed. Non-zero → skip.

### AskUserQuestion (after Step 8, NEVER auto-execute)

This step is purely a recommendation. Do not push to the board without explicit user opt-in.

```
AskUserQuestion(
    questions=[{
        "question": "Sync this session's work to the Azure DevOps board for {team_name}?",
        "header": "Board Sync (optional)",
        "multiSelect": true,
        "options": [
            {"label": "Create work items for next tasks",
             "description": "Turn followup-suggester output into new Tasks on the board"},
            {"label": "Update existing work items",
             "description": "Apply state/comment changes to items mentioned this session"},
            {"label": "Skip",
             "description": "Do not touch the board"}
        ]
    }]
)
```

### Execution rules

- **Default to Skip** when in doubt — never assume the user wants writes.
- **Show preview before each write**: list the exact create/update operations and ask one final confirmation if more than 2 items will change.
- **Resolve team defaults from the mapping file** (`~/.claude/devops-defaults.json`). Entry shape: `{"team": "...", "area_path": "...", "iteration_path": "..."}`. `area_path` is required for create; `iteration_path` is optional and, if omitted, resolve the current sprint at runtime:
  ```bash
  GENAIKIT=$(ls -d ~/.claude/plugins/cache/*/genaikit/*/skills/azure-boards | head -1)
  team=$(python3 -c "import json,pathlib; print(json.loads(pathlib.Path.home().joinpath('.claude/devops-defaults.json').read_text())['$repo']['team'])")
  current_iter=$(az boards iteration team list --team "$team" --timeframe current --output json 2>/dev/null | jq -r '.[0].path // empty' | sed -E 's#\\Iteration\\#\\#')
  ```
- **Use genaikit azure-boards scripts** — never call the Azure DevOps REST API directly:
  ```bash
  # Create
  GENAIKIT=$(ls -d ~/.claude/plugins/cache/*/genaikit/*/skills/azure-boards | head -1)
  tmp=$(mktemp); printf '%s' "$rationale" > "$tmp"
  bash "$GENAIKIT/scripts/create_work_item.sh" \
    --type Task --title "$title" \
    --area "$area_path" --iteration "$current_iter" \
    --description-file "$tmp"
  rm -f "$tmp"

  # Update (state transition)
  bash "$GENAIKIT/scripts/update_work_item.sh" --id "$id" --state "$state"
  ```
- **Source data to map**:
  - "Create work items for next tasks" → use the followup-suggester output verbatim (one work item per suggested task; title = task line, description = rationale if present)
  - "Update existing work items" → only if the session conversation explicitly referenced work item IDs (e.g., `#AB1234` patterns in commit messages or user requests). Do not invent IDs.
- **Failure handling**: on non-zero exit from `create_work_item.sh` / `update_work_item.sh`, surface stderr verbatim and stop. Common causes: expired PAT, missing Work Items scope on PAT, wrong iteration path, locked state transition.

### After the recommendation

Whether the user accepted or skipped, conclude the wrap normally. Step 9 does not affect Steps 7/8 (those have already run).

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

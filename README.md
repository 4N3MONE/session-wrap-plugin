# Session Wrap Plugin

A Claude Code plugin for comprehensive session wrap-up with multi-agent analysis.

**Fork of [team-attention/plugins-for-claude-natives](https://github.com/team-attention/plugins-for-claude-natives)** with enhanced wrapup.md auto-update feature.

## Features

- **Multi-Agent Analysis Pipeline**: 5 specialized agents analyze your session from different perspectives
- **2-Phase Architecture**: Parallel analysis followed by sequential validation
- **Auto wrapup.md Update**: Automatically updates `.claude/wrapup.md` with session progress, troubleshooting, and learnings
- **Documentation Updates**: Identify what should be added to CLAUDE.md (project guidelines)
- **Automation Discovery**: Find patterns worth automating as skills/commands/agents
- **Learning Capture**: Extract insights, mistakes, and discoveries in TIL format
- **Follow-up Planning**: Prioritized task list for next session
- **Duplicate Prevention**: Validates proposals against existing content

## Key Enhancement: wrapup.md Auto-Update

This fork adds automatic `wrapup.md` updates to track:
- Completed work and commits
- Troubleshooting records (errors + solutions)
- Learnings (TIL)
- Pending/uncommitted work
- Next tasks/TODO
- Remote server status

### File Roles

| File | Purpose | Content |
|------|---------|---------|
| `CLAUDE.md` | Project guidelines (persistent) | Instructions Claude should always follow |
| `wrapup.md` | Session progress (dynamic) | What happened this session, next tasks |

## Installation

### Option 1: Plugin Directory

```bash
# Clone to your plugins directory
git clone https://github.com/4N3MONE/session-wrap-plugin ~/.claude/plugins/session-wrap
```

### Option 2: Direct Use

```bash
claude --plugin-dir /path/to/session-wrap-plugin
```

## Usage

### Basic Usage

```
/wrap
```

Runs the full wrap-up workflow:
1. Check git status
2. Phase 1: Run 4 analysis agents in parallel
3. Phase 2: Validate proposals for duplicates
4. Present results and let you choose actions
5. Execute selected actions
6. **Auto-update wrapup.md** (always runs)

### Quick Commit

```
/wrap fix typo in README
```

When arguments are provided, creates a commit with that message directly.

## Architecture

```
Phase 1: Analysis (Parallel)
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ doc-updater  │ automation-  │ learning-    │ followup-    │
│              │ scout        │ extractor    │ suggester    │
└──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┘
       └──────────────┴──────────────┴──────────────┘
                           │
                           ▼
Phase 2: Validation (Sequential)
┌─────────────────────────────────────────────────────────────┐
│                    duplicate-checker                         │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
                   User Selection
                           │
                           ▼
              Auto-update wrapup.md
```

## Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| `doc-updater` | sonnet | Analyze CLAUDE.md and wrapup.md update needs |
| `automation-scout` | sonnet | Detect automation opportunities |
| `learning-extractor` | sonnet | Extract learnings and mistakes |
| `followup-suggester` | sonnet | Suggest prioritized follow-up tasks |
| `duplicate-checker` | haiku | Validate proposals for duplicates |

## wrapup.md Structure

```markdown
# Session Progress Summary

> Last Updated: [timestamp]

## Branch Info
- Current Branch: feature/xyz
- Remote Tracking: origin/feature/xyz

## Completed Work
### [commit hash] [message]
- Description
- Files changed

## Troubleshooting Log
### [Issue Title]
**Error:** [error message]
**Cause:** [root cause]
**Solution:** [how resolved]

## Learnings (TIL)
- [discoveries]

## Pending Work
| File | Status | Notes |
|------|--------|-------|

## Next Tasks
1. [prioritized list]
```

## Directory Structure

```
session-wrap/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest
├── commands/
│   └── wrap.md               # /wrap command
├── agents/
│   ├── doc-updater.md        # Documentation analysis
│   ├── automation-scout.md   # Automation detection
│   ├── learning-extractor.md # Learning capture
│   ├── followup-suggester.md # Task prioritization
│   └── duplicate-checker.md  # Validation
├── skills/
│   └── session-wrap/
│       ├── SKILL.md          # Best practices guide
│       └── references/
│           └── multi-agent-patterns.md
└── README.md
```

## When to Use

**Use `/wrap` when:**
- Ending a significant work session
- Before switching to a different project
- After completing a feature or bug fix
- When unsure what to document

**Skip when:**
- Very short session with trivial changes
- Only reading/exploring code
- Quick one-off question answered

## Credits

- Original plugin by [Team Attention](https://github.com/team-attention/plugins-for-claude-natives)
- wrapup.md auto-update enhancement by [4N3MONE](https://github.com/4N3MONE)

## References

- [Anthropic Multi-Agent Research](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Azure AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)

## License

MIT

# CLAUDE.md Update

**Date:** 2026-04-16

## Change

Updated `~/CLAUDE.md` to make workflow commands auto-execute in Claude Code.

## Problem

When users typed `start-workflow` in Claude Code, it was interpreted as a question instead of a command to execute.

## Solution

Added command recognition section to `~/CLAUDE.md`:

```markdown
### Important: Command Recognition

When user types these commands directly (without "run" or "execute"), treat them as bash commands to execute:
- `start-workflow` → Execute bash command
- `workflow-config show` → Execute bash command
- `jira-search <keyword>` → Execute bash command
- `check-git-safety` → Execute bash command
- `what-can-i-ask` → Execute bash command

These are NOT questions - they are commands to run.
```

## Result

Now users can type `start-workflow` directly in Claude Code and it will execute automatically, without needing:
- `!` prefix
- "run" keyword
- Explicit instruction

## Files Modified

- `~/CLAUDE.md` - Main Claude Code guidance file (lives in home directory, not in this repo)

## Testing

Try in a new Claude Code tab:
1. Type: `start-workflow`
2. Should execute the bash command automatically
3. Should guide through git safety, Jira search, branch creation

## Note

`CLAUDE.md` is not in this git repository - it lives in `~/CLAUDE.md` (home directory) and is automatically read by Claude Code on startup.

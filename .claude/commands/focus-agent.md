---
description: Scope an agent to focus only on documentation from a specific repository
argument-hint: <agent-file-path> <repo-path>
allowed-tools:
  - Bash:
      - cat
      - sed
      - grep
      - echo
      - tee
---

# Scope Agent to Repository

This command modifies an agent file to focus only on documentation from a specific repository.

**Agent file:** `$1`
**Repository path:** `$2`

## Step 1: Read the current agent file

!cat "$1"

## Step 2: Analyze and update documentation references

Now I'll update the agent file to scope all documentation references to the specific repository at `$2`.

I'll replace:
- References to "all claude.md" with specific paths like `$2/.claude/claude.md`
- Generic documentation references with repo-specific paths
- Add the repo path context to the agent's instructions

!sed -i \
  -e "s|all claude\.md|$2/.claude/claude.md|g" \
  -e "s|@claude\.md|@$2/.claude/claude.md|g" \
  -e "s|all \.claude/|$2/.claude/|g" \
  -e "s|@\.claude/|@$2/.claude/|g" \
  "$1"

## Step 3: Verify the changes

!echo "=== Updated agent file ==="
!cat "$1"

The agent at `$1` has been updated to focus only on documentation from `$2`.
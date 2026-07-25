# github-task-manager

A Claude Code skill for managing tasks via GitHub issues in the current repository.

## What It Does

- Fetches and processes open GitHub issues from the repository where Claude is running
- Proposes solutions, provides updates, and suggests next steps for each issue
- Never closes issues — you remain solely responsible for closing
- Uses `needs-input` labels to mark issues blocked on your response
- Cleans up orphaned `needs-input` labels from closed issues

## Installation

Copy the skill into your Claude Code skills directory:

```bash
cp -r .claude/skills/github-task-manager ~/.claude/skills/
```

Or reference this repository's `.claude/skills/` path in your Claude Code configuration.

## Usage

Trigger phrases:

- "Please complete all the outstanding tasks in the GitHub issues for this repository."
- "What's the current status of all open issues?"
- "Create a new issue for me: [title]"

## Requirements

- `GITHUB_TOKEN` environment variable (GitHub Personal Access Token)
- `curl` and `jq` for API calls

## Workflow

1. Claude fetches all open issues from the repo
2. Cleans up any `needs-input` labels on closed issues
3. Processes each issue in order (oldest first)
4. Adds progress comments and proposes solutions
5. Reports a summary when done

## Skill Structure

```
.claude/skills/github-task-manager/
└── SKILL.md   # The skill definition
```

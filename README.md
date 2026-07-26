# github-task-manager

A Claude Code plugin for managing tasks via GitHub issues in the current repository.

## What It Does

- Fetches and processes open GitHub issues from the repository where Claude is running
- Proposes solutions, provides updates, and suggests next steps for each issue
- Never closes issues — you remain solely responsible for closing
- Uses `needs-input` labels to mark issues blocked on your response
- Cleans up orphaned `needs-input` labels from closed issues

## Plugin Structure

```
github-task-manager/
├── .claude-plugin/
│   └── plugin.json     # Plugin manifest
├── skills/
│   └── complete-tasks/
│       └── SKILL.md    # The skill definition
└── README.md
```

## Installation

Install the plugin by referencing this repository:

```bash
/plugin install github-task-manager@<path-to-repo>
```

Or for local development:

```bash
cc --plugin-dir /path/to/github-task-manager
```

## Usage

Use the skill directly:

```
/complete-tasks
```

Trigger phrases:
- "Please complete all the outstanding tasks in the GitHub issues for this repository."
- "What's the current status of all open issues?"
- "Work on your assigned issues"

## Requirements

- `gh` CLI installed and authenticated
- GitHub authentication via `gh auth login`

## Workflow

1. Claude fetches all open issues assigned to the current user from the repo
2. Cleans up any `needs-input` labels on closed issues
3. Processes each issue in order (oldest first)
4. Adds progress comments and proposes solutions via PRs
5. Reports a summary when done

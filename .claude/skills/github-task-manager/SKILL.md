---
name: github-task-manager
description: Manage and complete tasks tracked in GitHub issues for the current repository. Use this skill whenever the user asks you to complete outstanding GitHub issues, create new issues, or provide updates on issue progress. This skill is the primary interface for task delegation — any request involving GitHub issues in the current repository should trigger this skill. Make sure to use this skill proactively when the user mentions completing issues, checking progress, or working on tracked tasks.
---

# GitHub Task Manager

This skill manages task delegation via GitHub issues in the repository where the session is running.

## Repository Resolution

Every invocation, re-derive the repository from the local git remote:
```bash
git remote -v
```
Parse the HTTPS URL to extract `owner/repo`. The local git config is the source of truth — do not hardcode or assume a fixed repository.

## Authentication

Use the GitHub REST API (not `gh` CLI). Authentication via:
- Environment variable: `GITHUB_TOKEN` (recommended — set in shell profile or Claude settings)
- If not set, prompt the user with: "I need a GitHub token to interact with issues. Please set `GITHUB_TOKEN` in your environment or provide one now."

Base URL: `https://api.github.com`

## Core Behavior

### Issue Retrieval

Fetch open issues from the repository using the issues endpoint:
```
GET /repos/{owner}/{repo}/issues?state=open&sort=created&direction=asc
```

Filter out pull requests — issues have `pull_request: null`:
```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/issues?state=open&sort=created&direction=asc" \
  | jq '[.[] | select(.pull_request == null)]'
```

### Cleanup

Before processing issues, also check for orphaned `needs-input` labels on closed issues. Fetch all closed issues with the `needs-input` label and remove it:
```
GET /repos/{owner}/{repo}/issues?state=closed&labels=needs-input
```

For each closed issue that still has the label, remove it:
```bash
curl -s -X DELETE -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/issues/$NUMBER/labels/needs-input"
```

This is a maintenance task — don't announce it unless you find and fix something.

### Issue Processing

For each open issue, determine:
- **Actionable**: You have enough context to make progress. Proceed.
- **Blocked**: Needs user input or clarification. Label it and skip. Use a consistent label like `needs-input` so the user can find all blocked issues at once.
- **In Progress**: You've started working on it. Update the issue body or add a comment to track this.

**Never close issues.** The user remains solely responsible for closing. Your job is to propose solutions, provide updates, and advance work.

### Label Conventions

Pick a clear, consistent label for blocked issues. For example:
- `needs-input` — issue is waiting for user response

Apply it when you need clarification:
```bash
curl -s -X POST -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"labels":["needs-input"]}' \
  "https://api.github.com/repos/$OWNER/$REPO/issues/$NUMBER/labels"
```

Remove it when you receive input:
```bash
curl -s -X DELETE -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/issues/$NUMBER/labels/needs-input"
```

### Creating Issues

When the user asks to create an issue, follow this workflow as if they created it themselves:
- Set appropriate labels
- Use a consistent title format (imperative mood: "Add feature X" not "Adding feature X")
- Include all context the user provided
- Return the issue URL so they can view it

```
POST /repos/{owner}/{repo}/issues
```

### Progress Tracking

When you start work on an issue, add a comment:
```
I'm looking into this issue and will provide an update shortly.
```

When you have a solution or finding, add a comment with your proposed approach. If the work is complex, periodically update with progress comments so the user can see what you've tried.

### Completing Issues (No Closing)

Since you won't close issues, when you've completed your work on an issue:
1. Add a comment summarizing what you did
2. Remove the `needs-input` label if present
3. The user will review and close when satisfied

### API Rate Limits

GitHub API allows 5,000 requests/hour authenticated. Batch operations where possible:
- Fetch all open issues in one call, then process locally
- Use `jq` for filtering instead of multiple API calls
- If you hit rate limits, wait and retry with `Retry-After` header guidance

## Workflow for "Complete all outstanding tasks"

1. **Cleanup pass** — Check for orphaned `needs-input` labels on closed issues and remove them
2. Fetch all open issues
3. Filter out PRs and already-processed issues (check for recent comments from you)
4. For each actionable issue, work through them in order (oldest first)
5. Add progress comments to each issue
6. Report a summary to the user when done

## Summary Output Format

After completing (or partially completing) issues, report:

```
## GitHub Task Summary

Cleanup: removed needs-input from N closed issue(s) (if any)
Processed N issue(s):

| # | Title | Status | Notes |
|---|-------|--------|-------|
| 1 | Issue title | Done / Blocked / In Progress | Notes |

Issues needing your input: [list with links]
Issues advanced but not closed: [list with links]
```

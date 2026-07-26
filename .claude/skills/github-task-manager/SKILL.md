---
name: github-task-manager
description: Manage and complete tasks tracked in GitHub issues for the current repository. Use this skill whenever the user asks you to complete outstanding GitHub issues, create new issues, or provide updates on issue progress. This skill is the primary interface for task delegation — any request involving GitHub issues in the current repository should trigger this skill. Make sure to use this skill proactively when the user mentions completing issues, checking progress, or working on tracked tasks.
---

# GitHub Task Manager

This skill manages task delegation via GitHub issues in the repository where the session is running.

## Invocation

This skill is triggered by requests like:
- "Please complete all issues assigned to you in this repo"
- "Work on your assigned issues"
- "Check and complete issues assigned to me"

## Repository Resolution

Every invocation, re-derive the repository from the local git remote:
```bash
git remote -v
```
Parse the HTTPS URL to extract `owner/repo`. The local git config is the source of truth — do not hardcode or assume a fixed repository.

## Authentication

Use the GitHub CLI (`gh`). Verify authentication:
```bash
gh auth status
```

If `gh` is not authenticated, stop and inform the user: "I don't have access to GitHub. Please ensure `gh` is authenticated with `gh auth login`."

## Identifying the Current User

Determine the username of the authenticated user:
```bash
gh api user --jq .login
```

## Cleanup Pass

Before processing issues, check for orphaned `needs-input` labels on closed issues and remove them:
```bash
gh issue list --state closed --label needs-input --repo "$OWNER/$REPO" --json number,title
```

For each closed issue with `needs-input`, remove the label:
```bash
gh issue edit $NUMBER --remove-label needs-input --repo "$OWNER/$REPO"
```

This is a maintenance task — don't announce it unless you find and fix something.

## Fetching Assigned Issues

Fetch only issues assigned to the current user that are **open** and do not have the `needs-input` label:
```bash
gh issue list --assignee "@me" --state open --repo "$OWNER/$REPO" --json number,title,labels,body,assignees
```

Filter locally to exclude issues with `needs-input` label.

## Issue Triage

For each assigned issue, determine:
- **Actionable (Simple)**: You have enough context and the issue is straightforward. Proceed directly.
- **Actionable (Complex)**: The issue requires significant work or multiple steps. Make a plan first, then work through it.
- **Blocked**: Needs user input or clarification. Label it `needs-input` and skip. Add a comment explaining what's needed.

### Labeling Blocked Issues

```bash
gh issue edit $NUMBER --add-label needs-input --repo "$OWNER/$REPO"
gh issue comment $NUMBER --body "I'm blocked on this issue and need your input: [explain what information or decision is needed]" --repo "$OWNER/$REPO"
```

## Processing Simple Issues

For issues you can resolve directly:
1. Add a comment: "I'm working on this issue now."
2. Take action (make changes, create files, etc.)
3. Add a comment summarizing what was done
4. Remove `needs-input` label if present

## Processing Complex Issues

For issues requiring a plan:
1. Add a comment: "I'm analyzing this issue and will provide a plan shortly."
2. Create a plan for addressing the issue
3. Present the plan as a comment on the issue for the reporter to review
4. Wait for their feedback or approval before proceeding
5. If they approve, execute the plan and report back

If the reporter doesn't respond or the issue needs their input to proceed:
1. Label the issue `needs-input`
2. Comment explaining what's needed
3. Skip and move to the next issue

## Never Close Issues

**Never close issues.** The user remains solely responsible for closing. Your job is to propose solutions, provide updates, and advance work.

## Progress Tracking

When you start work on an issue, add a comment:
```
I'm looking into this issue and will provide an update shortly.
```

When you have a solution or finding, add a comment with your proposed approach. If the work is complex, periodically update with progress comments so the user can see what you've tried.

## API Rate Limits

The `gh` CLI handles rate limits automatically. If you encounter errors:
- Wait and retry for transient issues
- Report persistent failures to the user

## Workflow for "Complete all issues assigned to you"

1. **Verify access** — Run `gh auth status` and stop if not authenticated
2. **Cleanup pass** — Remove `needs-input` from closed issues
3. **Fetch assigned issues** — Get open issues assigned to `@me`, excluding `needs-input`
4. **Filter and sort** — Oldest first, skip if already worked recently
5. **Process each issue**:
   - Simple → take action directly
   - Complex → make plan, present to reporter, wait for input if needed
   - Blocked → label `needs-input`, comment, skip
6. **Report summary**

## Summary Output Format

After completing (or partially completing) issues, report:

```
## GitHub Task Summary

User: @username
Repository: owner/repo

Cleanup: removed needs-input from N closed issue(s) (if any)
Processed N issue(s):

| # | Title | Status | Notes |
|---|-------|--------|-------|
| 1 | Issue title | Done / Blocked / Plan Pending | Notes |

Issues needing your input: [list with links]
Issues advanced but not closed: [list with links]
```

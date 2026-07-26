---
name: complete-tasks
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
2. Create a worktree for this issue (see below)
3. Make the necessary changes in the worktree
4. Push the branch and create a PR
5. Add a comment summarizing what was done with a link to the PR
6. Remove `needs-input` label if present

## Making Changes: Worktree and PR Workflow

**Never commit directly to `main` or make changes in the session's working directory.** Always use a dedicated worktree and PR workflow:

### Step 1: Create a Worktree

Create a new worktree with a dedicated branch for the issue. Worktrees are created inside the `.git` directory:
```bash
cd /workspaces/agent-skill-github-task-manager-5a7eadc2
git fetch origin main
git worktree add .git/worktrees/complete-tasks-issue-$NUMBER origin/main
cd .git/worktrees/complete-tasks-issue-$NUMBER
git checkout -b issue/$NUMBER-$short-description
```

Worktrees are created under `.git/worktrees/complete-tasks-issue-$NUMBER`.

### Step 2: Make Changes

Make all necessary code changes in the worktree directory.

### Step 3: Commit and Push

```bash
git add -A
git commit -m "Fix: $TITLE

Closes #$NUMBER

Co-Authored-By: Claude <noreply@anthropic.com>"
git push -u origin issue/$NUMBER-$short-description
```

### Step 4: Create PR

```bash
gh pr create --repo "$OWNER/$REPO" --title "$TITLE" --body "Fixes #$NUMBER

## Summary
[describe what was done]

## Testing
[describe how changes were tested]

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

### Step 5: Update Issue

Add a comment to the issue with the PR link and mark it as `needs-input` to prevent reprocessing:
```bash
gh issue comment $NUMBER --body "I've created a PR for this issue: $PR_URL" --repo "$OWNER/$REPO"
gh issue edit $NUMBER --add-label needs-input --repo "$OWNER/$REPO"
```

### Cleanup Worktrees

After creating the PR, you can remove the worktree:
```bash
cd /workspaces/agent-skill-github-task-manager-5a7eadc2
git worktree remove .git/worktrees/complete-tasks-issue-$NUMBER
```

**Note:** Worktrees created by this skill are stored under `.git/worktrees/complete-tasks-issue-*` inside the repo.

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

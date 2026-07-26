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

## Label Setup

Before any other GitHub operations, ensure the `needs-input` label exists:
```bash
gh label view needs-input --repo "$OWNER/$REPO" 2>/dev/null || gh label create needs-input --repo "$OWNER/$REPO" --color "#FFA500" --description "Issue is blocked and needs user input to proceed"
```

If the label already exists this is a no-op. Do this first — every subsequent operation depends on this label being present.

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

### Step 0: Derive Main Working Directory

Before running any git commands, derive the path to the main working directory (the one with the real `.git` folder):
```bash
git fetch origin main
MAIN_TREE=$(git worktree list --porcelain | grep -m1 "^worktree " | sed 's/^worktree //')
```

From the main tree, you can run `git worktree list` to see all worktrees and perform operations.

### Step 0b: Check for Existing PR

Before creating a new worktree, always check if a PR already exists for this issue to avoid duplicates:
```bash
# Check GitHub for an open or closed PR referencing this issue
EXISTING_PR=$(gh pr list --repo "$OWNER/$REPO" --state all --json number,title,headRefName --jq '.[] | select(.title | contains("#$NUMBER"))')
if [ -n "$EXISTING_PR" ]; then
  PR_NUMBER=$(echo "$EXISTING_PR" | jq -r '.number')
  PR_BRANCH=$(echo "$EXISTING_PR" | jq -r '.headRefName')
  echo "Found existing PR #$PR_NUMBER ($PR_BRANCH)"
  # Check if the branch exists locally as a worktree
  if git worktree list | grep -q "complete-tasks-issue-$NUMBER"; then
    echo "Worktree already exists at .git/worktrees/complete-tasks-issue-$NUMBER"
    cd .git/worktrees/complete-tasks-issue-$NUMBER
    git checkout "$PR_BRANCH"
  elif git branch -r | grep -q "origin/$PR_BRANCH"; then
    # Branch exists remotely but no local worktree — reuse it
    echo "Reusing remote branch $PR_BRANCH"
    git worktree add .git/worktrees/complete-tasks-issue-$NUMBER "origin/$PR_BRANCH"
    cd .git/worktrees/complete-tasks-issue-$NUMBER
  else
    echo "PR #$PR_NUMBER exists but branch $PR_BRANCH is gone — skipping this issue"
    exit 0
  fi
  # Skip to Step 2 with existing branch
else
  echo "No existing PR found — proceeding to create worktree"
fi
```

If a PR was found and reused, skip the worktree creation and go directly to Step 2 (Make Changes).

### Step 1: Create a Worktree

Create a new worktree with a dedicated branch for the issue:
```bash
cd "$MAIN_TREE"
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

# Rebase onto latest main to avoid merge conflicts from other merged PRs
git fetch origin main
git rebase origin/main
# If there are merge conflicts: resolve them, then:
# git rebase --continue

# Push with force-with-lease (safer than --force)
git push --force-with-lease origin issue/$NUMBER-$short-description
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
git worktree remove .git/worktrees/complete-tasks-issue-$NUMBER
```

**Note:** Worktrees created by this skill are stored under `.git/worktrees/complete-tasks-issue-*` inside the repo.

## Stale Worktree Cleanup

Periodically check for stale worktrees — branches whose remote counterpart no longer exists or whose PR was merged and closed:
```bash
git fetch --prune origin
git worktree list
```

For each worktree under `complete-tasks-issue-*` whose branch has been merged and deleted from remote:
```bash
git worktree remove .git/worktrees/complete-tasks-issue-$NUMBER --force
```

Use `--force` if the worktree directory has untracked files from a previous session.

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

1. **Sync main tree** — Pull the latest `main` branch into the main working tree:
   ```bash
   git fetch origin main
   git pull origin main
   ```
2. **Verify access** — Run `gh auth status` and stop if not authenticated
3. **Cleanup pass** — Remove `needs-input` from closed issues
4. **Fetch assigned issues** — Get open issues assigned to `@me`, excluding `needs-input`
5. **Plan the sequence** — Do not process issues in FIFO or LIFO order. Review the full set as a batch and decide the best execution order. See [Planning the Sequence](#planning-the-sequence) below.
6. **Process each issue in the planned order**:
   - Simple → take action directly
   - Complex → make plan, present to reporter, wait for input if needed
   - Blocked → label `needs-input`, comment, skip
7. **Report summary** including the planned sequence and the rationale for the chosen order

## Planning the Sequence

After fetching the assigned issues, pause before writing any code. Review the entire batch as a single unit and decide the order to execute them in. FIFO (oldest first) and LIFO (newest first) are both wrong defaults — they ignore the actual relationships between issues.

For each issue, gather at minimum:
- **Title and body** — what the work actually is
- **Scope** — single file/area vs. cross-cutting refactor
- **Dependencies** — does this issue reference other issues, branches, or PRs? Does it block or get blocked by another?
- **Open PRs** — is there already an in-flight PR for this issue? If so, prefer advancing that PR over starting new work.
- **Risk** — does it touch shared infrastructure, security-sensitive code, or release-blocking paths?
- **Complexity** — does it require a multi-step plan or expert review before code is written?

Then choose an order. Heuristics, in order of priority:

1. **Pick up where another PR left off** — if an existing PR for an issue already has reviewer feedback, address that first. Stale PRs are the highest-priority in-flight work.
2. **Unblock others first** — if issue A blocks issue B, do A first (and surface the dependency in the plan).
3. **Foundational before dependent** — refactors or shared infrastructure changes that other issues rely on go before the consumers.
4. **Small, low-risk wins first** — fast, isolated, well-understood issues can be resolved quickly and reduce noise. Use this to chip away at the queue when nothing else dictates order.
5. **High-risk last** — changes that touch many files or carry merge-conflict risk should run after the smaller ones, so rebase interaction is minimized.
6. **Group by area** — issues that touch the same module or file are best done together, so the second one benefits from the first's worktree state and the diff stays coherent.

State the planned sequence with one-line reasoning per issue **before** you start any work. Example:

```
Planned sequence:
1. #14 (PR has reviewer feedback — address first)
2. #11 (foundational, unblocks #12)
3. #9 (small, isolated)
4. #8 (touching the same area as #9, do together)
5. #5 (cross-cutting, run last to minimize rebase churn)
```

The order is a plan, not a contract. If partway through you discover an issue is more complex than it looked, or a dependency resolves itself, re-plan the remainder and note the change in the final summary.

## Summary Output Format

After completing (or partially completing) issues, report:

```
## GitHub Task Summary

User: @username
Repository: owner/repo

Cleanup: removed needs-input from N closed issue(s) (if any)

Planned sequence (and rationale):
1. #14 — PR has reviewer feedback, address first
2. #11 — foundational, unblocks #12
3. #9 — small, isolated

Processed N issue(s):

| # | Title | Status | Notes |
|---|-------|--------|-------|
| 1 | Issue title | Done / Blocked / Plan Pending | Notes |

Issues needing your input: [list with links]
Issues advanced but not closed: [list with links]
```

If the order changed mid-run, call that out explicitly.

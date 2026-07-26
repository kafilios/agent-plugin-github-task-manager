# github-task-manager Skill Distribution

This repository is the distribution source for the `github-task-manager` Claude skill.

# CLAUDE.md

## Skills

- Skills live in `.claude/skills/<skill-name>/SKILL.md`
- This repo is the distribution source for the `github-task-manager` skill

## Skill Development

- Test skill prompts in `/tmp/<skill-name>-workspace/iteration-N/`
- Use `git remote -v` to re-derive repo context per invocation

## GitHub Integration

- Use `git remote -v` to derive `owner/repo` for GitHub API calls
- Use `gh` CLI (available and authenticated in this environment)
- Authentication via `GITHUB_TOKEN` environment variable

## Git Commit Gotchas

- Sandbox blocks git operations by default — use `dangerouslyDisableSandbox: true`
- GitHub rejects pushes with private email addresses — use `users.noreply.github.com` format
- `user.name`/`user.email` config overrides `GIT_AUTHOR_*` env vars

## github-task-manager Skill

- Label `needs-input` marks issues blocked on user input
- Cleanup pass removes `needs-input` labels from closed issues
- Never closes issues — user is solely responsible for closing

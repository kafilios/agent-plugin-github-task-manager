# CLAUDE.md

## Skills

- Skills live in `.claude/skills/<skill-name>/SKILL.md`
- This repo is the distribution source for the `github-task-manager` skill

## GitHub Integration

- Use `git remote -v` to derive `owner/repo` for GitHub API calls
- Use `curl` + GitHub REST API — `gh` CLI is not available in this environment
- Authentication via `GITHUB_TOKEN` environment variable

## github-task-manager Skill

- Label `needs-input` marks issues blocked on user input
- Cleanup pass removes `needs-input` labels from closed issues
- Never closes issues — user is solely responsible for closing

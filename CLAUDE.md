# github-task-manager Plugin Source

Source repo for the `github-task-manager` Claude plugin. The dev
marketplace (declared in `.claude-plugin/marketplace.json`) is named
`dev`. The published distribution lives in `midnightideas/agent-plugins`
(catalog name `midnightideas`) and is updated by `./scripts/publish`.

# CLAUDE.md

## Skills

- Skills live in `plugins/github-task-manager/skills/<skill-name>/SKILL.md`
- This repo is the source for the `github-task-manager` plugin (dev marketplace: `dev`)

## Skill Development

- Test skill prompts in `/tmp/<skill-name>-workspace/iteration-N/`
- Use `git remote -v` to re-derive repo context per invocation

## Plugin Distribution

- Plugin lives at `plugins/<name>/.claude-plugin/plugin.json` + `plugins/<name>/skills/`
- Dev marketplace catalog at `.claude-plugin/marketplace.json` (`name: "dev"`, lists each plugin with `source: "./plugins/<name>"`)
- `.claude/settings.json` is tracked in this repo and points `enabledPlugins` at `github-task-manager@dev`
- For published distribution, swap the `directory` source for `git` (URL or `github` repo)
- Plugins are not auto-enabled by being declared in a marketplace; `enabledPlugins: true` is required

## Publishing

- `./scripts/publish` clones `midnightideas/agent-plugins` into `.worktrees/publish-<ts>/` (gitignored), copies `plugins/<slug>/.claude-plugin/plugin.json` and `plugins/<slug>/skills/*`, commits, and pushes
- Bump `version` in `plugins/<slug>/.claude-plugin/plugin.json` manually before publishing
- Test publishes: `PUBLISH_MARKETPLACE_BRANCH=test-publish-$(date +%s) ./scripts/publish`
- The script does NOT edit the marketplace catalog on `midnightideas/agent-plugins` — that is hand-curated

## Repo prerequisites

- `.worktrees/` must be in `.gitignore` (the publish script writes temp clones there)

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
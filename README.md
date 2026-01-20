# `bugbot-loop`

Claude Code plugin that resolves Cursor Bugbot PR comments in a loop.

## Install

First, in Claude Code run:

```
/plugin marketplace add ricardonunez-io/bugbot-loop
```

Then run:

```
/plugin install bugbot-loop@bugbot-loop
```

## Usage

On a PR branch with Bugbot comments:

```
/bugbot-loop:run
```

Set max iterations:

```
/bugbot-loop:run --max-iterations=5
```

### Prerequisites

- [GitHub CLI](https://cli.github.com)
- Permissions enabled for GitHub CLI commands:
  - `gh api graphql`
  - `gh repo view`
  - `gh pr view`
  - `gh pr checks`
- Permission enabled for `sleep` shell command
- Permission enabled for committing and pushing (only if auto-commit is enabled)

## Modes

- **Auto-commit**: Commits, pushes, waits for Bugbot, loops until resolved
- **Manual**: Fixes current comments, stops for user review

## How it works

Uses the `gh` CLI with some predefined GraphQL queries to look for unresolved comments left by Bugbot.

Once it finds the unresolved comments, it addresses them by either fixing the bugs or ignoring them if they're invalid reports, marking the comment as resolved after (only in auto-commit mode after pushing).

## Uninstall

```
/plugin uninstall bugbot-loop@bugbot-loop
/plugin marketplace remove bugbot-loop
```

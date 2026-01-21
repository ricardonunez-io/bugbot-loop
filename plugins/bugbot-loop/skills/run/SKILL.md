# Bugbot Loop

Resolve Cursor Bugbot PR comments in a loop until all are addressed.

## Arguments

$ARGUMENTS

Supported flags:

- `--max-iterations=N` - Maximum loop iterations (default: 10)
- `--auto` - Run in auto-commit mode without prompting
- `--manual` - Run in manual mode without prompting

## Instructions

You are resolving Cursor Bugbot comments on the current PR.

### Phase 0: Setup

First, verify you're on a PR branch:

```bash
gh pr view --json number,title,headRefName,url
```

If not on a PR branch, inform the user and stop.

Then, if you have no context on what the changes are that the user made, run `gh pr diff` to get the diff of the PR.

There are two modes in which you should operate:

- Auto-commit mode: You will commit, push, and loop automatically until all comments are resolved
- Manual mode: You will fix all current comments, then stop and notify the user to review/commit manually

If the `--auto` flag is present in the user's prompt, use auto-commit mode without asking the user anything.

If the `--manual` flag is present in the user's prompt, use manual mode without asking the user anything.

Otherwise, use the AskUserQuestion tool to ask which mode the user wants to use with a description of what each mode does.

### Phase 1: Wait for Bugbot CI (auto-commit mode only)

If in auto-commit mode, poll for Bugbot CI completion:

1. Run:
   ```bash
   gh pr checks --json name,state --jq '.[] | select(.name == "Cursor Bugbot")'
   ```
2. If state is "SUCCESS", "FAILURE", or "NEUTRAL", proceed to Phase 2
3. If state is "PENDING" or no result, wait by running:
   ```bash
   sleep 60
   ```
   Then check again. Note down the attempt number somewhere.
4. After 25 retries (25 minutes), stop and inform the user of timeout

If not in auto-commit mode, skip waiting and proceed directly to Phase 2.

### Phase 2: Fetch Unresolved Comments

Get PR info:

```bash
gh pr view --json number -q '.number'
gh repo view --json nameWithOwner -q '.nameWithOwner'
```

Then fetch unresolved Bugbot comments via GraphQL. Use `gh api graphql` with this query (substitute `OWNER`, `NAME`, `PR_NUMBER`):

```graphql
query {
  repository(owner: "OWNER", name: "NAME") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes {
              id
              databaseId
              author {
                login
              }
              path
              line
              body
            }
          }
        }
      }
    }
  }
}
```

Filter results for:

- `isResolved == false`
- Author login is "cursor-bot" or "cursor[bot]"

Extract for each comment: `threadId` (the thread's `id`), `path`, `line`, `body`.

If no unresolved comments, you're done.

### Phase 3: Process Each Comment

For each comment:

1. Read the file at the specified path and line to understand context
2. Analyze the Bugbot suggestion
3. Either:
   - **Fix**: Make the suggested change if it's valid
   - **Dismiss**: If the suggestion is incorrect or not applicable, note why

Track processed comment IDs and their thread IDs to avoid reprocessing.

**Do NOT resolve threads yet** - this happens after successful commit in auto-commit mode only.

### Phase 4: Commit and Continue

**If auto-commit mode:**

1. Commit all fixes with a message describing what was fixed (no coauthor, do not add Claude or Claude Code as coauthor under any circumstances)
2. Push commits: `git push`
3. After successful push, resolve all processed threads via GraphQL mutation using `gh api graphql` (substitute `THREAD_ID`):

```graphql
mutation {
  resolveReviewThread(input: { threadId: "THREAD_ID" }) {
    thread {
      id
      isResolved
    }
  }
}
```

4. Increment iteration counter
5. If iterations >= max-iterations, stop and report status
6. Otherwise, loop back to Phase 1

**If not auto-commit mode:**

1. Do not commit, push, or resolve threads
2. Report to the user:
   - Files modified and changes made
   - Comments that were dismissed (with reasons)
   - Instruct user to review changes, commit, push, and re-run `/bugbot-loop:run` after Bugbot runs again

### Completion

When no unresolved comments remain, report:

- Total comments resolved
- Any comments that were dismissed (with reasons)
- Final iteration count (if auto-commit mode)

## Important Notes

- Always read file context before making changes
- Never add coauthor lines to commits
- If a fix introduces new issues, the next Bugbot run will catch them
- The loop continues until Bugbot is satisfied or max iterations reached (auto-commit mode only)

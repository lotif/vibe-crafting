# `/spec pr` — Address PR Feedback

Address review feedback from a GitHub pull request. Triage every unresolved comment with the user before any code changes. After they confirm, this command reuses the standard implement loop (coding agent → CR → commit) for the approved comments, with feedback sourced from GitHub instead of the local CR agent. Deferred comments become GitHub issues. Pushed-back comments get a reply and no code change.

## Manager Role

**You are a manager. You do NOT write code, review code, run tests, or make technical decisions — ever.** If you catch yourself about to edit a file or run a test, stop. You are in the wrong role. Your only tools are: spawning sub-agents, resuming sub-agents, running git/gh commands, and outputting progress blocks.

The manager's responsibilities:
- Discover the PR and fetch feedback
- Triage every comment and wait for the user to confirm the plan
- Open a GitHub issue for each deferred comment
- Spawn coding sub-agents and CR sub-agents at the right times
- Route CR feedback back to the coding agent
- Verify that commits actually landed (via `git status`)
- Push changes and post replies to GitHub
- Surface summaries and roadblocks to the user
- Send minimal, well-structured prompts that point to reference files — not restate their content

**Important** even if asked to do work by the user, default to using sub-agents per these instructions, unless the user specifically requests you do it in this context! You are a manager: delegate.

## Progress Tracker

→ Read [references/shared/progress_tracker.md](shared/progress_tracker.md) for the progress block format, round counters, and rules. Follow them precisely.

Use the label **"PR Feedback Progress"** for the progress block. The full step list for this command:

```
- Step 0a: Find PR
- Step 0b: Fetch comments
- Step 0c: Triage and confirm
- Step 0d: Deferral issues and early replies
- Step 1: Coding
- Step 1b: Attestation
- Step 1c: UI signoff (if applicable)
- Step 2: Code review
- Step 3: Commit
- Step 4: Verify and push
- Step 5: Reply to implemented comments
- Step 6: Summary
```

## Invocation

```
/spec pr
/spec pr <pr-url-or-number>
```

An argument is optional. A GitHub pull request URL or a PR number selects that PR. With no argument, the command discovers the PR from the current branch.

## Step 0a: Find the PR

If the user passed a URL or number, run:

```bash
gh pr view <pr-url-or-number> --json number,url,title,headRefName
```

If they did not, run:

```bash
gh pr view --json number,url,title,headRefName
```

If no PR is found:

> No open PR found. Pass a PR URL or number, or push the current branch (`[branch name]`) and open a PR first.

A PR number resolves in the current repo. If the user passed a URL, the `owner/repo` in that URL must match `gh repo view --json nameWithOwner`. If it does not, stop:

> PR #[number] is in `[owner/repo]`, and this checkout is `[nameWithOwner]`. Open the command from that repo.

Do not compare `headRepository`. That is the head branch's repo, and it differs for pull requests opened from a fork.

Before any later coding step, the checkout must be the PR's head branch.

- If `git branch --show-current` already matches `headRefName`, continue.
- If the working tree is clean, check out that branch with `gh pr checkout {number}`.
- If the working tree is dirty, stop and tell the user. Do not check out over local changes.

Also extract `{owner}` and `{repo}` for subsequent API calls.

## Step 0b: Fetch PR Comments

Use the GraphQL API to fetch review threads with resolution status, and the REST API for general issue comments:

**Review threads (with resolution status):**

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            comments(first: 50) {
              nodes {
                id
                databaseId
                url
                author { login }
                body
                path
                line
              }
            }
          }
        }
      }
    }
  }
' -f owner='{owner}' -f repo='{repo}' -F number={number}
```

Filter out threads where `isResolved` is `true`.

**General issue comments:**

```bash
gh api repos/{owner}/{repo}/issues/{number}/comments --paginate
```

If no unresolved comments exist:

> No unresolved feedback on PR #[number]. Nothing to address.

### Format Feedback

Organize the feedback into a structured block for the coding agent:

```
<pr_feedback>
## PR #[number]: [title]
URL: [pr_url]

### Review Comments (line-level)

**[file]:[line]** — @[author] (comment_id: [databaseId])
Link: [url]
> [comment body]

**[file]:[line]** — @[author] (comment_id: [databaseId])
Link: [url]
> [comment body]

### General Comments

@[author] (comment_id: [id])
Link: [html_url]
> [comment body]
</pr_feedback>
```

`comment_id` is the numeric REST id: `databaseId` for review comments, `id` for issue comments. Reply APIs take that id, not the GraphQL node id.

`Link` is the comment's HTML URL (`url` from GraphQL, `html_url` from the issue-comments API). If the API omits it, build it:

- Review comment: `https://github.com/{owner}/{repo}/pull/{number}#discussion_r{databaseId}`
- Issue comment: `https://github.com/{owner}/{repo}/pull/{number}#issuecomment-{id}`

Keep the link. Triage, issues, and replies all use it.

### Context Loading

Check if there's an active project or task:

1. Read `.specs_skill_state/current_project.md` if it exists
2. If an active project exists, the coding agent gets spec context
3. If an active task exists, the coding agent gets task context
4. If neither exists, the coding agent works from code context alone

This is informational — the command works regardless of active project state.

## Step 0c: Triage and Confirm

Classify every unresolved comment into exactly one bucket. Do this before any code, issue, or reply.

- **Implement** — fix it in this run.
- **Defer** — do not fix it in this run. Use this when the comment is a lot of work or belongs in a follow-up. The default action is to open a GitHub issue. Include a concise reason.
- **Push back** — do not fix it, and do not open an issue. Use this when you disagree with the comment, or when implementing it would change behavior significantly. Include a concise reason.

Show the triage in this shape. Every decision has a link to the original comment. Defer and push back include the reason. Keep empty buckets visible.

```
## PR #[number] triage: [title]

[pr_url]

### Implement ([N])

- [one-line summary] — [comment link]

### Defer ([N])

Each deferred item opens a GitHub issue unless you say otherwise.

- [one-line summary] — [comment link]
  Why: [concise reason]

### Push back ([N])

- [one-line summary] — [comment link]
  Why: [concise reason]

Is everything good to go?
```

Stop here. Do not code, open issues, reply, commit, or push until the user confirms.

- A plain confirmation ("yes", "good to go", "lgtm") locks the plan.
- A confirmation that also corrects the plan ("yes, but move X to defer", "don't open an issue for Y") locks the corrected plan. Apply the correction and proceed. Do not ask again.
- A correction with no confirmation ("move X to defer") updates the lists. Show the updated triage and ask again.

The user can also edit a reason, or tell you not to open an issue for a specific deferred comment. Honor that when the plan locks.

## Step 0d: Deferral Issues and Early Replies

Run this only after the user confirms. Skip it when nothing was deferred or pushed back.

**Issues.** For each deferred comment whose action is still "open a GitHub issue":

```bash
gh issue create --title "Follow-up from PR #[number]: [one-line summary]" --body "$(cat <<'EOF'
Deferred from PR #[number] while addressing review feedback.

**Comment:** [comment link]
**Author:** @[author]
**Where:** [file:line, or "general comment"]

> [comment body]

**Why deferred:** [concise reason]

PR: [pr_url]
EOF
)"
```

Record each issue URL. If issue creation fails, stop and tell the user. Do not start coding with an untracked deferral.

**Replies.** Reply now on deferred and pushed-back threads, before coding starts, so those decisions are on the PR even if implementation later hits a roadblock.

Deferred, with an issue:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body="$(cat <<'EOF'
Deferred — tracked in [issue url].

[concise why]

---
*Deferred via `/spec pr`*
EOF
)"
```

Deferred, when the user said not to open an issue:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body="$(cat <<'EOF'
Deferred.

[concise why]

---
*Deferred via `/spec pr`*
EOF
)"
```

Pushed back:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body="$(cat <<'EOF'
Not implementing this.

[concise why]

---
*Reviewed via `/spec pr`*
EOF
)"
```

For a general issue comment, post a new issue comment on the PR instead of a review-thread reply (same shape as Step 5). A failed reply does not block coding. Report it in the Step 6 summary and continue.

If the confirmed plan has nothing to implement, skip Steps 1 through 5 and go to Step 6.

## Implementation Flow

**PROCESS GATE:** Before proceeding to Step 1, verify:
1. Steps 0a through 0c are done, and the user confirmed the triage
2. Step 0d is done for every deferred or pushed-back comment (or there were none)
3. The confirmed plan has at least one Implement item. If it does not, Step 0d already sent you to Step 6
4. You are about to output your first progress block
5. You have NOT written any code or edited any project files yourself
6. Your next action after the progress block is spawning a sub-agent

If any of these are false, stop and correct course.

**AUTONOMOUS FLOW: Once the user has confirmed the triage, drive the rest of the flow to completion without stopping for user input. The only exceptions are: escalation (roadblock from coding agent), and UI signoff (Step 1c) when the changes include significant UI changes.**

### Step 1: Spawn Coding Agent

Output your first progress block, then spawn a new coding sub-agent using the PR Feedback Coding Prompt template below.

→ Read [references/spawning_subagents.md](references/spawning_subagents.md) for how to spawn sub-agents.

The coding agent returns either:
- A summary with an attestation block indicating it's ready for code review
- A roadblock message (see Escalation below)

### Step 1b: Validate Attestation

Inspect the coding agent's return for an `<attestation>` block.

- If the block is **missing**, or any value is **FALSE**: resume the coding agent with:

  > Your return summary is missing the required `<attestation>` block, or not all items are TRUE. Review your workflow instructions, ensure all checks and tests pass, and return your summary with a complete attestation block.

- If all values are **TRUE** (or NA where appropriate): proceed to Step 1c.

Do NOT run checks yourself — the coding agent is responsible. You are verifying it reported completion.

### Step 1c: UI Signoff

If the PR feedback changes involve **significant UI changes** (new screens, layout changes, new components, visual redesigns), pause and ask the user to review the UI before proceeding to code review.

**Skip this step when:**
- The changes have no UI impact
- UI changes are trivial (copy/string updates, minor spacing tweaks)

**When UI signoff is needed:**
1. Tell the user the coding agent has finished and the UI is ready for review
2. Summarize what UI was built/changed (based on the coding agent's summary)
3. Ask the user to check the UI and confirm it looks good, or describe what needs to change
4. If the user requests changes: resume the coding agent with the feedback, then go back to **Step 1b** (validate attestation) and return here for another signoff
5. If the user approves: proceed to Step 2

This step exists because code review catches code quality issues, not design issues. Human eyes should approve significant UI before the CR/fix/commit loop begins.

### Step 2: CR Loop

Standard CR loop — identical to `/spec implement` and `/spec task`.

1. Spawn a fresh CR sub-agent using the CR Agent Prompt template below
2. CR agent returns structured feedback with severity labels
3. If the review is clean: proceed to Step 3
4. If issues exist:
   - Resume the coding agent with the CR Feedback Prompt template, passing the CR output
   - Coding agent addresses issues and returns a summary with attestation block
   - Validate attestation (same as Step 1b — resume coding agent if missing or FALSE)
   - Spawn a new CR sub-agent, passing prior feedback in a `<prior_cr_feedback>` block
   - Repeat until CR returns clean

→ Read [references/spawning_subagents.md](references/spawning_subagents.md) for how to spawn sub-agents.

### Step 3: Commit

**PROCESS GATE — No commit without clean CR:** Before proceeding to Step 3, verify:
1. The most recent sub-agent action was a CR that returned clean
2. No code has been written or changed since that clean CR
3. You did NOT skip re-review after the coding agent addressed CR feedback

If any of these are false, you must run (or re-run) the CR loop before committing. Every code change — including CR fixes — requires a clean CR before commit.

Resume the coding agent with the Commit Prompt template below. The coding agent commits all changes and returns the commit message and hash.

If the coding agent returns a pre-commit hook failure instead of a commit message:

1. Resume the coding agent to fix the issues reported by the hook
2. When it returns, go back to **Step 1b** (validate attestation) and then **Step 2** (CR loop)
3. Only tell it to commit again after attestation and CR both pass

Do NOT tell it to commit immediately after fixing — the fix is unreviewed code.

### Step 4: Verify and Push

Run `git status` to confirm:
- Working tree is clean (no uncommitted changes)
- The commit exists

If `git status` shows uncommitted changes, resume the coding agent:

> Commit appears incomplete — `git status` shows uncommitted changes. Please commit all changes.

Verify again after.

Once verified:

1. Run `git rev-parse HEAD` to capture the commit hash
2. Run `git push` to push the changes

### Step 5: Reply to Implemented Comments

Deferred and pushed-back threads were already answered in Step 0d. Reply here only for comments in the confirmed Implement set.

**For review comments (line-level):**

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body="$(cat <<'EOF'
Fixed in [COMMIT_HASH].

[1-2 sentence description of how it was resolved]

---
*Addressed by AI coding agent via `/spec pr`*
EOF
)"
```

**For general issue comments:**

```bash
gh api repos/{owner}/{repo}/issues/{number}/comments \
  -f body="$(cat <<'EOF'
Re: @[author]'s [feedback summary]

Fixed in [COMMIT_HASH].

[1-2 sentence description of how it was resolved]

---
*Addressed by AI coding agent via `/spec pr`*
EOF
)"
```

**Important:**
- Do NOT use `gh` to resolve comment threads — that's reserved for humans
- Step 0d replies to deferred and pushed-back comments. This step replies to implemented ones. Every comment from the triage ends up with a reply.
- Keep reply descriptions concise but specific

### Step 6: Present Summary

Show a summary to the user:

> **PR #[number] feedback addressed**
>
> **Commit:** [hash] — [commit message]
> **Push:** ✓ pushed to [branch]
>
> **Triage:** [N] implement, [N] defer, [N] push back
> - Fixed: [N]
> - Deferred: [N] ([issue links, or "no issue" when the user opted out])
> - Pushed back: [N]
>
> **Replies posted:** [N] / [total]
>
> When nothing was implemented, omit the commit and push lines and say that the confirmed plan had no code changes.

## Prompt Templates

These are the exact prompts the manager sends to sub-agents. Use them verbatim, filling in the bracketed values.

### PR Feedback Coding Prompt

```
You are a coding agent addressing PR feedback from GitHub.

The user already confirmed this set. Implement every comment in it. Deferred and pushed-back comments are out of scope.

[IF active project]: **Project specs:** [specs/projects/PROJECT_NAME/]
[IF active task]: **Task file:** [.specs_skill_state/tasks/SLUG.md]

Read `references/pr_coding_prompt.md` for your full instructions. Follow them precisely.

[The formatted <pr_feedback> block — confirmed Implement comments only]

Return a short summary of what you changed when ready for code review. If an item conflicts with the spec, would change behavior beyond the triage, or is much larger than it looked, return a roadblock instead of skipping it.
```

### CR Feedback Prompt (resume coding agent)

```
A code reviewer found issues with your implementation. Address all feedback below, then run automated checks until clean.

Return a short summary of changes made when ready for re-review.

<cr_feedback>
[CR agent's output]
</cr_feedback>
```

### Commit Prompt (resume coding agent)

```
Your code has passed review. Commit all changes with a descriptive message summarizing the PR feedback you addressed.

Return the commit message and the commit hash (run `git rev-parse HEAD` after committing).
```

### CR Agent Prompt

```
Review code changes addressing PR feedback.
[IF active project]: The project is at [specs/projects/PROJECT_NAME/].
[IF active task]: The task is described in [.specs_skill_state/tasks/SLUG.md].

The coding agent was addressing external PR feedback the user already confirmed. Here is that implement set:

[The formatted <pr_feedback> block — confirmed Implement comments only]

These comments were deferred or pushed back. They are out of scope. Do not treat them as missing work:

[One line per deferred or pushed-back comment: summary — link — bucket]

Read `references/cr_agent_prompt.md` for your full review instructions. Follow them precisely.
```

For re-reviews, append:

```
<prior_cr_feedback>
[Previous CR output]
</prior_cr_feedback>
```

## Escalation

The coding agent may surface a technical roadblock instead of a "ready for CR" summary. When the manager receives a roadblock message:

1. Present the roadblock to the user and wait for a decision
2. Resume the coding agent with the user's decision
3. Continue the flow from wherever the coding agent left off

## References

- [references/spawning_subagents.md](references/spawning_subagents.md) — How to spawn and resume sub-agents
- [references/pr_coding_prompt.md](references/pr_coding_prompt.md) — Full instructions for PR feedback coding sub-agents
- [references/shared/coding_workflow.md](shared/coding_workflow.md) — Shared workflow for coding agents
- [references/shared/coding_role.md](shared/coding_role.md) — Coding agent role and persona
- [references/cr_agent_prompt.md](references/cr_agent_prompt.md) — Full instructions for CR sub-agents

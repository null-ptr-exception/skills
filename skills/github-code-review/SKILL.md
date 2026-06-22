---
name: github-code-review
version: 0.2.0
description: Opinionated PR code review — runs the official /code-review plugin, then adds DESIGN and TESTS follow-up layers via parallel agents. Always discusses findings with the user before posting to GitHub. Use when the user asks to "review a PR", "review PR #N", or wants a thorough code review with design and test coverage analysis.
source: https://github.com/null-ptr-exception/skills/tree/main/skills/github-code-review
allowed-tools: Bash(gh *), Bash(git *)
---

# Opinionated PR Code Review

Run the official code review plugin, then add DESIGN and TESTS follow-up analysis. Nothing posts to GitHub without human approval.

## Arguments

The skill argument should be a PR number (e.g. `42` or `#42`). If no argument is given, ask the user which PR to review.

## Setup: Resolve PR metadata

Before starting, resolve the PR's owner/repo and commit SHA dynamically:

```bash
PR_URL="$(gh pr view <PR> --json url -q .url)"
# Extract owner/repo from URL: https://github.com/{owner}/{repo}/pull/{n}
OWNER_REPO="$(echo "$PR_URL" | sed 's|https://github.com/||; s|/pull/.*||')"
COMMIT_SHA="$(gh pr view <PR> --json headRefOid -q .headRefOid)"
```

Use `$OWNER_REPO` and `$COMMIT_SHA` in all subsequent API calls.

## Phase 1: Official Code Review

Run the official `/code-review` plugin against the PR. Do NOT pass `--comment` or `--fix` — findings must stay in-session.

```
Invoke Skill tool: code-review
Args: <PR number>
```

Capture the findings. Do NOT post anything to GitHub yet.

## Phase 2: Follow-up Agents (parallel)

After Phase 1 completes, spawn two agents in parallel using the Agent tool.

### DESIGN Agent

Prompt the agent with the diff (use `gh pr diff <PR>`) and the official review findings from Phase 1. The agent checks three dimensions:

1. **Abstraction depth** — Is the fix at the right level? Are special cases being layered where the underlying mechanism should be generalized? Should inline logic be extracted into a shared helper?
2. **Cross-file contract safety** — Does the change break unchanged code that depends on the same data format, return shape, or interface? The agent MUST grep for callers and consumers beyond the diff. Bugs in unchanged code broken by the PR's changes are in scope.
3. **Architectural fit** — Does the change fit the system's existing patterns? Does it introduce inconsistencies with how other parts of the codebase handle the same concern?

The agent must read files beyond the diff. It should return findings as a list with file, line, summary, and failure scenario for each.

### TESTS Agent

Prompt the agent with the diff and the PR number. The agent checks five things in order:

1. **Test infra discovery** — Does the project have unit tests? E2E tests? What frameworks (vitest, jest, playwright, etc.)?
2. **CI pipeline existence** — Does the project have CI (GitHub Actions, etc.) that runs tests? Check `.github/workflows/`.
3. **CI pipeline status** — Did CI pass on this PR? Run `gh pr checks <PR>` to verify.
4. **Test coverage of changes** — Does the PR include tests that exercise the new/changed behavior? Read the actual test files, not just check they exist.
5. **Tests masking regressions** — Do existing tests still pass but with fixtures that don't reflect the new on-disk/API reality introduced by the PR? These give false confidence.

The agent should return findings as a list with file, line, summary, and failure scenario for each.

## Phase 2.5: Dedup against existing reviews

After Phase 1+2 findings are collected, before presenting to the user:

1. Fetch existing review comments:
   ```bash
   gh api repos/$OWNER_REPO/pulls/<PR>/comments --paginate
   gh api repos/$OWNER_REPO/pulls/<PR>/reviews --paginate
   ```
2. Match each finding against existing comments (by file, line, or topic).
3. Mark findings as "already covered by {reviewer}" or "novel".

When presenting the numbered summary, annotate covered findings so the user can skip quickly. Only novel findings need detailed discussion.

## Phase 3: Present and discuss findings

Present all findings as a numbered summary organized in three sections:

```
### Official Code Review Findings
<findings from Phase 1, with dedup annotations>

### DESIGN Findings
<findings from DESIGN agent, with dedup annotations>

### TESTS Findings
<findings from TESTS agent, with dedup annotations>
```

Then say: **"Let's go through each finding. I'll show the draft comment — you can approve, skip, or discuss."**

Do NOT ask the user to batch-select which findings to post. Proceed directly into the per-item loop (Phase 4).

## Phase 4: Per-item discussion and posting

### Step 1: Create a pending review

```bash
REVIEW_JSON="$(gh api repos/$OWNER_REPO/pulls/<PR>/reviews -X POST \
  -f commit_id="$COMMIT_SHA")"
REVIEW_ID="$(echo "$REVIEW_JSON" | jq -r '.id')"
REVIEW_NODE_ID="$(echo "$REVIEW_JSON" | jq -r '.node_id')"
```

Save both `REVIEW_ID` (numeric, for REST submit) and `REVIEW_NODE_ID` (for GraphQL comment posting).

### Step 2: Sequential finding loop

For each finding, in order:

1. Show the draft inline comment — include file path, line number, and comment text.
2. The user says **ok** (post it), **skip** (move on), or **discusses** (refine the comment).
3. If approved, post via GraphQL:

```bash
gh api graphql -f query='
mutation($reviewId: ID!, $path: String!, $line: Int!, $body: String!) {
  addPullRequestReviewThread(input: {
    pullRequestReviewId: $reviewId
    path: $path
    line: $line
    side: RIGHT
    body: $body
  }) {
    thread { id }
  }
}' -f reviewId="$REVIEW_NODE_ID" -f path="<file>" -F line=<N> -f body="<comment>"
```

4. Move to the next finding.

### Line number rules

The GitHub API rejects inline comments on lines outside a diff hunk (422 error). Before posting each comment, verify the target line:

- **New files**: all lines are valid — file line number works directly.
- **Modified files**: line must fall within a hunk range. Parse hunk headers from `gh pr diff`: `@@ -old,count +new,count @@` — the line must be in `[new_start, new_start + new_count)`.
- **Line outside all hunks**: use a file-level comment instead:

```bash
gh api graphql -f query='
mutation($reviewId: ID!, $path: String!, $body: String!) {
  addPullRequestReviewThread(input: {
    pullRequestReviewId: $reviewId
    path: $path
    body: $body
    subjectType: FILE
  }) {
    thread { id }
  }
}' -f reviewId="$REVIEW_NODE_ID" -f path="<file>" -f body="<comment>"
```

### Step 3: Submit the review

After all findings are processed, draft the review summary body. The summary should be **concise and focus on architectural concerns** — do not repeat inline comment details, which are already on the PR. Point to inline comments for specifics.

Ask the user to pick the submission type:

- **COMMENT** — neutral observations, does not block merge
- **REQUEST_CHANGES** — blocks merge until resolved
- **APPROVE** — approve with comments

After user confirms, submit:

```bash
gh api repos/$OWNER_REPO/pulls/<PR>/reviews/$REVIEW_ID/events -X POST \
  -f event="<COMMENT|REQUEST_CHANGES|APPROVE>" \
  -f body="<summary>"
```

## Rules

- NEVER post to GitHub without explicit human confirmation
- NEVER pass `--comment` or `--fix` to the official code review plugin
- Always present findings for discussion before posting
- Each inline comment gets its own focused discussion — the user approves, skips, or refines
- The submission type (comment/request changes/approve) is always the human's choice
- The review summary focuses on architecture, not inline details

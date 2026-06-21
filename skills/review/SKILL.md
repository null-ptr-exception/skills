---
name: review
version: 0.1.0
description: Opinionated PR code review — runs the official /code-review plugin, then adds DESIGN and TESTS follow-up layers via parallel agents. Always discusses findings with the user before posting to GitHub. Use when the user asks to "review a PR", "review PR #N", or wants a thorough code review with design and test coverage analysis.
source: https://github.com/null-ptr-exception/skills/tree/main/skills/review
allowed-tools: Bash(gh *), Bash(git *)
---

# Opinionated PR Code Review

Run the official code review plugin, then add DESIGN and TESTS follow-up analysis. Nothing posts to GitHub without human approval.

## Arguments

The skill argument should be a PR number (e.g. `42` or `#42`). If no argument is given, ask the user which PR to review.

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

## Phase 3: Discuss with Human

Present all findings organized in three sections:

```
### Official Code Review Findings
<findings from Phase 1>

### DESIGN Findings
<findings from DESIGN agent>

### TESTS Findings
<findings from TESTS agent>
```

Ask the user which findings to post to GitHub and which to discard. Wait for explicit confirmation before proceeding to Phase 4.

## Phase 4: Post to GitHub (iterative)

### Step 1: Start a pending review

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews -X POST -f commit_id="$(gh pr view <PR> --json headRefOid -q .headRefOid)"
```

Save the returned review ID.

### Step 2: Inline comments loop

For each finding the user approved in Phase 3:

1. Draft the inline comment — include file path, line number, and comment text
2. Present the draft to the user for confirmation
3. If confirmed, post it:
   - If the target line is in the diff, post as a line comment on the review
   - If the target line is NOT in the diff, post with `subject_type: file` for a file-level comment
4. If rejected, skip or revise based on user feedback

### Step 3: Submit the review

Draft the review summary body. Then ask the user to pick the submission type:

- **COMMENT** — neutral observations, does not block merge
- **REQUEST_CHANGES** — blocks merge until resolved
- **APPROVE** — approve with comments

After user confirms, submit:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews/{review_id}/events -X POST \
  -f event="<COMMENT|REQUEST_CHANGES|APPROVE>" \
  -f body="<summary>"
```

## Rules

- NEVER post to GitHub without explicit human confirmation
- NEVER pass `--comment` or `--fix` to the official code review plugin
- Always present findings for discussion before posting
- Each inline comment requires individual human approval
- The submission type (comment/request changes/approve) is always the human's choice

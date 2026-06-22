# github-code-review Skill — Design Notes

Notes from real usage reviewing PRs #42 and #44 on db-runbooks.

## GitHub API: Adding Comments to Pending Reviews

The skill's Phase 4 describes creating a pending review then adding inline comments iteratively. The REST API does NOT support this — `reviews/{id}/comments` returns 404.

**Working approach**: GraphQL `addPullRequestReviewThread` mutation.

### Flow

1. **Create pending review (REST)**:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/reviews -X POST \
     -f commit_id="<sha>"
   ```
   Save both `id` (numeric, for REST submit) and `node_id` (for GraphQL).

2. **Add comments incrementally (GraphQL)**:
   ```bash
   gh api graphql -f query='
   mutation {
     addPullRequestReviewThread(input: {
       pullRequestReviewId: "<node_id>"
       path: "path/to/file.sh"
       line: 42
       side: RIGHT
       body: "comment text"
     }) {
       thread { id }
     }
   }'
   ```
   Repeatable — call once per approved comment.

3. **Submit review (REST)**:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/reviews/{id}/events -X POST \
     -f event="COMMENT|REQUEST_CHANGES|APPROVE" \
     -f body="summary"
   ```

### What Does NOT Work

| Approach | Result |
|---|---|
| `POST reviews/{id}/comments` | 404 — endpoint doesn't exist |
| `POST pulls/{pr}/comments` with `pull_request_review_id` | 422 — not a permitted key |
| Delete + recreate review with more comments | Works but wasteful — must resend all comments each time |

## Improvements to Discuss

### 1. Phase 4: Use GraphQL for incremental comments
Replace the current (broken) REST instructions with the GraphQL flow above. No more delete-and-recreate.

### 2. Phase 3+4: Merge discussion and posting into one sequential loop
Current Phase 3 asks the user to batch-select which findings to post ("which ones?"), then Phase 4 loops through approved findings with a separate draft-confirm step. In practice, the batch selection never happened — the user went straight into sequential discussion.

Proposed: Phase 3 presents all findings as a numbered overview (so the user has the full picture), then says "Let's go through each finding one by one." Phase 4's per-item loop becomes:
1. Show the draft inline comment
2. User says ok / skip / discusses and refines
3. If ok → post immediately via GraphQL
4. Next finding

The upfront "which ones to post?" gate is removed. The per-item approval remains — each comment gets its own focused discussion before posting.

### 3. Dedup step between Phase 2 and Phase 3
After Phase 1+2 findings are collected, run a dedup step before presenting the numbered summary. This step:
1. Fetches all existing review comments on the PR (`gh api repos/{owner}/{repo}/pulls/{pr}/comments`)
2. Fetches all review bodies (`gh api repos/{owner}/{repo}/pulls/{pr}/reviews`)
3. Matches each finding against existing comments — marks findings as "already covered by {reviewer}" or "novel"

The numbered summary in Phase 3 then annotates covered findings so the user can skip quickly. This avoids the manual "did CodeRabbit cover this?" back-and-forth.

Cannot run in parallel with DESIGN/TESTS — it needs those findings as input. But it's a quick step (fetch + string matching) so it adds minimal latency.

### 4. Summary should be concise and focus on architecture
The summary is written at the end (after all per-item discussions), which is fine. But it should NOT repeat inline comment details — those are already on the PR. The summary should focus on high-level architectural concerns and overall assessment. Keep it short; point to inline comments for specifics.

### 5. Line number resolution
The GitHub API rejects inline comments on lines outside a diff hunk (422 error). Every session rediscovers this by trial and error.

Rules:
- **New files**: all lines are valid — file line number works directly
- **Modified files**: line must fall within a hunk range (`[new_start, new_start + new_count)` from `@@ -old,count +new,count @@`)
- **Outside all hunks**: use file-level comment (GraphQL: `subjectType: FILE`, omit `line`)

Decision: Document the rules as prose in the skill for now. 

Future improvement: ship a helper script (e.g. `check-diff-line.sh`) that validates line numbers against the diff and suggests fallbacks. Reduces per-session reimplementation.

### 6. Resolve owner/repo dynamically
The skill hardcodes `{owner}/{repo}` placeholders. Use `gh pr view <PR> --json url -q .url` to extract the actual owner/repo, and `gh pr view <PR> --json headRefOid -q .headRefOid` for the commit SHA. Include these as concrete commands in the skill so sessions don't guess.

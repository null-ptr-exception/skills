---
name: github-actions-cicd
description: This skill should be used when writing, modifying, or reviewing GitHub Actions workflow YAML files. Covers action version pinning, Docker build/push/release patterns, Node.js runtime deprecation, and trunk-based CI/CD design. Use when the user asks to "create a workflow", "fix CI", "update GitHub Actions", "add a release step", "pin action versions", or is editing .github/workflows/*.{yml,yaml} files.
---

# GitHub Actions CI/CD Best Practices

Opinionated rules for writing reliable, maintainable GitHub Actions workflows.

## Action Version Pinning

Pin actions to full commit SHA. Add exact release tag in comment for auditability.

```yaml
# GOOD — SHA for immutability, exact tag for humans
- uses: actions/checkout@9f698171ed81b15d1823a05fc7211befd50c8ae0 # v6.0.3

# BAD — moving tag, can be retargeted
- uses: actions/checkout@v6

# BAD — ambiguous comment, v6 could be v6.0.0 or v6.0.3
- uses: actions/checkout@9f698171ed81b15d1823a05fc7211befd50c8ae0 # v6
```

To find the SHA for a release tag (works for both annotated and lightweight tags — GitHub Actions resolves both correctly):
```bash
gh api repos/{owner}/{repo}/git/ref/tags/{tag} --jq '.object.sha'
```

To find the latest release tag:
```bash
gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
```

## Node.js Runtime Deprecation

GitHub deprecates Node.js versions used by actions on a regular cycle. When CI logs show warnings like `Node.js 20 actions are deprecated`, update all actions to their latest major versions.

Check all actions in a workflow at once:
```bash
grep -E '^\s+uses:' .github/workflows/*.{yml,yaml}
```

Then look up the latest version for each and update both the SHA and the comment tag.

## Docker Build vs Push vs Release

Separate building from publishing. Not every merge to main should trigger a release.

**Pattern: always build, conditionally push and release.**

```yaml
- name: Check if version exists
  id: check
  run: |
    if version_exists_in_registry; then
      echo "exists=true" >> "$GITHUB_OUTPUT"
    else
      echo "exists=false" >> "$GITHUB_OUTPUT"
    fi

- name: Build image
  uses: docker/build-push-action@...
  with:
    push: false
    load: true

- name: Push image
  if: steps.check.outputs.exists == 'false'
  run: docker push ...

- name: Create release
  if: steps.check.outputs.exists == 'false'
  run: ...
```

**Why:** Building always validates the Dockerfile. Pushing and releasing only happen when the version in package.json (or equivalent) is bumped — an intentional release action.

## Skip-on-Condition, Not Fail-on-Condition

When a step is conditional (not every run should execute it), skip it gracefully instead of failing the whole workflow.

```yaml
# GOOD — skip with a message
if version_already_exists; then
  echo "exists=true" >> "$GITHUB_OUTPUT"
  echo "Version already exists — skipping push"
fi

# BAD — fails the entire workflow
if version_already_exists; then
  echo "::error::Version already exists"
  exit 1
fi
```

Reserve `exit 1` for conditions that genuinely indicate a problem (security checks, lint failures, broken tests). Use step-level `if:` conditions for optional work.

## GitHub Release Creation

Use the generate-notes API to get auto-generated changelog, then append custom content.

```yaml
- name: Create release
  run: |
    VERSION=v${{ steps.version.outputs.version }}
    git tag "$VERSION"
    git push origin "$VERSION"
    NOTES=$(gh api repos/${{ github.repository }}/releases/generate-notes \
      -f tag_name="$VERSION" --jq .body)
    printf '%s\n\n## Docker Image\n\n```\ndocker pull %s:%s\n```\n' \
      "$NOTES" "$IMAGE" "${{ steps.version.outputs.version }}" > /tmp/release-notes.md
    gh release create "$VERSION" -F /tmp/release-notes.md
```

**Why not `--generate-notes` flag directly?** You can combine `--generate-notes` with `--notes` to prepend custom text, but the API approach gives full control over composition — e.g. appending a Docker image section after the changelog rather than before it.

## Workflow Structure for Trunk-Based Development

For projects where main is the trunk:

1. **lint-and-test** — always runs (push + PR)
2. **e2e** — always runs, depends on lint-and-test
3. **docker** — only on main, depends on e2e
   - Always build (validates Dockerfile)
   - Only push + release when version is new

```yaml
docker:
  runs-on: ubuntu-latest
  needs: e2e
  if: github.ref == 'refs/heads/main'
  permissions:
    contents: write    # for git tag + release
    packages: write    # for ghcr push
```

**Permissions:** Use `contents: write` (not `read`) when the job creates git tags or GitHub releases.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `push: ${{ github.ref == 'refs/heads/main' }}` combines build+push | Separate into build (always) and push (conditional) steps |
| `contents: read` on a job that creates tags/releases | Use `contents: write` |
| Version check that fails the workflow | Use skip pattern with step outputs |
| Need to append (not prepend) custom content to release notes | Use the API to get notes, then compose |
| Action pinned to `@v6` without SHA | Pin to SHA, add exact tag in comment |

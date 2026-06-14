---
name: fix-cve
description: Scan a container image for CVEs with Docker Scout and patch what can be fixed locally. Use when the user asks to "fix CVEs", "scan for vulnerabilities", or points at a Dockerfile / image tag and wants it hardened. Default scope is CRITICAL only; include HIGH only when the user asks.
---

# Fix CVE

Scan a container image and apply minimal Dockerfile changes to resolve fixable CVEs. Stop before committing — git is out of scope.

## Scope

- **Default severity:** CRITICAL only.
- **Include HIGH** only when the user explicitly asks ("also high", "crit+high", etc.).
- **Scanner:** Docker Scout (`docker scout cves`). If not installed, offer to install via:
  ```
  curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
  ```

## Workflow

### 1. Identify target
- If user gives an image tag → scan directly.
- If user gives a Dockerfile / folder → build first with a local test tag (e.g. `<name>-cve-test`), then scan.

### 2. Scan
```
docker scout cves --only-severity critical <image>
```
Add `high` to severity list only when requested.

### 3. Classify each finding
For every CVE, place it in one of two buckets:

**A. Locally fixable** — the vulnerable package can be upgraded in our Dockerfile:
  - Alpine `apk` package with a fixed version available → `apk upgrade <pkg>` or `apk add <pkg>=<fixed-ver>`
  - Debian `apt` package with a fixed version → `apt-get install -y <pkg>=<fixed-ver>`
  - Tool downloaded via `curl` at a pinned version → bump the pinned version

**B. Upstream-only** — compiled into a vendor binary (Go stdlib, vendored Go/Rust deps inside a prebuilt binary like `gitlab-runner`, `helm`, etc.):
  - Check if the base image / tool is already on its latest upstream release (check Docker Hub tags or GitHub releases).
  - If yes → cannot be fixed locally; goes in waiver report.
  - If no → bump the base image / pinned tool version and re-scan.

### 4. Patch (bucket A only)
Edit the Dockerfile with the smallest possible change:
- Prefer targeted upgrades (`apk upgrade expat`) over blanket `apk upgrade`.
- Keep existing style: same `RUN` layout, alphabetical package ordering where the file already uses it.
- Pin to the fixed version when upstream still ships newer vulnerable versions.

### 5. Verify
Rebuild and rescan:
```
docker build -q -t <name>-cve-test <context>
docker scout cves --only-severity critical <name>-cve-test
```
Confirm the fixable CVEs are gone.

### 6. Report
Present two sections to the user:

**Fixed** — list of CVEs patched, with the Dockerfile lines added.

**Waiver candidates** (upstream-only, base already latest) — as a markdown table:

| CVE | Package | Installed | Fixed in | Status |
|---|---|---|---|---|
| CVE-XXXX-YYYYY | pkg-name | x.y.z | a.b.c | Compiled into vendor binary; awaits upstream rebuild |

Include a one-line **Justification** stating the base image is already on the latest upstream release (with version + release date).

### 7. Stop
Do not `git add` or `git commit`. End with "Ready to commit — review and commit when you're happy."

## Notes

- Never run `apk upgrade` with no package list unless the user asks — it bloats the image and obscures what changed.
- If a CVE has no fixed version yet, note it in the waiver report, not the fixed section.
- Rescan after every Dockerfile change — don't assume a fix worked.

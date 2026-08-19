<!-- markdownlint-disable -->

# Hardening Report: dependabot--fetch-metadata/v3.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dependabot--fetch-metadata/v3.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable version tags instead of immutable 40-character SHA digests. Affected references: `actions/checkout@v6` (all workflow files), `actions/setup-node@v6` (check-uncommitted.yml, ci.yml, dependabot-build.yml, release-bump-version.yml), and `actions/publish-immutable-action@v0.0.4` (release-publish-package.yml). These can be silently updated by the action author to inject malicious code.

Locations:

- `.github/workflows/check-uncommitted.yml:16`
- `.github/workflows/check-uncommitted.yml:18`
- `.github/workflows/check-uncommitted.yml:33`
- `.github/workflows/ci.yml:15`
- `.github/workflows/ci.yml:17`
- `.github/workflows/dependabot-auto-merge.yml:16`
- `.github/workflows/dependabot-build.yml:18`
- `.github/workflows/dependabot-build.yml:37`
- `.github/workflows/dependabot-build.yml:40`
- `.github/workflows/release-bump-version.yml:27`
- `.github/workflows/release-bump-version.yml:30`
- `.github/workflows/release-move-tracking-tag.yml:46`
- `.github/workflows/release-publish-package.yml:17`
- `.github/workflows/release-publish-package.yml:20`

### missing-permissions (severity: medium)

Six workflow files have no top-level `permissions:` key and no per-job `permissions:` keys. Without explicit permissions, workflows run with the default (often broad) token permissions. Each of these files should declare minimal required permissions at the top level or per job.

Locations:

- `.github/workflows/check-uncommitted.yml:1`
- `.github/workflows/ci.yml:1`
- `.github/workflows/dependabot-auto-merge.yml:1`
- `.github/workflows/dependabot-build.yml:1`
- `.github/workflows/release-bump-version.yml:1`
- `.github/workflows/release-move-tracking-tag.yml:9`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands, allowing expression values to be parsed as shell syntax before the shell ever sees them.

(a) `dependabot-auto-merge.yml`: `gh pr merge --auto --merge '${{ github.event.pull_request.html_url }}'` — the PR URL from a pull_request_target event is attacker-controlled and injected directly into the shell command.

(a) `release-bump-version.yml`: `bin/bump-version ${{ github.event.inputs.version_type || 'minor' }}` — a workflow_dispatch input is interpolated directly into the shell command.

(a) `release-bump-version.yml`: `git checkout -b "bump-to-${{ env.NEW_VERSION }}"` and `git commit -m "${{ env.NEW_VERSION }}"` — env context values are interpolated directly into shell commands.

(a) `release-bump-version.yml` (Set summary step): Multiple `echo` lines interpolate `${{ env.NEW_VERSION }}`, `${{ github.repository }}` directly into shell.

(a) `release-move-tracking-tag.yml`: `echo "... ${{ github.event.release.name }} ..." >> $GITHUB_STEP_SUMMARY` — release name is attacker-influenced and injected into shell.

(a) `release-publish-package.yml`: `echo "... ${{ github.event.release.name }}" >> $GITHUB_STEP_SUMMARY` — same issue.

Locations:

- `.github/workflows/dependabot-auto-merge.yml:18`
- `.github/workflows/release-bump-version.yml:37`
- `.github/workflows/release-bump-version.yml:50`
- `.github/workflows/release-bump-version.yml:54`
- `.github/workflows/release-bump-version.yml:64`
- `.github/workflows/release-bump-version.yml:68`
- `.github/workflows/release-move-tracking-tag.yml:56`
- `.github/workflows/release-publish-package.yml:22`

### github-env-injection (severity: high)

In `release-bump-version.yml`, the 'Bump the version' step writes a value derived from the untrusted `${{ github.event.inputs.version_type || 'minor' }}` workflow_dispatch input to `$GITHUB_ENV` without sanitization. The script runs `NEW_VERSION=$(bin/bump-version ${{ github.event.inputs.version_type || 'minor' }})` and then `echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_ENV`. Because the input is interpolated directly into the shell command that computes `NEW_VERSION`, and the result is written to `$GITHUB_ENV` without the required `printf '%s' "$NEW_VERSION" | tr -d '\n\r'` sanitization step, an attacker with workflow_dispatch access could inject newlines to set arbitrary environment variables for subsequent steps.

Locations:

- `.github/workflows/release-bump-version.yml:37`
- `.github/workflows/release-bump-version.yml:39`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all 4 findings across 7 workflow files:

1. **unpinned-uses**: Pinned all mutable action references to full SHA digests:
   - `actions/checkout@v6` → `@d23441a48e516b6c34aea4fa41551a30e30af803 # v6`
   - `actions/setup-node@v6` → `@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6`
   - `actions/publish-immutable-action@v0.0.4` → `@4bc8754ffc40f27910afb20287dbbbb675a4e978 # v0.0.4`

2. **missing-permissions**: Added `permissions: contents: read` at the top level of all 6 affected workflow files. Added `permissions: contents: write` at the job level for jobs that need to push commits/tags (dependabot-build, release-bump-version, release-move-tracking-tag).

3. **script-injection**: Moved all `${{ }}` expressions out of `run:` shell strings into `env:` blocks:
   - `dependabot-auto-merge.yml`: PR URL moved to `PR_URL` env var
   - `release-bump-version.yml`: version_type input moved to `VERSION_TYPE` env var; NEW_VERSION and REPOSITORY moved to env vars in commit/summary steps
   - `release-move-tracking-tag.yml`: release.name moved to `RELEASE_NAME` env var
   - `release-publish-package.yml`: release.name moved to `RELEASE_NAME` env var

4. **github-env-injection**: In `release-bump-version.yml`, sanitized NEW_VERSION with `printf '%s' "$NEW_VERSION" | tr -d '\n\r'` before writing to $GITHUB_ENV. Also moved the version_type input to an env var to prevent shell injection in the bump-version command.


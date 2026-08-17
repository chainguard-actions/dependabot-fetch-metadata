<!-- markdownlint-disable -->

# Hardening Report: dependabot--fetch-metadata/v3.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dependabot--fetch-metadata/v3.1.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

actions/checkout@v6 and actions/setup-node@v6 are referenced by mutable version tags, not pinned to a full 40-character commit SHA. This exposes the workflow to supply-chain attacks if the tag is moved.

Locations:

- `.github/workflows/check-uncommitted.yml:19`
- `.github/workflows/check-uncommitted.yml:21`
- `.github/workflows/check-uncommitted.yml:37`
- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:21`
- `.github/workflows/dependabot-auto-merge.yml:19`
- `.github/workflows/dependabot-build.yml:21`
- `.github/workflows/dependabot-build.yml:43`
- `.github/workflows/dependabot-build.yml:47`
- `.github/workflows/release-bump-version.yml:31`
- `.github/workflows/release-bump-version.yml:35`
- `.github/workflows/release-move-tracking-tag.yml:37`
- `.github/workflows/release-publish-package.yml:10`
- `.github/workflows/release-publish-package.yml:14`

### script-injection (severity: high)

Rule (a): GitHub Actions expressions are interpolated directly inside run: shell commands, allowing an attacker to inject arbitrary shell commands. Affected lines:
- dependabot-auto-merge.yml: `run: gh pr merge --auto --merge '${{ github.event.pull_request.html_url }}'` — github.event.pull_request.html_url is attacker-controlled via a PR.
- release-bump-version.yml: `NEW_VERSION=$(bin/bump-version ${{ github.event.inputs.version_type || 'minor' }})` — workflow_dispatch input interpolated directly into shell.
- release-bump-version.yml: `git checkout -b "bump-to-${{ env.NEW_VERSION }}"` and `git commit -m "${{ env.NEW_VERSION }}"` — env context expressions interpolated into shell.
- release-move-tracking-tag.yml: `run: git tag -f ${{ steps.extract_major.outputs.major_tag }}` and `run: git push -f origin refs/tags/${{ steps.extract_major.outputs.major_tag }}` — step output interpolated directly into shell commands.

Locations:

- `.github/workflows/dependabot-auto-merge.yml:21`
- `.github/workflows/release-bump-version.yml:40`
- `.github/workflows/release-bump-version.yml:51`
- `.github/workflows/release-bump-version.yml:54`
- `.github/workflows/release-move-tracking-tag.yml:51`
- `.github/workflows/release-move-tracking-tag.yml:54`

### github-env-injection (severity: high)

An untrusted input value is written to $GITHUB_ENV without sanitization. In release-bump-version.yml, `${{ github.event.inputs.version_type || 'minor' }}` is interpolated directly into the shell command that computes NEW_VERSION, and the result is then written to $GITHUB_ENV via `echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_ENV` with no `printf '%s' ... | tr -d '\n\r'` sanitization step. A malicious version_type input containing newlines could inject arbitrary environment variables.

Locations:

- `.github/workflows/release-bump-version.yml:40`
- `.github/workflows/release-bump-version.yml:42`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all 14 unpinned action references across 6 workflow files by pinning to full commit SHAs (actions/checkout@v6→d23441a48e516b6c34aea4fa41551a30e30af803, actions/setup-node@v6→249970729cb0ef3589644e2896645e5dc5ba9c38, actions/publish-immutable-action@v0.0.4→4bc8754ffc40f27910afb20287dbbbb675a4e978). Fixed all 6 script injection locations by moving ${{ }} expressions into env: blocks and referencing them as plain shell variables. Fixed github-env-injection in release-bump-version.yml by sanitizing NEW_VERSION with printf/tr before writing to GITHUB_ENV. Also repaired a file corruption in dependabot-build.yml that occurred during editing.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in three workflow files:
1. `.github/workflows/release-bump-version.yml` (line 89): Moved `${{ env.PR_URL }}`, `${{ env.NEW_VERSION }}`, and `${{ github.repository }}` into an `env:` block as `SUMMARY_PR_URL`, `SUMMARY_NEW_VERSION`, and `SUMMARY_REPOSITORY`, then referenced them as plain shell variables in the `run:` script.
2. `.github/workflows/release-move-tracking-tag.yml` (line 63): Moved `${{ steps.extract_major.outputs.major_tag }}` and `${{ github.event.release.name }}` into an `env:` block as `SUMMARY_MAJOR_TAG` and `SUMMARY_RELEASE_NAME`, then referenced them as plain shell variables.
3. `.github/workflows/release-publish-package.yml` (line 27): Moved `${{ github.event.release.name }}` into an `env:` block as `SUMMARY_RELEASE_NAME`, then referenced it as a plain shell variable.

In all cases, the attacker-controllable values (especially `github.event.release.name`) are now passed through the environment rather than being interpolated directly into shell command strings by the YAML template engine.

### Iteration 3

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'Extract major version from release tag' step in .github/workflows/release-move-tracking-tag.yml. Added sanitization of MAJOR_TAG before writing to $GITHUB_OUTPUT: `safe_major_tag=$(printf '%s' "$MAJOR_TAG" | tr -d '\n\r')` and then writing `safe_major_tag` instead of `MAJOR_TAG` to prevent newline injection attacks via crafted release tag names.


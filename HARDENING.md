<!-- markdownlint-disable -->

# Hardening Report: dependabot--fetch-metadata/v2.5.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **dependabot--fetch-metadata/v2.5.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct ${{ }} expression interpolation inside run: shell commands. In dependabot-auto-merge.yml, `${{ github.event.pull_request.html_url }}` is interpolated directly into a shell command — on a pull_request_target trigger this is attacker-controlled. In release-bump-version.yml, `${{ github.event.inputs.version_type || 'minor' }}` is interpolated into a shell command (workflow_dispatch input), and multiple `${{ env.NEW_VERSION }}`, `${{ github.repository }}`, `${{ env.PR_URL }}` expressions appear directly in run: blocks. In release-publish-package.yml and release-move-tracking-tag.yml, `${{ github.event.release.name }}` is interpolated directly into echo commands in run: blocks. All of these bypass shell quoting and allow injection of shell metacharacters.

Locations:

- `.github/workflows/dependabot-auto-merge.yml:19`
- `.github/workflows/release-bump-version.yml:38`
- `.github/workflows/release-bump-version.yml:52`
- `.github/workflows/release-bump-version.yml:55`
- `.github/workflows/release-bump-version.yml:68`
- `.github/workflows/release-publish-package.yml:24`
- `.github/workflows/release-move-tracking-tag.yml:56`

### permissions (severity: medium)

missing-permissions: These workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any job. Without explicit permissions, workflows inherit the default repository permissions (often write-all for private repos), violating the principle of least privilege. Affected files: ci.yml, check-uncommitted.yml, dependabot-auto-merge.yml (uses pull_request_target — especially dangerous without restricted permissions), dependabot-build.yml, release-bump-version.yml, release-move-tracking-tag.yml.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/check-uncommitted.yml:1`
- `.github/workflows/dependabot-auto-merge.yml:1`
- `.github/workflows/dependabot-build.yml:1`
- `.github/workflows/release-bump-version.yml:1`
- `.github/workflows/release-move-tracking-tag.yml:1`

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tag refs instead of immutable 40-character commit SHAs. This exposes the workflows to supply-chain attacks if the referenced tags are moved or compromised. Unpinned references found: `actions/checkout@v6` (used in all 7 workflow files), `actions/setup-node@v6` (used in ci.yml, check-uncommitted.yml, dependabot-build.yml, release-bump-version.yml), and `actions/publish-immutable-action@v0.0.4` (used in release-publish-package.yml). Note: `actions/create-github-app-token@29824e69f54612133e76f7eaac726eef6c875baf` is correctly pinned to a SHA.

Locations:

- `.github/workflows/ci.yml:14`
- `.github/workflows/ci.yml:16`
- `.github/workflows/check-uncommitted.yml:14`
- `.github/workflows/check-uncommitted.yml:16`
- `.github/workflows/check-uncommitted.yml:30`
- `.github/workflows/dependabot-auto-merge.yml:16`
- `.github/workflows/dependabot-build.yml:18`
- `.github/workflows/dependabot-build.yml:27`
- `.github/workflows/dependabot-build.yml:29`
- `.github/workflows/release-bump-version.yml:29`
- `.github/workflows/release-bump-version.yml:31`
- `.github/workflows/release-move-tracking-tag.yml:46`
- `.github/workflows/release-publish-package.yml:14`
- `.github/workflows/release-publish-package.yml:19`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection

**Notes:**

Fixed all three finding categories across 7 workflow files:

1. unpinned-uses: Pinned actions/checkout@v6 to SHA df4cb1c069e1874edd31b4311f1884172cec0e10, actions/setup-node@v6 to SHA 249970729cb0ef3589644e2896645e5dc5ba9c38, and actions/publish-immutable-action@v0.0.4 to SHA 4bc8754ffc40f27910afb20287dbbbb675a4e978. All pinned with # tag-name comments.

2. permissions: Added top-level 'permissions: contents: read' to ci.yml, check-uncommitted.yml, dependabot-auto-merge.yml, dependabot-build.yml, release-bump-version.yml, and release-move-tracking-tag.yml. release-publish-package.yml already had job-level permissions (contents: read, id-token: write, packages: write) which are appropriate for publishing.

3. script-injection: Moved all ${{ }} expressions out of run: shell commands into env: blocks — PR_URL in dependabot-auto-merge.yml; VERSION_TYPE, NEW_VERSION, REPOSITORY, PR_URL in release-bump-version.yml; RELEASE_NAME in release-publish-package.yml and release-move-tracking-tag.yml. All shell references now use plain $VAR_NAME syntax.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection finding in .github/workflows/release-bump-version.yml. In the 'Bump the version' step, the NEW_VERSION value (derived from bin/bump-version output) is now sanitized using `printf '%s' "$NEW_VERSION" | tr -d '\n\r'` before being written to $GITHUB_ENV. This prevents a malicious workflow_dispatch input containing newline characters from injecting arbitrary environment variable assignments into subsequent steps. Also added proper quoting around $GITHUB_ENV.


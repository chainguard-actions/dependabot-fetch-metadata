<!-- markdownlint-disable -->

# Hardening Report: dependabot--fetch-metadata/v2.5.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dependabot--fetch-metadata/v2.5.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tag-based refs instead of full 40-character SHA commit hashes. This exposes the workflow to supply-chain attacks if the tag is moved. Affected refs include: `actions/checkout@v6`, `actions/setup-node@v6`, and `actions/publish-immutable-action@v0.0.4`. Note: `actions/create-github-app-token@29824e69f54612133e76f7eaac726eef6c875baf` is correctly pinned.

Locations:

- `.github/workflows/check-uncommitted.yml:14`
- `.github/workflows/check-uncommitted.yml:16`
- `.github/workflows/check-uncommitted.yml:25`
- `.github/workflows/ci.yml:14`
- `.github/workflows/ci.yml:16`
- `.github/workflows/dependabot-auto-merge.yml:18`
- `.github/workflows/dependabot-build.yml:21`
- `.github/workflows/dependabot-build.yml:37`
- `.github/workflows/dependabot-build.yml:39`
- `.github/workflows/release-bump-version.yml:29`
- `.github/workflows/release-bump-version.yml:33`
- `.github/workflows/release-move-tracking-tag.yml:44`
- `.github/workflows/release-publish-package.yml:14`
- `.github/workflows/release-publish-package.yml:18`

### missing-permissions (severity: medium)

These workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any job. Without explicit permissions, the GITHUB_TOKEN is granted its default (often broad) permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/check-uncommitted.yml:1`
- `.github/workflows/ci.yml:1`
- `.github/workflows/dependabot-auto-merge.yml:1`
- `.github/workflows/dependabot-build.yml:1`
- `.github/workflows/release-bump-version.yml:1`
- `.github/workflows/release-move-tracking-tag.yml:1`

### script-injection (severity: high)

Direct `${{ ... }}` expression interpolation inside `run:` shell command strings (sub-rule a). Before the shell executes the command, GitHub Actions substitutes the expression value as raw text, allowing an attacker to inject arbitrary shell commands.

1. `dependabot-auto-merge.yml` line 20: `run: gh pr merge --auto --merge '${{ github.event.pull_request.html_url }}'` — `github.event.pull_request.html_url` is attacker-controlled via a PR and is interpolated directly into the shell command.

2. `release-bump-version.yml` line 36: `NEW_VERSION=$(bin/bump-version ${{ github.event.inputs.version_type || 'minor' }})` — `github.event.inputs.version_type` is a `workflow_dispatch` input interpolated directly into a shell command without quoting.

3. `release-bump-version.yml` line 47: `git checkout -b "bump-to-${{ env.NEW_VERSION }}"` — `env.*` context interpolated directly in a `run:` block.

4. `release-bump-version.yml` line 52: `git commit -m "${{ env.NEW_VERSION }}" -m "Release notes: https://github.com/${{ github.repository }}/releases/tag/${{ env.NEW_VERSION }}"` — multiple `env.*` and `github.*` expressions interpolated directly in a `run:` block.

Locations:

- `.github/workflows/dependabot-auto-merge.yml:20`
- `.github/workflows/release-bump-version.yml:36`
- `.github/workflows/release-bump-version.yml:47`
- `.github/workflows/release-bump-version.yml:52`

### github-env-injection (severity: high)

In `release-bump-version.yml`, the `Bump the version` step writes `NEW_VERSION` to `$GITHUB_ENV` without sanitization. `NEW_VERSION` is derived from `${{ github.event.inputs.version_type }}` — a user-controlled `workflow_dispatch` input — passed directly to `bin/bump-version`. The output of that command is captured into `NEW_VERSION` and then written as `echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_ENV` (line 38) without the required `printf '%s' ... | tr -d '\n\r'` sanitization step. A value containing newlines could inject additional environment variable assignments into subsequent steps.

Locations:

- `.github/workflows/release-bump-version.yml:38`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all 4 findings across 7 workflow files:

1. **unpinned-uses**: Pinned all mutable action refs to full SHAs:
   - actions/checkout@v6 → @d23441a48e516b6c34aea4fa41551a30e30af803
   - actions/setup-node@v6 → @249970729cb0ef3589644e2896645e5dc5ba9c38
   - actions/publish-immutable-action@v0.0.4 → @4bc8754ffc40f27910afb20287dbbbb675a4e978
   Applied in: check-uncommitted.yml, ci.yml, dependabot-auto-merge.yml, dependabot-build.yml, release-bump-version.yml, release-move-tracking-tag.yml, release-publish-package.yml

2. **missing-permissions**: Added top-level `permissions: {}` and minimal job-level permissions to check-uncommitted.yml, ci.yml, dependabot-auto-merge.yml, dependabot-build.yml, release-bump-version.yml, and release-move-tracking-tag.yml.

3. **script-injection**: Moved all `${{ }}` expressions out of `run:` blocks into `env:` blocks and referenced them as plain shell variables. Fixed in dependabot-auto-merge.yml (PR_URL) and release-bump-version.yml (VERSION_TYPE, NEW_VERSION, REPOSITORY).

4. **github-env-injection**: Sanitized NEW_VERSION and PR_URL before writing to $GITHUB_ENV using `printf '%s' "$VAR" | tr -d '\n\r'` in release-bump-version.yml.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities where `${{ github.event.release.name }}` was directly interpolated into `run:` shell strings:
1. `.github/workflows/release-move-tracking-tag.yml` (line 62): Moved the expression into `env: RELEASE_NAME: ${{ github.event.release.name }}` and replaced the inline expression with `$RELEASE_NAME` in the shell script.
2. `.github/workflows/release-publish-package.yml` (line 27): Same fix applied — expression moved to `env:` block and referenced as `$RELEASE_NAME` in the shell script.
Both changes prevent attacker-controlled release names from being executed as shell commands.


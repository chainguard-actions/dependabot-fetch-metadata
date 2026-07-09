<!-- markdownlint-disable -->

# Hardening Report: dependabot--fetch-metadata/v1.3.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **dependabot--fetch-metadata/v1.3.6** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable version tags (@v3) instead of full 40-character SHA digests. This exposes the workflow to supply-chain attacks if the tag is moved or the upstream action is compromised. Affected references: actions/checkout@v3 and actions/setup-node@v3 in all four workflow files.

Locations:

- `.github/workflows/check-dist.yml:14`
- `.github/workflows/check-dist.yml:22`
- `.github/workflows/ci.yml:13`
- `.github/workflows/ci.yml:21`
- `.github/workflows/dependabot-auto-merge.yml:11`
- `.github/workflows/dependabot-build.yml:22`
- `.github/workflows/dependabot-build.yml:42`
- `.github/workflows/dependabot-build.yml:50`

### missing-permissions (severity: medium)

check-dist.yml and ci.yml have no top-level permissions: key and no job-level permissions: key on any job. Without explicit permissions, workflows inherit the default repository permissions (which may include write access), violating the principle of least privilege.

Locations:

- `.github/workflows/check-dist.yml:1`
- `.github/workflows/ci.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions

**Notes:**

Fixed all 8 unpinned action references across 4 workflow files: replaced actions/checkout@v3 with @f43a0e5ff2bd294095638e18286ca9a3d1956744 and actions/setup-node@v3 with @3235b876344d2a9aa001b8d1453c930bba69e610, preserving the original tag as a comment. Added top-level 'permissions: contents: read' to check-dist.yml and ci.yml which had no permissions block. The dependabot-auto-merge.yml and dependabot-build.yml already had explicit permissions blocks (pull-requests: write, contents: write) which were preserved.


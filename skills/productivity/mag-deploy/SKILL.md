---
name: mag-deploy
description: Deploy a set of PRs to TEST or PROD. Covers three repo types: Data-lake-framework (version bump / releases.tf), TF repo (port changes to next environment), and data-lake-* (open PR to next branch). Trigger on /mag-deploy-test, /mag-deploy-prod, or when the user says "deploy to test", "deploy to prod/production", or "push to test/prod".
---

## Overview

Handles deployments in either direction:

| Direction | TF source | TF target | data-lake branch |
|---|---|---|---|
| dev -> test | tf-apps-dev | tf-apps-tst | develop -> test |
| test -> prod | tf-apps-tst | tf-application | test -> main |

For exact repo paths and path prefixes per environment, refer to the TF environment table in CLAUDE.md (always loaded in context). Note that the prod path uses a `prod/us-west-2/` prefix instead of `us-west-2/`.

Not every deployment touches all three repo types - use only what applies.

All PRs are opened following the `/mag-pr` conventions.

**Execution order: open data-lake PRs (Step 2) before the TF PR (Step 3)** so their URLs are available for sections 4 and 5 of the TF PR body.

---

## Step 0: Gather info before starting

### dev->test

Ask the user (or confirm from context):

1. **Which PRs are being deployed?** PR URLs/numbers for the framework PR (if any), the TF source PR, and the data-lake repo PR(s). These are optional - the user may not have them (TF already merged, no PR opened, etc.). If missing, explain how to proceed without them (see Step 3b).
2. **Deployment ticket number?** Always confirm - it may be a separate tracking ticket or the same as the feature ticket.

### test->prod

For test->prod, the set of changes is often cumulative and not tied to specific open PRs. Do not ask for PR numbers upfront. Instead:

1. **Confirm the deployment ticket number.**
2. **Diff the TF workspaces** to identify what needs to be ported (see Step 3b). Derive the list of related PRs from the diff - they are only needed for the PR body (section 4), not to determine what to port.

---

## Step 1: Data-lake-framework (if applicable)

### dev->test: bump version to release

If the feature includes a framework change:

1. Fetch and check out `trunk` in `data-lake/Data-lake-framework`
2. Read `src/magfw/__init__.py` to get the current version
3. If already a release version (no `.devN`), skip this step
4. If it is a `.devN` (e.g. `5.5.5.dev1`), strip the suffix to get the release version (e.g. `5.5.5`)
5. Edit `src/magfw/__init__.py` with the release version
6. Show the diff and open a PR to `trunk` with `/mag-pr`:
   - Branch: `<ticket-lowercase>/bump-magfw-<version>`
   - PR title: `<TICKET>: bump magfw version to <version>`
   - Commit message: `bump magfw version to <version>` (no ticket number in commit message)
   - Release mode: `[x] Patching release`
   - Release notes: list the combined feature changes in this version
   - Section 3c: tick `[x] branch name release/x.y.z` (the ALL environments option)
7. Tell the user: after this PR merges, create branch `release/<version>` from trunk to trigger the build pipeline

### test->prod: check releases.tf

The build already ran when deploying to test, so no new CodeBuild step. But if `fw_version` is being bumped as part of this deployment, the new version must exist in the target TF repo's `releases.tf` before it can be applied.

1. Identify the new `fw_version` from the TF workspace diff (Step 3b)
2. Check whether that version is already listed in `tf-application/prod/us-west-2/datalake-framework/releases.tf`
3. If not, add it to the `toset([...])` list
4. This is included as part of the TF PR (Step 3), with `datalake-framework` listed first in the apply order

---

## Step 2: data-lake-* repos (if applicable)

Open data-lake PRs before the TF PR so their URLs are available for TF PR sections 4 and 5.

| Direction | Head branch | Base branch |
|---|---|---|
| dev->test | develop | test |
| test->prod | test | main |

### Standard case: full branch promote

The feature is fully merged into the head branch. Open a PR directly - no new branch or commits needed:

```
gh pr create --base <base> --head <head> --title "<TICKET>: deploy <feature> to <ENV>" --assignee tnatomtongMAG --body "..."
```

### Selective/partial case: only some changes are ready

When only specific files from the head branch should be promoted (not everything):

1. Create a new branch from the base branch: `git checkout -b <ticket>/deploy-<feature>-to-<env> origin/<base>`
2. Bring in only the relevant files: `git checkout origin/<head> -- <path/to/file> [<path/to/file2> ...]`
3. Commit and push, then open a PR to the base branch with `/mag-pr`

### PR body

Use the repo's `pull_request_template.md`:
- **Section 3b:** tick only the target environment
- **Section 3c:** for promote PRs (no new code written in this PR), tick all boxes - self-review and code quality checks apply to the feature PRs already merged, not new work here
- **Section 3d:** leave the post-merge checkbox unticked

This PR is merged AFTER the TF is applied. Reference it in section 5 of the TF PR.

---

## Step 3: TF repo (if applicable)

**Goal:** Port the TF changes from the source environment into the target environment.

Look up the correct source/target TF repos and paths from the CLAUDE.md environment table.

### 3a: Prepare the branch

1. Fetch and check out `main` in the target TF repo
   - If the local checkout is actively in use by another session, use a git worktree instead: `git worktree add /tmp/<name> main` and work there. Remove it after the PR is merged: `git worktree remove /tmp/<name>`
2. Create a branch: `<ticket-lowercase>/deploy-<short-description>-to-<env>`

### 3b: Identify the delta

**If a source TF PR exists and is open:**
```
gh pr diff <number> --repo MarkAnthonyGroup2/<source-tf-repo>
```

**If the TF changes are already merged (no open PR):**
Diff the matching workspace directories between source and target to find what's missing:
```
diff -rq <source-tf-repo>/us-west-2/datalake-<repo>/ <target-tf-repo>/us-west-2/datalake-<repo>/
```
Then inspect the differing files to determine what needs to be ported.

**For test->prod cumulative deployments:**
Diff all changed workspaces between tf-apps-tst and tf-application to get a full picture of everything that hasn't been ported yet. Only port what's in scope for this deployment ticket.

### 3c: Apply the changes

Apply the identified changes to the matching workspace path in the target TF repo.

**Skip these files - never port them:**
- `backend.tf` - contains env-specific state workspace names and keys
- Any file with hardcoded account IDs, region-specific ARNs, or other env-specific values that are intentionally different between environments

**Cross-check:** the ported changes should be functionally identical to the source (same locals, env vars, IAM statements, Glue args) with only environment-parameterized values differing (handled by `var.environment_suffix`).

### 3d: Framework version (if applicable)

Before bumping `fw_version` anywhere in TF configs, always check the version is actually available first (i.e. already listed in the target repo's `releases.tf`, or the release/build has already completed). Do not bump to a version that hasn't been released or built yet.

**dev->test:** if Step 1 produced a new release version, also:
- Check whether that version is already listed in `us-west-2/datalake-framework/releases.tf` in the target TF repo; add it if missing
- Update `fw_version` in the relevant workspace's `locals.tf`

**test->prod:** if Step 1 found a missing version in releases.tf, include that change here.

### 3e: Open the PR

Show the full diff, then open a PR to `main` with `/mag-pr`:

- PR title: `<TICKET>: deploy <feature description> in <ENV>`
- Use the `pull_request_template.md` in the repo root (MAG Terraform Deployment template)
- **Section 0:** `[x]`
- **Section 1:** bullet points describing what changed
- **Section 2:** deployment ticket URL
- **Section 3 (Terraform Apply Order):** always fill this in carefully:
  - If `datalake-framework/releases.tf` changed (either dev->test or test->prod): list `datalake-framework` FIRST. For dev->test add the note "CodeBuild must have completed the build before applying". For test->prod just note that framework must be applied before the data-lake workspace.
  - Then list the data-lake workspace(s) after
  - If no framework change: just list the workspace(s) in logical order
- **Section 4:** list the related PR links (framework PRs, source TF PR, data-lake PRs) as bullet points, no extra detail needed. If no source PR exists, note the source commit or describe the diff instead.
- **Section 5:** "After TF is applied, merge <data-lake PR link(s)>"

---

## Step 4: Cross-references

Once all PR numbers are known, update section 4 and 5 of the TF PR with the exact links and current status of each PR.

---

## Deployment order (remind the user)

**dev->test:**
1. Framework PR merges to trunk -> user creates `release/<version>` branch -> CodeBuild runs
2. Cloud team applies `datalake-framework` TF workspace (after build completes)
3. Cloud team applies data-lake workspace(s)
4. Merge the data-lake test PR(s)

**test->prod:**
1. Cloud team applies `datalake-framework` TF workspace (if releases.tf changed)
2. Cloud team applies data-lake workspace(s) in tf-application
3. Merge the data-lake main PR(s)

---

## Notes

- Never add ticket number to framework commit messages
- Always confirm the deployment ticket number before opening any PR - it may differ from the feature ticket
- The Cloud team merges and applies TF PRs; data-lake PRs are merged by us after TF is applied
- Jira base URL: `https://markanthony.atlassian.net/browse/`

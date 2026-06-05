---
name: mag-pr
description: Create a branch, commit, push, and open a PR for MAG repos. Trigger on /mag-pr and proactively when the user asks to commit and push, open or create a PR, wrap up or ship a task, or says things like "let's open a PR", "push this up", "create the PR", or "wrap this up".
---

## Git identity

Always verify the repo's local git config is using the correct identity before committing:
- **name:** `tnatomtongMAG`
- **email:** `tomtong.natomtong@markanthony.com`

If not set, configure it locally first:
```
git config user.name "tnatomtongMAG"
git config user.email "tomtong.natomtong@markanthony.com"
```

## Branch naming

Format: `<ticket-number-lowercase>/<kebab-case-description>`

Examples:
- `deng-7487/precommit-hook-to-all`
- `feature/vip-sftp-ep2`

## Commit message

Use conventional commits format: `<type>: <short description>`

Common types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`

Examples:
- `feat: extend pre-commit hook to all datalake repos`
- `fix: correct s3 path for uscom raw bucket`
- `chore: bump magfw version to 5.5.5`

Do NOT put the ticket number in the commit message.

## PR title format

`<TICKET-NUMBER>: <short descriptive title>`

The title should express:
- **What** changed
- **Scope** if relevant (e.g. which repos, which envs, which team)

Examples:
- `DENG-7487: extend pre-commit hook to all datalake repos and envs`

Ask the user to confirm the title before committing if unsure.

## PR description

Always read the actual `pull_request_template.md` file in the repo and populate every section it contains. The file is the source of truth — do not assume sections based on what is written here.

The template shown below is the most common one across MAG data lake repos. The checklist rules below only apply when those sections are present in the actual file:

```
# MAG Data Lakehouse Pull Request

## 1: Description
- bullet points describing what changed and why

## 2: Jira Ticket
https://markanthony.atlassian.net/browse/<TICKET>

## 3: Checklist before requesting a review

### 3a: Title
- [x] I have added Jira ticket number to the start of PR title

### 3b: Environments
This PR is for
- [x/blank] develop
- [x/blank] test
- [x/blank] main (prod)

### 3c: Quality of code
- [x] I have performed a self-review of my code
- [ ] I have reviewed and resolved Pylint errors and warnings
- [ ] I have applied `Black` with `"--line-length=88"` and `isort` to my code
- [ ] I have commented on my code, particularly in hard-to-understand areas

### 3d: Test results
If this PR is for develop branch
- [x] I have posted an expected result and my test result under the comment in my Jira ticket
Here: https://markanthony.atlassian.net/browse/<TICKET>?focusedCommentId=<ID>

If this PR is for test or main branch
- [ ] I will post an expected result and my test result under the comment in my Jira ticket after merged
```

### Checklist rules (only apply when the section exists in the actual template)
- **3a**: always `[x]` if ticket number is in the title
- **3b**: tick only the target base branch
- **3c**: `self-review` is always `[x]`; Pylint/Black/isort/comments are `[ ]` unless the user confirms
- **3d**: for develop PRs, ask the user for the Jira comment link to populate the `Here:` line; for test/main leave the post-merge checkbox unticked

## Steps

1. Read `pull_request_template.md` in the repo
2. Gather any missing info from the user **before** opening the PR:
   - Ticket number (if not already known)
   - Target base branch (develop / test / main)
   - Jira comment link for the test results `Here:` line (develop PRs only)
3. Show the proposed branch name and PR title to the user and wait for confirmation
4. Create and switch to the new branch from the correct base: `git checkout -b <branch> origin/<base>`
5. Stage only the relevant files (never `git add -A` blindly)
6. Commit — the repo's local config should already have the correct identity; no need to pass `-c user.name/email` unless the config is wrong
7. Push: `git push -u origin <branch>`
8. Open PR with `gh pr create` using the fully populated template body — show the full description to the user for review before running the command
   - Always add `--assignee tnatomtongMAG` to the `gh pr create` command
   - If `gh` is unavailable, provide the GitHub URL from the push output and the full description for the user to paste

## Repo-specific PR notes

### tf-apps-dev
- Do NOT use the `.github/PULL_REQUEST_TEMPLATE.md` in this repo — it is incorrect
- Write a short free-form description instead (1-2 bullets on what changed and the Jira ticket)
- Target branch is `main`

### tf-apps-tst and tf-application
- Both use the "MAG Terraform Deployment" PR template (sections: Expected Changes, Deployment Notes, Deployment Tickets, Terraform Apply Order, Related GitHub PRs, Post deployment tasks)
- tf-apps-tst: template is at `pull_request_template.md` in the repo root
- tf-application: template is at `.github/PULL_REQUEST_TEMPLATE.md`
- Target branch is `main` for both

### Data-lake-framework
- Uses its own PR template — sections are: 1 (Release mode), 2 (Release notes), 3a (Title), 3b (Quality of code), 3c (Releasing). There is no section 3d, so do not ask for a Jira comment link.
- Target branch is `trunk`
- Development releases: bump version in `src/magfw/__init__.py` (increment `.devN`) based on current `trunk` version before committing

## Notes

- Never skip hooks (`--no-verify`)
- Never force push to main/trunk/develop
- Jira base URL: `https://markanthony.atlassian.net/browse/`
- Never add `Co-Authored-By` or any Claude attribution to commit messages

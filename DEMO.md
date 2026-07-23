# Demo: Reproducing the attack and the block

## Prerequisites

- Admin access to this repository
- GitHub Actions enabled

## Step 1 — Configure protection (already done via ruleset)

The repository ruleset on `main` enforces:
- 2 required approving reviews
- Dismiss stale reviews on push
- Require approval of the most recent reviewable push
- Required status checks: `CI / lint` and `Four-Eyes Check / verify`
- Code owner review required
- Admins cannot bypass

## Step 2 — Open a bot PR

Trigger the `Auto Translation Bot` workflow via the Actions tab
(`workflow_dispatch`). It creates a branch `bot/translation-sync-*` and
opens a PR titled "chore: sync translations" authored by
`github-actions[bot]`.

## Step 3 — Simulate the attack

As the human developer (write access), push a commit to the bot's branch:

```bash
git fetch origin
git checkout bot/translation-sync-<timestamp>
echo "// backdoor" >> src/app.js
git add src/app.js
git commit -m "chore: add translation helper"
git push origin bot/translation-sync-<timestamp>
```

Then approve the PR via the GitHub UI or:

```bash
gh pr review <PR_NUMBER> --approve
```

## Step 4 — Observe the block

The `Four-Eyes Check` required status check will **fail** with:

```
Four-eyes violation: approver(s) [jbaenaxd] also committed to this PR.
The four-eyes principle requires a second person who did not write the
code to approve it.
```

The PR cannot be merged.

## Step 5 — Verify the legitimate path works

Open a normal PR where the author does NOT push commits to anyone else's
branch. A second person approves it. The `Four-Eyes Check` passes. The PR
merges normally — no developer friction.
# git-sec-test-01

Demonstration repository for closing the GitHub bot/fork PR self-approval
bypass — the four-eyes principle vulnerability demonstrated in
[trezor/trezor-suite#24825](https://github.com/trezor/trezor-suite/pull/24825).

## What this repo proves

1. **The attack works without protection** — a developer can push a commit
   to a bot's PR branch, approve it themselves, and merge it, satisfying
   GitHub's "required review" with only one pair of eyes.

2. **The solution blocks it** — a combination of native GitHub ruleset
   settings and a custom required status check closes the bypass.

## Structure

```
.github/
  CODEOWNERS                          # every file requires @jbaenaxd review
  workflows/
    ci.yml                             # basic required check
    four-eyes-check.yml                # the core mitigation
    auto-translation-bot.yml           # demo: opens a bot PR to attack
SOLUTION.md                            # formal writeup
DEMO.md                               # step-by-step reproduction guide
src/app.js                             # trivial source
```

## The mitigation in one sentence

GitHub's "PR author cannot approve their own PR" rule is keyed to who
**opened** the PR, not who **committed** to it — so a custom required
status check verifies that no approver appears among the PR's commit
authors or committers, fail-closed on unlinked emails.

See [SOLUTION.md](SOLUTION.md) for the full writeup.
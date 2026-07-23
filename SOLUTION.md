# Solution: Closing the Bot/Fork PR Self-Approval Bypass

## The vulnerability

A developer with write access can bypass the four-eyes principle by
pushing a commit to a **bot's pull request** (or a **fork's pull request**),
then **approving the PR themselves**.

GitHub's only author-binding rule is *"the PR author cannot approve their
own PR"* — keyed to who **opened** the PR, not who **committed** to it. The
bot opened the PR, so the bot is the author. The human attacker is "another
person" in GitHub's eyes and their approval satisfies the required review.

### Real-world demonstration

This is exactly what happened in [`trezor/trezor-suite#24825`][pr]:

1. `trezor-bot` opened a PR titled *"Crowdin translations update [DESKTOP]"*
2. `komret` (Jan Komárek, SatoshiLabs) pushed a commit to the bot's branch
   adding a React component into the **address confirmation modal** — the
   screen where users verify a recipient address before sending crypto
3. `komret` **approved the PR himself** (GitHub allowed it — he is not the
   PR author, the bot is)
4. `komret` **merged the PR** 38 seconds later

One human. One pair of eyes. Four-eyes principle satisfied on paper.

The planted code was a benign Dune quote ("Litany Against Fear") but was
**production-only** (`isCodesignBuild()`), **time-bombed** (expired
2026-02-10), and placed in a **security-critical UI flow** — proving any
code could be smuggled through this path.

[pr]: https://github.com/trezor/trezor-suite/pull/24825

### Why GitHub's native settings are insufficient

GitHub shipped `require_last_push_approval` on 2022-10-20 in direct
response to the Legit Security "pull request hijacking" disclosure. It
blocks the **single most recent pusher** from approving. But it does not
block:

- **Any prior committer** (only the *last* pusher is checked)
- **Collusion** (attacker A pushes, attacker B approves)
- **Unlinked email bypass** (a committer whose email is not linked to a
  GitHub account silently drops out of the identity set — the Palantir
  policy-bot#1263 bug)

GitHub confirmed this is **intended behavior** and assigned no CVE.
The Mercari Engineering blog (Dec 2024) concluded: *"it isn't possible to
prevent this attack using just features provided by GitHub."*

---

## The solution: layered, minimal-impact

### Layer 1 — Native GitHub Enterprise settings (free, instant)

Configured as a **repository ruleset** (available on GitHub Enterprise):

| Setting | What it blocks |
|---|---|
| `require_last_push_approval: true` | Last pusher cannot self-approve |
| `dismiss_stale_reviews_on_push: true` | New commits invalidate prior approvals |
| Disable "Allow GitHub Actions to create and approve pull requests" | Blocks `github-actions[bot]` self-approval |
| `required_approving_review_count: 2` | Two distinct human reviewers required |
| `require_code_owner_review: true` + CODEOWNERS | Narrows the approver pool per path |
| `enforce_admins: true` | Admins cannot bypass the rules |
| "Require signed commits" | Commits bound to a verified key |

On GitHub Enterprise, these can be enforced at the **organization level**
so every repo inherits them — zero per-repo maintenance.

### Layer 2 — Custom required status check (the core fix)

A GitHub Actions workflow (`four-eyes-check.yml`) that fails the build if
**any approver also appears in the PR's commit authors or committers**.

This is the check GitHub does not ship natively. It closes the gap because
it binds to **who committed**, not just who opened the PR.

Key design decisions:
- **Fail-closed on unlinked emails** — if a commit author/committer email is
  not linked to a GitHub account, the check fails. This prevents the
  Palantir policy-bot#1263 bypass where an unlinked email silently drops
  the contributor out of the banned set.
- **Runs on `merge_group`** — re-evaluates inside the merge queue after
  rebase, so last-minute tampering is caught.
- **Bot approvals don't count** — only human approvals satisfy the check.
- **~60 lines of inline JavaScript** — no third-party Action dependency,
  pinned to `actions/github-script@v7` only.

Registered as a **required status check** in the ruleset so the PR cannot
merge unless it passes.

### Layer 3 — Optional defense-in-depth

- **Merge queue** with `merge_group` trigger — re-runs all checks on the
  actual merged commit, decoupling the reviewed SHA from the merged SHA
- **Gitsign (Sigstore) signed commits** — keyless signing bound to GitHub
  OIDC identity, logged to a transparency log
- **SLSA Source L4 attestation** — cryptographic proof of two-party review

---

## Why this meets the constraints

| Constraint | How |
|---|---|
| Plugs the security hole | `require_last_push_approval` + custom check closes the bot/fork self-approval bypass and its collusion variant |
| Minimal maintenance | Org-level ruleset = one config, all repos. Custom check is ~60 lines inline, no third-party runtime dependency |
| Minimal cost | All native settings are free on GitHub Enterprise. The Action runs on included CI minutes |
| Minimal dev-speed impact | No extra human steps for normal PRs (reviewer ≠ committer → check passes instantly). Friction only appears for the attack pattern itself |

---

## Framework alignment

- **OpenSSF Scorecard** Branch-Protection Tier 5 (10/10): dismiss stale
  reviews + enforce admins + ≥2 reviewers
- **SLSA v1.2 Source L4** (Two-party review): *"Changes in protected
  branches MUST be agreed to by two or more trusted persons"*
- **NIST SSDF** (SP 800-218): PO.2 (separation of duties), PW.7 (human code
  review), PS.1/PS.2 (release integrity)

---

## Demonstration

This repository includes a working demo:

1. `auto-translation-bot.yml` — a workflow that opens a PR as
   `github-actions[bot]`, mimicking the Trezor exercise
2. `four-eyes-check.yml` — the mitigation, registered as a required check
3. The branch protection ruleset enforces all Layer 1 settings

To reproduce the attack and see it blocked, see [DEMO.md](DEMO.md).

## References

- [trezor/trezor-suite#24825](https://github.com/trezor/trezor-suite/pull/24825) — the exercise PR
- [Legit Security — Bypassing GitHub Required Reviewers (2022)](https://www.legitsecurity.com/blog/bypassing-github-required-reviewers-to-submit-malicious-code)
- [Mercari Engineering — How to bypass GitHub's Branch Protection (2024)](https://engineering.mercari.com/en/blog/entry/20241217-github-branch-protection/)
- [GitHub Changelog — Last Pusher rule (2022-10-20)](https://github.blog/changelog/2022-10-20-new-branch-protections-last-pusher-and-locked-branch/)
- [GitHub Changelog — Security enhancements to required approvals (2023-06-06)](https://github.blog/changelog/2023-06-06-security-enhancements-to-required-approvals-on-pull-requests/)
- [palantir/policy-bot#1263](https://github.com/palantir/policy-bot/pull/1263) — the unlinked-email bypass
- [suzuki-shunsuke/validate-pr-review-action](https://github.com/suzuki-shunsuke/validate-pr-review-action) — production-grade alternative
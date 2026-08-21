# await-approval

In-house manual-approval gate for GitHub Actions: pause a job on a GitHub issue until an
allowlisted approver comments approval. Built as the org's replacement for
`trstringer/manual-approval` (bounded-systems/infra#62) so no third-party code runs in
privileged lanes — and hosted here, publicly, because GitHub never resolves a private
repo's actions into public caller repos (no setting overrides that).

## How it works

1. Opens an issue in the calling repo titled with `issue-title`, linking the waiting run.
2. Polls the issue's comments. **Only authors on the `approvers` allowlist count** —
   comments from anyone else are ignored entirely, so a drive-by comment can neither
   release nor cancel the gate.
3. An allowlisted `approve` / `approved` / `lgtm` / `yes` lets the job proceed;
   `deny` / `denied` / `no` — or the timeout — **fails closed**.
4. The issue is closed with the outcome either way.

Needs `issues: write` on the job and the ambient `github.token` — no new secret.

## Usage

Always consume this action **SHA-pinned** (never `@main`):

```yaml
permissions:
  issues: write

steps:
  - name: Await approval
    uses: bounded-systems/await-approval@<commit-sha>
    with:
      approvers: alice,bob
      issue-title: "Approve production deploy"
      issue-body: >
        A deploy run is waiting. Approve to let it mint credentials and ship.
      timeout-minutes: "50"   # optional, default 50 — keep <= the job timeout
      poll-seconds: "15"      # optional, default 15
```

## The non-blocking split (`open` + `verify`)

`wait` holds a live runner through the whole of human latency — billed, and capped by
`timeout-minutes`, so an approver at lunch doesn't delay a deploy, they **fail** it and
every tripwire re-runs. Measured on one `bounded-systems/infra` broker deploy:
**10m46s** of runner time whose entire work was `sleep 15` and a comments API call.

`open` and `verify` split that in two, with no idling. Both halves live in **one workflow
file**, which matters: `job_workflow_ref` — the thing the OIDC broker pins — is file-level,
so splitting this way changes no pin anywhere.

```yaml
on:
  workflow_dispatch:
  issue_comment:

jobs:
  request:
    if: github.event_name == 'workflow_dispatch'
    permissions: { issues: write }
    steps:
      - run: ./tripwires.sh                       # fail BEFORE asking a human
      - uses: bounded-systems/await-approval@<sha>
        with:
          mode: open
          approvers: alice,bob
          issue-title: "Approve production deploy"
          issue-body: "…what this will mint and ship…"
          bind: ${{ github.sha }}                 # see below
      # the run ends here

  resume:
    if: github.event_name == 'issue_comment'
    permissions: { issues: write, id-token: write }
    environment: production                       # the broker's environment pin
    steps:
      - uses: bounded-systems/await-approval@<sha>
        with:
          mode: verify
          approvers: alice,bob
          issue-number: ${{ github.event.issue.number }}
          bind: ${{ github.sha }}
      - run: ./mint-and-deploy.sh                 # only reached on approval
```

`verify` refuses unless **all** of these hold, because a split gate is forgeable in ways a
blocking one is not:

| check | what it stops |
|---|---|
| issue opened by `github-actions[bot]` | a human filing a lookalike request and approving it |
| carries the `awaiting-approval` label | the same, via an issue this action never opened |
| issue is still **open** | replay — a second `approve` on an acted-on issue |
| `bind` matches the value recorded in the issue | an approval given for one artifact resuming against another |

**`bind` is the one that is easy to skip and shouldn't be.** With `wait`, the approved run
holds its own checkout, so what was approved and what ships are the same thing by
construction. Once split, `main` can move between the request and the resume. Pass a commit
SHA, a registry digest, or both.

`open` and `verify` share **one** matcher implementation with `wait`. Two copies would
drift, and a gate whose halves disagree about what counts as approval is worse than either
half alone.

## What counts as approval

The comment body must be **exactly** one keyword after lowercasing and stripping all
whitespace — `approve`, `approved`, `lgtm`, `yes` (or `deny`, `denied`, `no`). Not
"approve if CI is green", not "I approve", not `approve` with a signature under it.

That strictness is load-bearing beyond typo-resistance: **it is what keeps an automated
caller from operating a human gate.** An agent commenting with the maintainer's credentials
still authenticates as the maintainer — the allowlist cannot tell them apart — but agent
tooling generally appends an attribution footer to what it posts, and a body carrying
anything besides the keyword is not a verdict. This has been exercised for real: a Claude
Code session asked to approve posted `approve`, the harness appended its footer, and the
gate correctly ignored it.

Do not "fix" this by matching the first line or a substring. That would make the gate
agent-operable, and the audit trail would show a human's login.

## Security posture

- **SHA-pin only.** A commit SHA is immutable content; the pin — not this repo's branch
  state — is what your workflow trusts.
- Fail closed: no approval means the step fails; the gate cannot be released by
  non-allowlisted users, timeouts, or issue edits.
- Self-contained bash + `gh`; zero dependencies, no `curl | sh`, nothing fetched at
  runtime.

Consumers in this org: `bounded-systems/infra` (privileged apply/deploy lanes) and
`bounded-systems/front-desk-scheduler` (`lease-deploy`).

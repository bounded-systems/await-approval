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

## Security posture

- **SHA-pin only.** A commit SHA is immutable content; the pin — not this repo's branch
  state — is what your workflow trusts.
- Fail closed: no approval means the step fails; the gate cannot be released by
  non-allowlisted users, timeouts, or issue edits.
- Self-contained bash + `gh`; zero dependencies, no `curl | sh`, nothing fetched at
  runtime.

Consumers in this org: `bounded-systems/infra` (privileged apply/deploy lanes) and
`bounded-systems/front-desk-scheduler` (`lease-deploy`).

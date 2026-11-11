# Agentiqa — End-to-End Agentic QA in CI

Run your saved [Agentiqa](https://agentiqa.com) test plans in CI — nightly, on a release branch, or on demand — and gate the job on the result.

Agentiqa is AI-powered QA: an LLM agent drives a real browser through your saved test plans (explore / agent-based end-to-end flows you author in the app), discovers regressions, and reports what it found. This action is a thin, **versioned invocation contract** around the published [`agentiqa`](https://www.npmjs.com/package/agentiqa) npm CLI (`npx agentiqa@latest run --engine ...`). It runs your project's plans against your deployed app, captures a machine-readable JSON envelope, always uploads artifacts (screenshots, video, `result.json`), and fails the job per a configurable exit-code policy.

```yaml
- uses: Agentiqa/qa-action@v1
  with:
    service-key: ${{ secrets.AGENTIQA_SERVICE_KEY }}
```

That runs **all** plans in the service key's project on Agentiqa Cloud and fails the job on a plan failure or a config error (see [gating](#exit-code-contract-and-fail-on) below).

---

## Prerequisites

1. An **Agentiqa account** (sign up at [agentiqa.com](https://agentiqa.com)).
2. A **project with at least one saved test plan.** Plans are authored in the Agentiqa web or desktop app, not in this action — the action only *runs* plans that already exist in the project. A run that executes zero plans is treated as a configuration error, not a pass.
3. A **CLI service key** for that project, stored as a **repository secret.** Mint one in the web app: **Project Settings → CLI Service Keys → Create Key**, and copy the raw `sk_...` value immediately (it is shown once). Add it under **Settings → Secrets and variables → Actions** — the examples below assume the name `AGENTIQA_SERVICE_KEY`. The key is project-scoped: the action runs that project's plans. Never inline the key; always pass it from a secret.

---

## Quickstart

A nightly run that tests your app every morning and fails the job on real regressions:

```yaml
# .github/workflows/agentiqa-nightly.yml
name: Agentiqa Nightly QA
on:
  schedule:
    - cron: '0 3 * * *' # 03:00 UTC daily
  workflow_dispatch: # also run on demand from the Actions tab
jobs:
  qa:
    runs-on: ubuntu-latest
    # A composite action cannot set its own timeout; GitHub's job default is
    # 360 min. Cap the job so a hung run fails fast instead of burning minutes.
    timeout-minutes: 30
    steps:
      - id: agentiqa
        uses: Agentiqa/qa-action@v1
        with:
          service-key: ${{ secrets.AGENTIQA_SERVICE_KEY }}
          fail-on: plan-failure # default: plan failures + config errors fail; retryable infra doesn't
      - name: Summarize
        if: always()
        run: |
          echo "Outcome: ${{ steps.agentiqa.outputs.outcome }}"
          echo "Plans passed/total: ${{ steps.agentiqa.outputs.plans-passed }}/${{ steps.agentiqa.outputs.plans-total }}"
```

A nightly `schedule` (plus `workflow_dispatch` for on-demand runs) is the intended shape: it tracks current product behavior and matches how Agentiqa runs are metered. Pin `uses: Agentiqa/qa-action@v1` to ride the moving `v1` major, or pin an exact tag (`@v1.0.0`) for byte-stable action behavior.

---

## Inputs

| Input              | Required                    | Default                           | Description                                                                                                 |
| ------------------ | --------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `service-key`      | **yes**                     | —                                 | Project-scoped CLI service key (`sk_...`). Pass from a secret, never inline.                                 |
| `api-url`          | no                          | `''` (prod: `agentiqa.com`)       | Control-plane API override (`AGENTIQA_API_URL`). Set `https://s.agentiqa.com` for staging.                  |
| `runtime`          | no                          | `cloud`                           | _Advanced._ Execution engine location; defaults to `cloud` (browser on Agentiqa's infrastructure, managed LLM). Leave unset for the standard hosted flow. |
| `engine-url`       | no                          | `''` (cloud default engine)       | Hosted-engine URL override; only meaningful with `runtime: cloud`. Empty → `https://engine.agentiqa.com`.   |
| `gemini-api-key`   | no                          | `''`                              | _Advanced._ Ignored for cloud runs, which use Agentiqa's managed LLM.                                        |
| `plan-id`          | no                          | `''`                              | Run a single plan by id (`tplan_...`). Overrides `label-ids`.                                                |
| `label-ids`        | no                          | `''`                              | Comma-separated label ids — run all plans tagged with any of them.                                          |
| `mode`             | no                          | `sequential`                      | `sequential` or `parallel`.                                                                                  |
| `cli-version`      | no                          | `latest`                          | Agentiqa CLI version: npm dist-tag (`latest`, `staging`) or exact semver (e.g. `1.4.2`).                     |
| `artifacts-dir`    | no                          | `agentiqa-artifacts`              | Directory (relative to the workspace) for artifacts + the JSON envelope.                                     |
| `fail-on`          | no                          | `plan-failure`                    | Gating policy: `never` / `plan-failure` / `any` (see below).                                                 |
| `node-version`     | no                          | `22`                              | Node.js version to set up on the runner (18+).                                                               |
| `upload-artifacts` | no                          | `true`                            | Upload the artifacts dir (screenshots / video / result JSON) as a build artifact.                           |
| `artifact-name`    | no                          | `''` → `agentiqa-artifacts-<run>` | Name for the uploaded artifact. Defaults to `agentiqa-artifacts-<run_id>-<attempt>` when empty.             |
| `share-links`      | no                          | `false`                           | **Deprecated no-op**, accepted so existing workflows keep working. Public share links were retired in favor of org-member team access: the step summary's Details column always links a cloud run's detail page (Agentiqa login in the run owner's org required). |
| `retention-days`   | no                          | `14`                              | Retention (days) for the uploaded artifact. Applies when `upload-artifacts: true`; the repo/org artifact-retention cap still bounds it. |
| `review`           | no                          | `false`                           | `true`: QA Manager reviews the batch after the run (see [QA review after the run](#qa-review-after-the-run)). Never changes the outcome or the gate. |
| `deployed-sha`     | with `review: true`         | `''`                              | The commit of the app this run tested (deployed where the plans ran); full/short sha or tag in the checkout. |
| `review-since`     | no                          | `''`                              | Where plans without a starting point start (`--since`); needed on the first review. Keep it recent: it widens the whole review. |
| `review-dry-run`   | no                          | empty (saves)                     | `true`: nothing is added to your project (`--dry-run`). For the first nights. With `review-only: true`, only `false` written out saves. Every review sends one copy, with a trace of how it reached its cards, to the Agentiqa team; deleted after 30 days. |
| `review-trace-hunks` | no | `false` | Accepted for compatibility; changes nothing (`--trace-hunks`): every copy carries the diff hunks each decision rested on. |
| `review-timeout-minutes` | no                    | `40`                              | The most minutes the review may take; a review still running then is stopped (`review-exit-code: 124`). Keep it below the time the job has left after its longest run. |
| `review-only`      | no                          | `false`                           | `true`: no plan runs; the setup check, then a review of an earlier batch, dry unless `review-dry-run: false` (see [Setup check and review-only](#setup-check-and-review-only)). The job fails on a missing line. |
| `review-batch`     | no                          | `''`                              | With `review-only: true`: the batch to review; empty = the project's latest finished batch with a run. |

## Outputs

| Output                 | Example                                                    | Description                                                                                                                                                              |
| ---------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `outcome`              | `passed` / `plan-failure` / `needs-review` / `config-error` / `infra-error` | Outcome bucket used for gating. Exit 0 with zero executed plans buckets as `config-error`.                                                                              |
| `json-path`            | `.../agentiqa-artifacts/agentiqa-result.json`              | Path to the JSON envelope (`schemaVersion:1`); empty if the run was not attempted.                                                                                      |
| `exit-code`            | `0` / `1` / `2` / `3`                                      | Raw CLI process exit code.                                                                                                                                              |
| `plans-total`          | `4`                                                        | Plans reported in the envelope.                                                                                                                                         |
| `plans-passed`         | `3`                                                        | Plans with `outcome == passed`.                                                                                                                                         |
| `plans-failed`         | `1`                                                        | Plans with `outcome != passed`.                                                                                                                                         |
| `artifact-name`        | `agentiqa-artifacts-123-1`                                 | Resolved artifact name used for the upload.                                                                                                                             |
| `cli-resolved-version` | `1.1.32`                                                   | CLI version actually executed, resolved from `agentiqa --version` at run time (see [Reproducibility](#reproducibility-and-version-attribution)). Empty if capture failed. |
| `batch-run-id`         | `batch_…`                                                  | The batch the run recorded (the envelope's top-level `batchRunId`); empty when none. |
| `review-exit-code`     | `0` / `2` / `3` / `124`                                    | QA review exit code: finished / could not run (no retry helps) / transient / stopped at `review-timeout-minutes`. Empty when `review` is off. |
| `review-json-path`     | `.../agentiqa-artifacts/agentiqa-review.json`              | The review's JSON envelope; empty when the review did not run. |
| `setup-check`          | `ok` / `missing`                                           | The review setup check; empty when the review step did not run. |

---

## Exit-code contract and `fail-on`

The CLI exits per a fixed contract: `0` pass, `1` plan failure, `2` usage/config error, `3` infra/runtime error. The action maps exits to **outcome** buckets:

| CLI exit                         | `outcome`      | Meaning                                                                                                                                                   |
| -------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0` (valid envelope, ≥1 plan)    | `passed`       | All selected plans passed.                                                                                                                                |
| `0` with **zero plans**          | `config-error` | A run that executes zero plans is a configuration error, not a pass (empty project, or a selector matching nothing).                                       |
| `0` without a **valid envelope** | `config-error` | Exit 0 must be backed by exactly one JSON object with `ok:true` and a `plans` array — missing/polluted stdout or `ok:false` is a false green, not a pass. |
| `1`                              | `plan-failure` | A plan failed / errored during the run.                                                                                                                   |
| `2` (or unknown)                 | `config-error` | Usage/config error, or an account / plan-limit block (e.g. `org_cap_exhausted`) — permanent until a human fixes it.                                       |
| `3`                              | `infra-error`  | Retryable infra/runtime error (engine unreachable, auth/exchange failure, a lost engine connection) — **or** a no-verdict batch (see below).              |
| `3` with envelope `outcome: needs-review` | `needs-review` | No plan failed, but at least one plan has steps that need a person's review before the run counts as passed (see below). Not a pass, and not an infra error. |

`fail-on` decides which outcomes fail the job:

| `fail-on`                | `plan-failure` | `needs-review` | `config-error` | `infra-error` | Use for                                                                          |
| ------------------------ | -------------- | -------------- | -------------- | ------------- | -------------------------------------------------------------------------------- |
| `never`                  | no             | no             | no             | no            | Report-only (a digest reads `outcome`).                                          |
| `plan-failure` (default) | **yes**        | **yes**        | **yes**        | no            | Nightly — real failures and misconfig fail loudly; retryable infra is swallowed. |
| `any`                    | **yes**        | **yes**        | **yes**        | **yes**       | Release gate — block on anything nonzero.                                        |

A few things worth internalizing:

- **Config errors always fail (unless `fail-on: never`).** A missing/typo'd service key or a bad `label-ids` is permanent — swallowing it would keep a nightly green forever while testing nothing.
- **Zero plans is never a pass.** If the CLI exits `0` but the JSON envelope reports `plans` length `0` (empty project, selector matching nothing), the action buckets the run as `config-error` and fails it under the default policy. The CLI itself exits `2` (`no_plans`, → `config-error`) on an empty project; the action's zero-plans guard also re-buckets any exit-`0` / zero-plan envelope (from older CLIs) to `config-error`, so the outcome is the same either way.
- **Exit 0 requires a valid envelope.** The action validates stdout strictly — exactly one JSON object (multi-doc or junk-polluted stdout fails validation), `ok:true`, and a `plans` array. An exit-0 run that cannot prove what it did via the envelope buckets as `config-error`.
- **Infra errors are swallowed by the default policy.** The CLI exits `3` for a retryable infra/runtime error (engine unreachable, service-key exchange/auth failure). The action buckets that as `infra-error`, and `fail-on: plan-failure` (default) warns-but-does-not-fail on it (retrying is the right move, not a red build). To gate on it too, use `fail-on: any`, or read the `outcome` step output and branch yourself.
- **A no-verdict batch also exits `3`, not `1`.** When every non-passing plan in a batch could not reach a verdict — a persistent engine disconnect (after the per-plan retry budget is exhausted), or the blocked cascade of one — the CLI classifies the whole batch as infra (exit `3`), because a transient WebSocket drop is not a product regression. A batch that also contains a genuine failing verdict still exits `1` (the real failure dominates). So under the default policy a flaky-disconnect nightly shows an `infra-error` warning instead of a red `plan-failure`; switch to `fail-on: any` if you want disconnects to fail the job.
- **A plan that needs review is not a pass.** When no plan failed but some plan has steps that need a person to look at them (run status `needs_review`), the CLI exits `3` and the envelope's top-level `outcome` is `needs-review`. The action buckets the run as `needs-review` — not the retryable `infra-error` — so the default `fail-on: plan-failure` and `fail-on: any` fail the job; only `fail-on: never` lets it through. The step summary marks those plans "needs review" with a warning annotation, not as failed. Open the run to review the steps.
- **A quota / org-cap block fails loudly, and the gate says why.** A run rejected _before any plan starts_ because the account or organization has hit a limit is a permanent, human-fix condition — nothing executed. The CLI exits `2` (→ `config-error`) and puts the server's reason in the envelope's `error.code`, so the default `fail-on: plan-failure` gate fails the job instead of swallowing it as retryable `infra-error`. The gate message then names the real fix instead of pointing at your workflow inputs: `org_cap_exhausted` means the organization's run credits are used up or have expired (contact Agentiqa support — nothing in the workflow can clear it); `credits_exhausted` means the account's included usage and credit balance are used up (upgrade the plan — or buy credits where the plan offers them — from the limit dialog in the web app, or wait for the next monthly window); other plan-limit codes are reported with the CLI's own message. (A quota block hit mid-run, after some plans already reached a verdict, still surfaces per plan as a no-verdict outcome and can exit `3`.)

---

## QA review after the run

With `review: true`, a step after the run summary runs
`agentiqa review --batch <batch-run-id> --head <deployed-sha> [--since <review-since>] [--dry-run] --summary <file> --json`
in the job's working directory, which must be a full-history checkout of the app's repository
(`actions/checkout` with `fetch-depth: 0`) holding `deployed-sha`. QA Manager reads what changed since the
last review and what the run saw, and proposes test edits, possible bugs and tests to add as cards; a person
decides each card. The step:

- runs after a red run too, and before the artifact upload, so `agentiqa-review.json` and
  `agentiqa-review-summary.md` ship in the artifact;
- runs on Agentiqa's managed model access (no model key needed) and passes `GITHUB_TOKEN` so the review can
  read the pull requests behind the commits — give the job `contents: read` and `pull-requests: read`
  (a `permissions:` block sets every scope it does not list to none);
- resolves `deployed-sha` / `review-since` (short sha, tag or ref) to full shas with `git rev-parse`;
- appends the review's job-summary section under the run table;
- never fails: it warns, writes `review-exit-code` / `review-json-path`, and the gate never reads them;
- stops a review that runs past `review-timeout-minutes` (default 40; the CLI itself gets 2 minutes less as
  `AGENTIQA_REVIEW_MAX_MINUTES` unless the job sets it, so it wraps up first): TERM at the limit, KILL 10 s later,
  a warning and `review-exit-code: 124`. A job that reaches its own `timeout-minutes` is cancelled, gate
  included, so the limit must fit in what the job has left after its longest run.

The steps you add before the Action only for the review (the checkout, finding the deployed commit) need a
`timeout-minutes` of their own. A step that fails skips every later step, the Action included, so no plans
run that night: check the setup by hand on a day run before the first night (below).

With `review-dry-run: true` nothing is added to your Agentiqa project (no review, suggestion, issue or
annotation; no watermark moves). Every review, dry or saved (a saved one after its writes), sends one copy
of the review to the Agentiqa team: the cards, the summary, and a trace of how the review reached them
(changed file paths, commit subjects, pull request titles, how plans were linked to files, the AI model's
reasoning, the diff hunks each decision rested on; secret-looking values redacted, hunks capped), and the
change journal the next night starts from. It holds no whole files or prompts; like a saved card, it can
quote the short text cited as evidence. Only the Agentiqa team reads it, and it is deleted after 30 days.
The hunks never reach the project, the job summary or the review JSON. A copy that does not go out is a
warning; it never changes the review's exit code.

It does not start (warning, `review-exit-code: 2`) without a batch, without `deployed-sha`, outside a git
checkout, or when the checkout lacks the commit (a checkout with no commit at all, which is what a failed
`actions/checkout` leaves, is named as such); a shallow checkout only warns. Docs:
[docs.agentiqa.com/docs/qa-review](https://docs.agentiqa.com/docs/qa-review).

```yaml
permissions:
  contents: read
  pull-requests: read
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0
  - uses: Agentiqa/qa-action@v1
    with:
      service-key: ${{ secrets.AGENTIQA_SERVICE_KEY }}
      review: true
      deployed-sha: ${{ github.sha }}
      review-dry-run: true # the first nights: show the cards, save nothing
```

### Setup check and review-only

The review step starts with a setup check: one line per prerequisite (`ok`, `missing` or `unchecked`) and
the workflow line that fixes each missing one. It checks `review` and `review-dry-run` (exactly `true` or
`false`), a full-history checkout of this repository at the workspace root, `deployed-sha` and
`review-since` as commits of it (`review-since` on the history of `deployed-sha`), the service key and its
project, managed model access, the job token's pull-request read, and in review-only the batch. No model
call, no write. The step writes the `setup-check` output.

What no line proves: that the checked-out repository is your app's. A workflow that lives in a separate
repository and checks itself out passes every line, so keep the nightly in the app's repository.

`review-dry-run` is fail-safe: only `false` (or empty, outside review-only) saves cards. Any other value, such as a quoted
`'True'`, runs the review dry, and the check flags it.

- **Nightly** (`review: true`): the table goes above the review section only when a line is missing, as
  warnings. Nothing else changes.
- **Review-only** (`review-only: true`, a run you start by hand before the first night): no plan runs. The
  table comes first in the summary, the review takes `review-batch` or the latest finished batch with a
  run (a newer batch that finished with no run is skipped and named), and the last step fails the job
  unless every line is ok and the review finished. It is a dry run unless `review-dry-run: false` is
  written out: a nightly review job that runs no plan of its own sets it once its review goes live, and
  then saves its cards.

The recipe (the lines to add to your nightly, how to start the check):
[docs.agentiqa.com/docs/qa-review#before-the-first-night](https://docs.agentiqa.com/docs/qa-review#before-the-first-night).

## Reproducibility and version attribution

`cli-version: latest` (the default) always runs the newest published CLI — the right choice for **nightlies**, which should track current product behavior. The trade-off: a verdict can change between two runs purely because the CLI advanced, with nothing in your workflow changing. For **release-gating** workflows, pin an exact version instead — `cli-version: "1.1.32"` (or your chosen `1.1.x`) — so verdicts only change when you deliberately bump it.

`latest` stays the default because attribution, not pinning, is the primary guard: the action records the CLI build that actually ran and surfaces it three ways — echoed in the job log, appended to the step-summary stat line (`Agentiqa · 8 / 8 plans passed · outcome passed · v1.1.32`), and exposed as the `cli-resolved-version` output — so verdict drift across publishes is attributable instead of silent.

---

## Sequential vs parallel

`mode` defaults to `sequential`: plans run one at a time and per-run memory threads forward across them in order, so a later plan can build on state an earlier one established. Set `mode: parallel` for **independent** plans that share no ordering — they run concurrently (bounded to 4 at a time), which cuts wall-clock on a larger suite. There is **no** cross-plan memory threading in parallel, so keep any suite whose plans depend on run order on the default `sequential`.

---

## Artifacts and video

- The action always uploads `artifacts-dir` (default `agentiqa-artifacts`) — `if-no-files-found: warn`, retention `retention-days` (default `14`, bounded by the repo/org cap) — including the `agentiqa-result.json` envelope, screenshots, and any recorded video. Disable with `upload-artifacts: false`.
- **Video is recorded server-side.** On cloud runs the session video is recorded on Agentiqa's infrastructure — no runner-side ffmpeg is needed — and its public URL surfaces as `videoUrl` in the envelope (and on the run detail page linked from the step summary's Details column).
- **Share links are retired.** `share-links` is accepted as a deprecated no-op so existing workflows keep working. Public share links were retired in favor of org-member team access: the step summary's Details column always links a cloud run's detail page, and whoever opens it needs an Agentiqa login in the run owner's organization.

---

## More examples

### Release gate (block on anything nonzero)

```yaml
# .github/workflows/agentiqa-release-gate.yml
name: Agentiqa Release Gate
on:
  push:
    branches: [release/**]
jobs:
  qa:
    runs-on: ubuntu-latest
    timeout-minutes: 30 # composite actions can't set their own timeout; default is 360 min
    steps:
      - uses: Agentiqa/qa-action@v1.0.0 # exact pin for a release gate
        with:
          service-key: ${{ secrets.AGENTIQA_SERVICE_KEY }}
          label-ids: lbl_smoke # only the release-blocking plans
          cli-version: '1.1.32' # pin the CLI too, so verdicts change only on a deliberate bump
          fail-on: any # block the release on plan, config, OR infra failure
```

### Consuming the JSON envelope

```yaml
- id: agentiqa
  uses: Agentiqa/qa-action@v1
  with:
    service-key: ${{ secrets.AGENTIQA_SERVICE_KEY }}
    fail-on: never
- name: Inspect envelope
  run: jq . "${{ steps.agentiqa.outputs.json-path }}"
```

The envelope schema (`schemaVersion:1`) — one JSON object on stdout, all logs on stderr — looks like:

```jsonc
// success
{ "ok": true, "schemaVersion": 1, "outcome": "passed", "batchUrl": "https://web.agentiqa.com/projects/proj_.../batch-runs/batch_...", "plans": [ { "title": "...", "outcome": "passed", "durationSec": 42, "exitCode": 0 } ] }
// failure
{ "ok": false, "schemaVersion": 1, "error": { "code": "run_error", "message": "..." } }
```

The top-level **`batchUrl`** is the page for the whole invocation — every plan of this run, grouped, with per-run outcomes and reasons. The step summary renders it as a **View full results** link under the stat line, noting it is available to every member of your Agentiqa org. It is present only for hosted runs that minted a batch record, omitted otherwise (additive, so `schemaVersion` stays `1`), and — like `runUrl` — viewable only by a signed-in project/org member.

On **cloud** runs each passing/failing plan may additionally carry two links: `runUrl`, a deep link to the run's detail page in the Agentiqa web app (rendered in the step-summary table's Details column) — **viewable only by a signed-in project/org member**; and `videoUrl`, the engine's session recording — a **public, durable link** anyone with the URL can open (safe to embed, but treat it as public; surfaced in the JSON envelope and on the run detail page, not in the summary table).

---

## License

[MIT](./LICENSE) © Agentiqa

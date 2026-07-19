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
| `share-links`      | no                          | `false`                           | Mint an opt-in **public**, revocable share link for each cloud run (`--share`) so reviewers can open a run from the step summary without an Agentiqa login. Privacy-conservative default: off. Cloud runtime only. |
| `retention-days`   | no                          | `14`                              | Retention (days) for the uploaded artifact. Applies when `upload-artifacts: true`; the repo/org artifact-retention cap still bounds it. |

## Outputs

| Output                 | Example                                                    | Description                                                                                                                                                              |
| ---------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `outcome`              | `passed` / `plan-failure` / `config-error` / `infra-error` | Outcome bucket used for gating. Exit 0 with zero executed plans buckets as `config-error`.                                                                              |
| `json-path`            | `.../agentiqa-artifacts/agentiqa-result.json`              | Path to the JSON envelope (`schemaVersion:1`); empty if the run was not attempted.                                                                                      |
| `exit-code`            | `0` / `1` / `2` / `3`                                      | Raw CLI process exit code.                                                                                                                                              |
| `plans-total`          | `4`                                                        | Plans reported in the envelope.                                                                                                                                         |
| `plans-passed`         | `3`                                                        | Plans with `outcome == passed`.                                                                                                                                         |
| `plans-failed`         | `1`                                                        | Plans with `outcome != passed`.                                                                                                                                         |
| `artifact-name`        | `agentiqa-artifacts-123-1`                                 | Resolved artifact name used for the upload.                                                                                                                             |
| `cli-resolved-version` | `1.1.32`                                                   | CLI version actually executed, resolved from `agentiqa --version` at run time (see [Reproducibility](#reproducibility-and-version-attribution)). Empty if capture failed. |

---

## Exit-code contract and `fail-on`

The CLI exits per a fixed contract: `0` pass, `1` plan failure, `2` usage/config error, `3` infra/runtime error. The action maps exits to **outcome** buckets:

| CLI exit                         | `outcome`      | Meaning                                                                                                                                                   |
| -------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0` (valid envelope, ≥1 plan)    | `passed`       | All selected plans passed.                                                                                                                                |
| `0` with **zero plans**          | `config-error` | A run that executes zero plans is a configuration error, not a pass (empty project, or a selector matching nothing).                                       |
| `0` without a **valid envelope** | `config-error` | Exit 0 must be backed by exactly one JSON object with `ok:true` and a `plans` array — missing/polluted stdout or `ok:false` is a false green, not a pass. |
| `1`                              | `plan-failure` | A plan failed / errored during the run.                                                                                                                   |
| `2` (or unknown)                 | `config-error` | Usage/config error — permanent until a human fixes it. On newer CLIs this also covers a **pre-run quota / org-cap** rejection (nothing executed).          |
| `3`                              | `infra-error`  | Retryable infra/runtime error (engine unreachable, auth/exchange failure, quota) — **or** a no-verdict batch (see below).                                 |

`fail-on` decides which outcomes fail the job:

| `fail-on`                | `plan-failure` | `config-error` | `infra-error` | Use for                                                                          |
| ------------------------ | -------------- | -------------- | ------------- | -------------------------------------------------------------------------------- |
| `never`                  | no             | no             | no            | Report-only (a digest reads `outcome`).                                          |
| `plan-failure` (default) | **yes**        | **yes**        | no            | Nightly — real failures and misconfig fail loudly; retryable infra is swallowed. |
| `any`                    | **yes**        | **yes**        | **yes**       | Release gate — block on anything nonzero.                                        |

A few things worth internalizing:

- **Config errors always fail (unless `fail-on: never`).** A missing/typo'd service key or a bad `label-ids` is permanent — swallowing it would keep a nightly green forever while testing nothing.
- **Zero plans is never a pass.** If the CLI exits `0` but the JSON envelope reports `plans` length `0` (empty project, selector matching nothing), the action buckets the run as `config-error` and fails it under the default policy. Older CLI releases return exit `0` on a zero-plan project — so the action's built-in guard is what re-buckets that to `config-error`. Newer CLI releases exit `2` (config-error) directly on an empty project; the action's guard applies either way, so the outcome is the same.
- **Exit 0 requires a valid envelope.** The action validates stdout strictly — exactly one JSON object (multi-doc or junk-polluted stdout fails validation), `ok:true`, and a `plans` array. An exit-0 run that cannot prove what it did via the envelope buckets as `config-error`.
- **Infra errors are swallowed by the default policy.** The CLI exits `3` for a retryable infra/runtime error (engine unreachable, service-key exchange/auth failure, quota). The action buckets that as `infra-error`, and `fail-on: plan-failure` (default) warns-but-does-not-fail on it (retrying is the right move, not a red build). To gate on it too, use `fail-on: any`, or read the `outcome` step output and branch yourself.
- **A no-verdict batch also exits `3`, not `1`.** When every non-passing plan in a batch could not reach a verdict — a persistent engine disconnect (after the per-plan retry budget is exhausted), or the blocked cascade of one — the CLI classifies the whole batch as infra (exit `3`), because a transient WebSocket drop is not a product regression. A batch that also contains a genuine failing verdict still exits `1` (the real failure dominates). So under the default policy a flaky-disconnect nightly shows an `infra-error` warning instead of a red `plan-failure`; switch to `fail-on: any` if you want disconnects to fail the job.
- **Pre-run quota / org-cap rejection fails loudly (newer CLIs).** A run rejected _before any plan starts_ because the account or org has hit its run cap / quota is a permanent, human-fix condition — nothing executed. Newer CLI releases classify that pre-run rejection as a usage/config error (exit `2` → `config-error`), so the default `fail-on: plan-failure` gate fails the job loudly rather than swallowing it as retryable `infra-error`. (A quota block hit mid-run still surfaces as `infra-error` / exit `3`.)

---

## Reproducibility and version attribution

`cli-version: latest` (the default) always runs the newest published CLI — the right choice for **nightlies**, which should track current product behavior. The trade-off: a verdict can change between two runs purely because the CLI advanced, with nothing in your workflow changing. For **release-gating** workflows, pin an exact version instead — `cli-version: "1.1.32"` (or your chosen `1.1.x`) — so verdicts only change when you deliberately bump it.

`latest` stays the default because attribution, not pinning, is the primary guard: the action records the CLI build that actually ran and surfaces it three ways — echoed in the job log, appended to the step-summary headline (`8 / 8 plans passed · CLI exit 0 · agentiqa v1.1.32`), and exposed as the `cli-resolved-version` output — so verdict drift across publishes is attributable instead of silent.

---

## Sequential vs parallel

`mode` defaults to `sequential`: plans run one at a time and per-run memory threads forward across them in order, so a later plan can build on state an earlier one established. Set `mode: parallel` for **independent** plans that share no ordering — they run concurrently (bounded to 4 at a time), which cuts wall-clock on a larger suite. There is **no** cross-plan memory threading in parallel, so keep any suite whose plans depend on run order on the default `sequential`.

---

## Artifacts and video

- The action always uploads `artifacts-dir` (default `agentiqa-artifacts`) — `if-no-files-found: warn`, retention `retention-days` (default `14`, bounded by the repo/org cap) — including the `agentiqa-result.json` envelope, screenshots, and any recorded video. Disable with `upload-artifacts: false`.
- **Video is recorded server-side.** On cloud runs the session video is recorded on Agentiqa's infrastructure — no runner-side ffmpeg is needed — and its public URL surfaces as `videoUrl` in the envelope and the step-summary table.
- **Public share links (opt-in).** `share-links: true` mints an opt-in **public**, revocable share link for each cloud run (`--share`) so reviewers can open a run from the step summary without an Agentiqa login. The default is `false` (privacy-conservative). Cloud runtime only.

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
{ "ok": true, "schemaVersion": 1, "outcome": "passed", "plans": [ { "title": "...", "outcome": "passed", "durationSec": 42, "exitCode": 0 } ] }
// failure
{ "ok": false, "schemaVersion": 1, "error": { "code": "run_error", "message": "..." } }
```

On **cloud** runs each passing/failing plan may additionally carry two links (both rendered in the step-summary table): `runUrl`, a deep link to the run's detail page in the Agentiqa web app — **viewable only by a signed-in project/org member**; and `videoUrl`, the engine's session recording — a **public, durable link** anyone with the URL can open (safe to embed, but treat it as public). With `share-links: true`, a public `shareUrl` is also minted per run and preferred in the Run column.

---

## License

[MIT](./LICENSE) © Agentiqa

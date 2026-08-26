# Shannon Pentest Action

Run [Shannon](https://github.com/KeygraphHQ/shannon) — Keygraph's autonomous white-box AI pentester — against a target URL and its source code, directly from a GitHub Actions workflow. This action is a thin wrapper around the [`@keygraph/shannon`](https://www.npmjs.com/package/@keygraph/shannon) CLI.

Shannon spins up a local Temporal stack and an ephemeral worker container, drives a five-phase pipeline (pre-recon → recon → vulnerability analysis → exploitation → reporting) with 13 AI agents, and produces a security assessment report (PDF + Markdown) plus an optional SARIF log.

## Requirements

- A runner with **Docker** (daemon + `docker compose` v2) and network access to Docker Hub. `ubuntu-latest` GitHub-hosted runners satisfy this out of the box.
- **Node.js ≥ 18** (this action sets it up for you).
- **Do not** run the job inside a `container:` and **do not** invoke via `sudo` — the Shannon CLI refuses to run as root.
- Point Shannon at a **staging or development** environment you own — it actively exploits vulnerabilities, so it must never target production or systems you are not authorized to test.
- An AI provider API key (see [Credentials](#credentials)).

## Usage

```yaml
name: Security scan
on:
  workflow_dispatch:

jobs:
  pentest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Shannon
        uses: KeygraphHQ/shannon-action@v1
        with:
          url: http://localhost:3000
          repo: ${{ github.workspace }}
          api-key: ${{ secrets.SHANNON_AI_API_KEY }}
```

The scan runs to completion (`--follow`). The job **passes only on a complete assessment**: it fails if the scan produced no report, or if any vulnerability class could not be assessed (a partial run — because absence of findings for an unassessed class is not a clean result). Two artifacts are uploaded:

- `shannon-report-<workspace>` — the report deliverables bundled together: `Security-Assessment-Report.pdf`, `Security-Assessment-Report.md`, and `report.sarif` (when the config produces one).
- `shannon-run-<workspace>` — the full run folder (logs, agent transcripts, deliverables, browser artifacts) for debugging, uploaded even on failure. The target's saved auth session (`auth-state.json`) is excluded.

GitHub wraps every artifact in a zip on download, so `shannon-report-<workspace>` downloads as a single zip holding the three deliverables (bundling avoids a pointless one-file zip per report). Artifact names must be unique within a run, so if you run this action in a **matrix**, give each leg a distinct `workspace`.

## Examples

Ready-to-copy workflows in [`examples/`](./examples):

- [`weekly-scan.yml`](./examples/weekly-scan.yml) — scheduled weekly deep scan of a deployed environment, findings sent to code scanning.
- [`pr-scan.yml`](./examples/pr-scan.yml) — scan a pull request and block it on a demonstrated critical.
- [`release-scan.yml`](./examples/release-scan.yml) — assess the staging deployment when a release is published.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | yes | — | Target URL to test (the running application). |
| `repo` | | `${{ github.workspace }}` | Path to the source repo to analyze (white-box). |
| `config` | | — | Path to a Shannon YAML config (auth, scope, report options). |
| `workspace` | | `${{ github.run_id }}` | Named workspace; sets the output dir. Defaults to the run id so each run is isolated. Set a stable value to resume across runs (and to give matrix legs distinct names). |
| `model` | | `anthropic:claude-sonnet-4-6` | Model spec `<provider>:<model-id>` (`SHANNON_AI_MODEL`). |
| `base-url` | | — | Override the provider endpoint (`SHANNON_AI_BASE_URL`). |
| `openai-format` | | `chat-completions` | Wire format for an OpenAI-compatible gateway (`SHANNON_AI_OPENAI_FORMAT`): `chat-completions` or `responses`. Only with `model: openai:...` + `base-url`. |
| `api-key` | | — | LLM provider API key (`SHANNON_AI_API_KEY`) for any non-Bedrock provider, including the default Anthropic model. |
| `aws-bearer-token-bedrock` | | — | Amazon Bedrock bearer token (`AWS_BEARER_TOKEN_BEDROCK`). Required for `model: amazon-bedrock:...`. |
| `aws-region` | | — | AWS region for Bedrock (`AWS_REGION`). Required for `model: amazon-bedrock:...`. |
| `version` | | `2.5.3` | Version of the `@keygraph/shannon` npm package to run. |
| `pipeline-testing` | | `false` | Minimal prompts for fast pipeline testing. |
| `fail-on-severity` | | `none` | Fail the job if any **exploited** finding is at or above this severity: `none`, `low`, `medium`, `high`, `critical`. See [Severity gating](#severity-gating). |
| `upload-sarif` | | `false` | Upload `report.sarif` (when the config produces one) to GitHub code scanning. Requires `security-events: write`. |

## Outputs

| Output | Description |
|---|---|
| `workspace-dir` | Absolute path to the scan workspace directory. |
| `report-pdf` | Path to `Security-Assessment-Report.pdf`, if produced. |
| `report-md` | Path to `Security-Assessment-Report.md`, if produced. |
| `sarif` | Path to `report.sarif`, if produced. |

## Credentials

Pass your LLM provider's API key via `api-key` (it maps to `SHANNON_AI_API_KEY`, which works for any provider the `model` names, including the default):

```yaml
with:
  api-key: ${{ secrets.SHANNON_AI_API_KEY }}
```

For any other plain-API-key provider, set `model` and pass that provider's key through the same input:

```yaml
with:
  model: openai:gpt-5
  api-key: ${{ secrets.OPENAI_API_KEY }}
```

### Amazon Bedrock

Bedrock authenticates via a bearer token plus a region, not a plain API key, so it has dedicated inputs:

```yaml
with:
  model: amazon-bedrock:us.anthropic.claude-sonnet-4-5-20250929-v1:0
  aws-bearer-token-bedrock: ${{ secrets.AWS_BEARER_TOKEN_BEDROCK }}
  aws-region: us-east-1
```

### OpenAI-compatible gateway

Point OpenAI at a proxy/gateway with `base-url`, and pick the wire format if it isn't the default `chat-completions`:

```yaml
with:
  model: openai:gpt-5
  base-url: https://my-gateway.example.com/v1
  openai-format: responses   # or omit for chat-completions
  api-key: ${{ secrets.OPENAI_API_KEY }}
```

## SARIF / code scanning

Shannon emits `report.sarif` only when your config enables exploitation and `report.sarif`. To push that file to GitHub code scanning, set `upload-sarif: true` and grant the job `security-events: write` so the findings can reach the repository's Security tab:

```yaml
permissions:
  security-events: write
# ...
with:
  config: .shannon/ci.yaml
  upload-sarif: true
  api-key: ${{ secrets.SHANNON_AI_API_KEY }}
```

If `upload-sarif` is false (the default) or the config produced no SARIF, this step is skipped. The SARIF file is still included in the report artifact regardless.

## Severity gating

By default the job's pass/fail is decided only by whether the scan completed (see above). To also fail on demonstrated impact, set `fail-on-severity`:

```yaml
with:
  fail-on-severity: high   # fail if any exploited finding is high or critical
```

The gate counts **only findings Shannon actually exploited** (`status: exploited`) — a proven vulnerability, not a suspicion — so it needs no confidence threshold and rarely fires on false positives. Severities rank `critical > high > medium > low`; a finding fails the job when its severity is at or above the threshold.

Because it keys on exploited findings, this gate applies to **exploit-mode scans only**. An analysis-only scan (`exploit: false`) has no exploited findings, so `fail-on-severity` never trips regardless of severity.

## Notes

- A findings summary is written to the run's job summary: exploited counts by severity, or total findings by severity for an analysis-only scan (`exploit: false`), followed by download links to the uploaded artifacts. The job **log** also ends with a plain-text "Next steps" footer carrying the same counts and the artifact download URLs, so it is useful even without opening the Summary tab.
- The first run pulls the `keygraph/shannon` worker image and the Temporal image; subsequent runs on a warm runner are faster.
- Pass/fail is decided from the structured `report.json`: the job passes only when every vulnerability class was assessed. It does **not** fail merely because vulnerabilities were found — gate on findings by inspecting the report if you need that.

## License

This action is licensed under the [Apache License 2.0](./LICENSE). The Shannon CLI it runs (`@keygraph/shannon`) is licensed separately under AGPL-3.0.

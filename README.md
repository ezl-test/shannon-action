# Shannon Pentest Action

Run [Shannon](https://github.com/KeygraphHQ/shannon) — Keygraph's autonomous white-box AI pentester — against a target URL and its source code, directly from a GitHub Actions workflow. This action is a thin wrapper around the [`@keygraph/shannon`](https://www.npmjs.com/package/@keygraph/shannon) CLI.

Shannon spins up a local Temporal stack and an ephemeral worker container, drives a five-phase pipeline (pre-recon → recon → vulnerability analysis → exploitation → reporting) with 13 AI agents, and produces a security assessment report (PDF + Markdown) plus an optional SARIF log.

## Requirements

- A runner with **Docker** (daemon + `docker compose` v2) and network access to Docker Hub. `ubuntu-latest` GitHub-hosted runners satisfy this out of the box.
- **Node.js ≥ 18** (this action sets it up for you).
- **Do not** run the job inside a `container:` and **do not** invoke via `sudo` — the Shannon CLI refuses to run as root.
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

      # Start your app so Shannon has a live URL to test (example).
      # - run: docker compose up -d && ./scripts/wait-for-ready.sh

      - name: Run Shannon
        uses: KeygraphHQ/shannon-action@v1
        with:
          url: http://localhost:3000
          repo: ${{ github.workspace }}
          api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The scan runs to completion (`--follow`). The job **passes only on a complete assessment**: it fails if the scan produced no report, or if any vulnerability class could not be assessed (a partial run — because absence of findings for an unassessed class is not a clean result). Two artifacts are uploaded by default:

- `shannon-report-<workspace>` — just the report (PDF + Markdown, and SARIF if produced).
- `shannon-workspace-<workspace>` — the full workspace (logs, agent transcripts, deliverables, browser artifacts) for debugging, uploaded even on failure. The target's saved auth session (`auth-state.json`) is excluded.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | ✅ | — | Target URL to test (the running application). |
| `repo` | | `${{ github.workspace }}` | Path to the source repo to analyze (white-box). |
| `config` | | — | Path to a Shannon YAML config (auth, scope, report options). |
| `workspace` | | `shannon-ci` | Named workspace; sets the output dir and enables resume. |
| `model` | | `anthropic:claude-sonnet-4-6` | Model spec `<provider>:<model-id>` (`SHANNON_AI_MODEL`). |
| `base-url` | | — | Override the provider endpoint (`SHANNON_AI_BASE_URL`). |
| `openai-format` | | `chat-completions` | Wire format for an OpenAI-compatible gateway (`SHANNON_AI_OPENAI_FORMAT`): `chat-completions` or `responses`. Only with `model: openai:...` + `base-url`. |
| `api-key` | | — | LLM provider API key (`SHANNON_AI_API_KEY`) for any non-Bedrock provider, including the default Anthropic model. |
| `aws-bearer-token-bedrock` | | — | Amazon Bedrock bearer token (`AWS_BEARER_TOKEN_BEDROCK`). Required for `model: amazon-bedrock:...`. |
| `aws-region` | | — | AWS region for Bedrock (`AWS_REGION`). Required for `model: amazon-bedrock:...`. |
| `version` | | `latest` | Version of the `@keygraph/shannon` npm package to run. |
| `pipeline-testing` | | `false` | Minimal prompts for fast pipeline testing. |
| `fail-on-severity` | | `none` | Fail the job if any **exploited** finding is at or above this severity: `none`, `low`, `medium`, `high`, `critical`. See [Severity gating](#severity-gating). |
| `upload-artifact` | | `true` | Upload two artifacts: `shannon-report-<workspace>` (report only) and `shannon-workspace-<workspace>` (full workspace for debugging, minus `auth-state.json`). |

## Outputs

| Output | Description |
|---|---|
| `workspace-dir` | Absolute path to the scan workspace directory. |
| `report-pdf` | Path to `Security-Assessment-Report.pdf`, if produced. |
| `report-md` | Path to `Security-Assessment-Report.md`, if produced. |
| `sarif` | Path to `report.sarif`, if produced. |

## Credentials

Pass your LLM provider's API key via `api-key`. With the default model (`anthropic:claude-sonnet-4-6`) that's your Anthropic key:

```yaml
with:
  api-key: ${{ secrets.ANTHROPIC_API_KEY }}
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

Shannon emits `report.sarif` only when your config enables exploitation and `report.sarif`. Whenever that file is produced, the action uploads it to GitHub code scanning automatically — no separate toggle. Just grant the job `security-events: write` so the findings can reach the repository's Security tab:

```yaml
permissions:
  security-events: write
# ...
with:
  config: .shannon/ci.yaml
  api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

If the config does not request SARIF, this step is skipped.

## Severity gating

By default the job's pass/fail is decided only by whether the scan completed (see above). To also fail on demonstrated impact, set `fail-on-severity`:

```yaml
with:
  fail-on-severity: high   # fail if any exploited finding is high or critical
```

The gate counts **only findings Shannon actually exploited** (`status: exploited`) — a proven vulnerability, not a suspicion — so it needs no confidence threshold and rarely fires on false positives. Severities rank `critical > high > medium > low`; a finding fails the job when its severity is at or above the threshold.

Because it keys on exploited findings, this gate applies to **exploit-mode scans only**. An analysis-only scan (`exploit: false`) has no exploited findings, so `fail-on-severity` never trips regardless of severity.

## Notes

- The first run pulls the `keygraph/shannon` worker image and the Temporal image; subsequent runs on a warm runner are faster.
- Pass/fail is decided from the structured `report.json`: the job passes only when every vulnerability class was assessed. It does **not** fail merely because vulnerabilities were found — gate on findings by inspecting the report if you need that.

## License

See [LICENSE](./LICENSE).

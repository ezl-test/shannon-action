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
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The scan runs to completion (`--follow`) and the job **fails if the pipeline fails**. The report is uploaded as a workflow artifact named `shannon-report` by default.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | ✅ | — | Target URL to test (the running application). |
| `repo` | | `${{ github.workspace }}` | Path to the source repo to analyze (white-box). |
| `config` | | — | Path to a Shannon YAML config (auth, scope, report options). |
| `workspace` | | `shannon-ci` | Named workspace; sets the output dir and enables resume. |
| `output` | | — | Extra directory to copy deliverables into after the run. |
| `model` | | `anthropic:claude-sonnet-4-6` | Model spec `<provider>:<model-id>` (`SHANNON_AI_MODEL`). |
| `base-url` | | — | Override the provider endpoint (`SHANNON_AI_BASE_URL`). |
| `anthropic-api-key` | | — | Anthropic API key (used by the default model). |
| `api-key` | | — | Generic provider key (`SHANNON_AI_API_KEY`) for any non-Bedrock provider. |
| `version` | | `latest` | Version of the `@keygraph/shannon` npm package to run. |
| `node-version` | | `20` | Node.js version to set up. |
| `pipeline-testing` | | `false` | Minimal prompts for fast pipeline testing. |
| `extra-args` | | — | Raw args appended to `shannon start`. |
| `upload-artifact` | | `true` | Upload the workspace (report + logs) as an artifact. |
| `artifact-name` | | `shannon-report` | Name of the uploaded artifact. |
| `upload-sarif` | | `false` | Upload `report.sarif` to code scanning (needs exploit + `report.sarif` in config). |

## Outputs

| Output | Description |
|---|---|
| `workspace-dir` | Absolute path to the scan workspace directory. |
| `report-pdf` | Path to `Security-Assessment-Report.pdf`, if produced. |
| `report-md` | Path to `Security-Assessment-Report.md`, if produced. |
| `sarif` | Path to `report.sarif`, if produced. |

## Credentials

Provide exactly one provider's credential. The simplest is Anthropic (matches the default model):

```yaml
with:
  anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

For any other provider whose credential is a plain API key, use the generic key plus a model spec:

```yaml
with:
  model: openai:gpt-5
  api-key: ${{ secrets.OPENAI_API_KEY }}
```

> Amazon Bedrock (which authenticates via `AWS_BEARER_TOKEN_BEDROCK` + `AWS_REGION`) is not yet exposed as dedicated inputs — open an issue if you need it.

## SARIF / code scanning

When your config enables exploitation and `report.sarif`, set `upload-sarif: true` and grant the job `security-events: write` to surface findings in the repository's Security tab:

```yaml
permissions:
  security-events: write
# ...
with:
  config: .shannon/ci.yaml
  upload-sarif: true
  anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Notes

- The first run pulls the `keygraph/shannon` worker image and the Temporal image; subsequent runs on a warm runner are faster.
- `start --follow` exits non-zero when the **pipeline** fails, not when vulnerabilities are found. Gate on findings by inspecting the report if you need that.

## License

See [LICENSE](./LICENSE).

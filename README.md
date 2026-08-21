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

The scan runs to completion (`--follow`) and the job **fails if the pipeline fails**. The report (PDF + Markdown) is uploaded as a workflow artifact named `shannon-report-<workspace>` by default.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | ✅ | — | Target URL to test (the running application). |
| `repo` | | `${{ github.workspace }}` | Path to the source repo to analyze (white-box). |
| `config` | | — | Path to a Shannon YAML config (auth, scope, report options). |
| `workspace` | | `shannon-ci` | Named workspace; sets the output dir and enables resume. |
| `model` | | `anthropic:claude-sonnet-4-6` | Model spec `<provider>:<model-id>` (`SHANNON_AI_MODEL`). |
| `base-url` | | — | Override the provider endpoint (`SHANNON_AI_BASE_URL`). |
| `api-key` | | — | LLM provider API key (`SHANNON_AI_API_KEY`) for any non-Bedrock provider, including the default Anthropic model. |
| `version` | | `latest` | Version of the `@keygraph/shannon` npm package to run. |
| `pipeline-testing` | | `false` | Minimal prompts for fast pipeline testing. |
| `upload-artifact` | | `true` | Upload the report (PDF + Markdown, and SARIF if produced) as an artifact named `shannon-report-<workspace>`. |

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

For any other provider, set `model` and pass that provider's key through the same input:

```yaml
with:
  model: openai:gpt-5
  api-key: ${{ secrets.OPENAI_API_KEY }}
```

> Amazon Bedrock (which authenticates via `AWS_BEARER_TOKEN_BEDROCK` + `AWS_REGION`, not a plain API key) is not supported through `api-key` — open an issue if you need it.

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

## Notes

- The first run pulls the `keygraph/shannon` worker image and the Temporal image; subsequent runs on a warm runner are faster.
- `start --follow` exits non-zero when the **pipeline** fails, not when vulnerabilities are found. Gate on findings by inspecting the report if you need that.

## License

See [LICENSE](./LICENSE).

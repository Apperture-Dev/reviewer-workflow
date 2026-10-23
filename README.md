# AI Code Review with Conductor

A GitHub Action that reviews every pull request with AI and posts the findings as comments:

- **Inline comments** on the diff lines, with applicable suggestions when it makes sense.
- **A summary comment** that is updated on every push (never duplicated).
- **Full first pass, incremental afterwards:** each push can only produce comments on what changed since the last review.
- **Thread follow-up:** the reviewer replies in its own threads when the author answers or fixes the code.

Orchestration uses [Microsoft Conductor](https://github.com/microsoft/conductor): three reviewers in parallel (correctness, security, performance/maintainability + repository rules) and a verifier that deduplicates and drops false positives. It works with **Claude Code** (subscription or API key), **GitHub Copilot** or any **OpenAI-compatible AI gateway** with your own API key.

This repository only publishes the Action. The code, the issue tracker and the GitLab CI/CD component live at **[gitlab.com/apperturedev/reviewer-workflow](https://gitlab.com/apperturedev/reviewer-workflow)**.

## Usage

`.github/workflows/code-review.yml`:

```yaml
name: code-review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: code-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  code-review:
    if: ${{ !github.event.pull_request.draft && github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-latest
    continue-on-error: true
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false
      - uses: Apperture-Dev/reviewer-workflow@v1
        with:
          provider: claude
        env:
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reviewer-output
          path: reviewer-output/
          retention-days: 7
```

- **`persist-credentials: false` is mandatory:** otherwise the token stays in `.git/config` and the agents could read it.
- Use the `pull_request` event, never `pull_request_target`: the latter runs the pull request's code with access to the secrets.
- The `if` skips drafts (reviewed when marked *ready for review*) and pull requests from forks, which receive no secrets.
- Comments come from `github-actions[bot]` with the workflow's `GITHUB_TOKEN`. With `contents: read` the reviewer replies in its threads but cannot resolve them; GitHub only lets `GITHUB_TOKEN` resolve conversations with `contents: write`. The token is removed from the environment before the agents start either way.

The Action runs a Docker container, so it needs a Linux runner.

## Engines

Pass the engine credentials as `env` of the step, from repository secrets.

| `provider` | What it uses | Credentials | `model` |
|---|---|---|---|
| `claude` | Claude Code (claude-agent-sdk) | `CLAUDE_CODE_OAUTH_TOKEN` (`claude setup-token`, subscription) **or** `ANTHROPIC_API_KEY` | Optional; defaults to `claude-sonnet-5-5` |
| `copilot` | GitHub Copilot with your subscription | `COPILOT_GITHUB_TOKEN`: fine-grained PAT with the **Copilot Requests** permission | Optional; defaults to the account's model |
| `gateway` | GitHub Copilot in BYOK mode against an OpenAI-compatible endpoint (`/chat/completions`): an AI gateway, vLLM, Ollama… | `COPILOT_PROVIDER_API_KEY` plus the `gateway-url` input | **Required**: the exact id from `GET <gateway-url>/models` |

```yaml
      - uses: Apperture-Dev/reviewer-workflow@v1
        with:
          provider: gateway
          gateway-url: https://gateway.example.com/v1
          model: glm-5-2
        env:
          COPILOT_PROVIDER_API_KEY: ${{ secrets.COPILOT_PROVIDER_API_KEY }}
```

The gateway model must support *tool calling*: the reviewers read the checkout with tools.

## Inputs

| Input | Default | Description |
|---|---|---|
| `github-token` | `${{ github.token }}` | Token that publishes the comments; it never reaches the AI agents |
| `provider` | `claude` | `claude`, `copilot` or `gateway` |
| `model` | *(empty)* | Model for every agent. Required with `gateway` |
| `gateway-url` | *(empty)* | Base URL of the OpenAI-compatible endpoint for `gateway` |
| `language` | `en` | Language of the comments (`en`, `es`, …) |
| `rules-files` | `AGENTS.md,CLAUDE.md,REVIEW.md` | Repository files with project rules the review must enforce (category `repo-rules`) |
| `extra-instructions` | *(empty)* | Extra free-text instructions for the reviewers |
| `exclude-paths` | lockfiles, `vendor/*`, `dist/*`, minified files… | Comma-separated globs that are not reviewed |
| `max-findings` | `20` | Maximum number of published findings (most severe first) |
| `max-diff-chars` | `150000` | Larger diffs are truncated at file boundaries and the review says so |
| `output-dir` | `reviewer-output` | Where `findings.json`, the context and the Conductor log are written |

## Commands in commits

A line that **starts** with `@reviewer` in the message of any new commit changes what the reviewer does on that push:

| Command | Effect |
|---|---|
| `full` | Reviews the full pull request diff instead of only the new changes |
| `impact` | Posts a Mermaid impact diagram in its own comment |
| `skip` | Does not review this push (only on the last commit of the push) |
| `focus <area>…` | Runs only the given reviewers: `correctness`, `security`, `quality` |

The full documentation is in the [project README](https://gitlab.com/apperturedev/reviewer-workflow#readme).

## License

[MIT](https://github.com/Apperture-Dev/reviewer-workflow/blob/main/LICENSE)

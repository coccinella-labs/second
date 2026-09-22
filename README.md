<p align="center">
  <img src="https://raw.githubusercontent.com/coccinella-labs/second/main/.github/assets/thumbnail.png" alt="second" width="100%">
</p>

# Second

Model-backed inline PR review through OpenRouter. A second pair of eyes on the diff: comments only on real defects, silent otherwise.

## How model selection works

Per the [OpenRouter docs](https://openrouter.ai/docs/guides/routing/routers/free-router), the default model `openrouter/free` (Free Models Router) auto-selects a free model at random from the available pool, filtered to capabilities the request needs. The response reports which model was actually used.

- Pin a specific free model with the `:free` suffix, e.g. `meta-llama/llama-3.2-3b-instruct:free` (example from the docs; check the models page for what is currently free).
- Add `fallback-models` (comma-separated) tried in order if the primary fails.
- Free quota caveats from the docs: rate limits, availability swings, higher latency at peak. The action treats an empty or failed model response as "no findings" rather than erroring the run.

## Usage

```yaml
- uses: coccinella-labs/second@v1
  with:
    openrouter-key: ${{ secrets.OPENROUTER_API_KEY }}
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| github-token | GitHub token for API access | `github.token` |
| openrouter-key | OpenRouter API key (secret, free quota suffices) | required |
| repository | Repository to review (`owner/name`) | `github.repository` |
| pr-number | PR to review (defaults to the event) | - |
| model | OpenRouter model slug | `openrouter/free` |
| fallback-models | Comma-separated fallback slugs | - |
| max-comments | Maximum inline comments per run | `5` |
| max-diff-chars | Truncate the reviewed diff beyond this | `30000` |
| exclude-paths | Comma-separated path prefixes skipped | `Cargo.lock,package-lock.json,yarn.lock,pnpm-lock.yaml` |
| comment | Post report as PR comment: `always`, `on-failure`, `never` | `on-failure` |

## Outputs

| Output | Description |
|--------|-------------|
| posted | Number of inline comments posted |
| model-used | Model slug OpenRouter reports as actually used |

## Using `openrouter/auto` instead

Pass it as the `model` input; no code change needed, the request shape is identical:

```yaml
- uses: coccinella-labs/second@v1
  with:
    openrouter-key: ${{ secrets.OPENROUTER_API_KEY }}
    model: openrouter/auto
```

Know what changes: per the [Auto Router docs](https://openrouter.ai/docs/guides/routing/routers/auto-router), `openrouter/auto` ranks by market spend and you pay the standard rate of whichever model is selected. It is not free-quota compatible: a $0-capped key cannot pay those rates, so requests will fail or be limited. Use it only with a funded key, optionally with the `auto-router` plugin (`allowed_models`, `excluded_models`, `cost_tier`) to bound selection. The `model-used` output reports what was actually picked.

## Review policy (baked into the prompt)

Comment only on: logic bugs, missing restores a sibling path performs, introduced dead code, unhandled error paths, test gaps on changed behavior. Never style, naming, or preferences. Findings need exact mechanism plus concrete fix, anchored to diff lines.

## Honest limits

- The live model call needs a real key and has not been exercised end to end here; everything around it (request shape, fallback chain, normalization, review construction) is verified.
- Finding quality depends on the selected free model and is unverified; treat early comments skeptically and tighten `max-comments` if noise appears.

## Example

```yaml
name: Second Opinion
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: coccinella-labs/second@v1
        with:
          openrouter-key: ${{ secrets.OPENROUTER_API_KEY }}
```

## License

MIT

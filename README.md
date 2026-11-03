# Nodagentry Agent Cost Check

See what a pull request does to your AI agent's cost before you merge it.

This action comments on every pull request that changes **LangGraph**, **CrewAI** or **OpenAI Agents SDK** code with the estimated cost per run before and after the change, file by file and step by step. Catch the switch to a bigger model, a longer system prompt or an extra step before it reaches your bill.

It reads your code as text. Nothing is run and no model is called.

## What the comment looks like

Here a pull request moves the Writer agent in a CrewAI crew from `gpt-6-luna` to `gpt-6.1-sol`:

> ### Agent cost check
>
> Estimated cost per run for 1 agent file touched by this pull request.
>
> | File | Framework | Before | After | Change |
> | --- | --- | ---: | ---: | ---: |
> | `crew/brief.py` | CrewAI | $0.0094 | $0.0140 | **+50%** 🔺 |
>
> <details open><summary>Steps that changed in <code>crew/brief.py</code></summary>
>
> | Step | Model | Before | After |
> | --- | --- | ---: | ---: |
> | Draft | gpt-6-luna → gpt-6.1-sol | $0.00025 | $0.0049 |
>
> </details>
>
> <sub>Estimated by [Nodagentry](https://nodagentry.com/docs/ci-cost-check) from each step's prompt in your code and its model's list price (replies assumed ~400 tokens, loops counted once, tools free), on the task “A customer writes in: they were charged twice for one order and want a refund. Handle it.”. Not a provider bill.</sub>

## Set it up

1. **Create a token.** Sign in at [nodagentry.com/docs/ci-cost-check](https://nodagentry.com/docs/ci-cost-check) and create a CI token. The check is part of the Nodagentry Monthly plan (US$29 a month).
2. **Add it to the repository.** In GitHub, open Settings, then Secrets and variables, then Actions, and add a repository secret named `NODAGENTRY_TOKEN` with the token as its value.
3. **Add the workflow.** Save this as `.github/workflows/agent-cost.yml` on your default branch.

```yaml
name: Agent cost check
on:
  pull_request:
    paths: ["**/*.py", "**/*.js", "**/*.ts", "**/*.mjs", "**/*.cjs", "**/*.mts", "**/*.cts"]
permissions:
  contents: read
  pull-requests: write
jobs:
  cost:
    runs-on: ubuntu-latest
    steps:
      - uses: wasapilelabs/agent-cost-check@v1
        with:
          token: ${{ secrets.NODAGENTRY_TOKEN }}
          # Optional: fail the check when a file's estimated cost rises by more than this %.
          # fail-above-pct: 20
          # Optional: a typical task for your agent, used for the estimate.
          # task: "Summarise this support ticket and draft a reply"
```

The next pull request that touches agent code gets a comment, and later pushes update the same comment.

## Inputs

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| `token` | yes | | Your Nodagentry CI token, from the `NODAGENTRY_TOKEN` secret. |
| `fail-above-pct` | no | empty | Fail the check when a file's estimated cost per run rises by more than this percentage, for example `20`. Make the check required in branch protection to block the merge. Empty means comment only. |
| `task` | no | empty | A typical request for your agent, up to 500 characters. Longer tasks mean more input tokens in the estimate. Empty means a short support request. |
| `checkout` | no | `true` | Check out the repository with full history first. Set it to `false` if an earlier step already ran `actions/checkout` with `fetch-depth: 0`. |
| `github-token` | no | `github.token` | The token used to post the comment. |

The job needs `pull-requests: write` to post the comment and `contents: read` to read the code.

## What is sent to Nodagentry

Only the files the pull request changes that look like LangGraph, CrewAI or OpenAI Agents SDK code, in their before and after versions. They are parsed in memory to estimate cost and not stored. The whole script is in [`action.yml`](action.yml), so you can read exactly what it sends.

## How it estimates

Each step's input is the prompt it would be sent: the prompt text found in your code (system prompts, CrewAI roles and task descriptions, Agents SDK instructions), the task, and what earlier steps pass on, at about 4 characters per token. Replies can't be known before a run, so each model step is assumed to write about 400 tokens. Well-known models (GPT, Claude, Gemini) are priced at their list price, others at their tier's typical price. Every step is counted once, so loops and retries aren't multiplied, and tools add no model cost. The numbers are for comparing versions, not a provider bill.

## Pull requests from forks

GitHub doesn't give repository secrets to workflows started by pull requests from forks, so the action has no token there and skips the check with a warning instead of failing.

## Limits

- Up to 100 changed agent files per pull request, each up to 200,000 characters.
- 500 checks per token per day, and up to 5 tokens per account.
- Files are matched by path, so a renamed file shows as removed and new.

## Try it without CI

Paste LangGraph, CrewAI, OpenAI Agents SDK or Flowise code into the free [agent cost calculator](https://nodagentry.com/agent-cost-calculator) to see the cost of each step, no account needed.

## Support

Questions or problems: contact@nodagentry.com. [Nodagentry](https://nodagentry.com) is made by Wasapile Labs.

## License

[MIT](LICENSE)

# smart-contracts-agentic-review

Centralized agentic PR review for Solidity / Foundry repositories in
0xPolygon. Every Solidity repo in the org calls a workflow here via a
thin stub. Improvements here ship to every repo on its next PR.

## What's in here

Each setup is a self-contained folder under `.github/workflows/`:

- `.github/workflows/default/` — the default generic Solidity review
  - `ai-review.yml` — the reusable workflow (`workflow_call`)
  - `CLAUDE_PROMPT.md` — Claude's review prompt
  - `CODEX_PROMPT.md` — Codex's review prompt

Specialized setups (ZK circuits, audits, specific protocols, etc.) can
be added as sibling folders. Each has its own `ai-review.yml` plus its
own prompts.

## What's intentionally NOT in here

`CLAUDE.md` / `AGENTS.md` / `README.md` templates are **not** part of
this setup. Those files are per-repo repository context — they belong
in each consuming repo for local Claude sessions, human onboarding,
and auditor context. The prompts instruct the review agents to read
them if they exist, but the centralized CI doesn't own them.

## How a Solidity repo gets wired up

Drop this stub at `.github/workflows/ai-review-stub.yml` in the
consuming repo:

```yaml
name: AI Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]
jobs:
  review:
    uses: 0xPolygon/smart-contracts-agentic-review/.github/workflows/default/ai-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
    secrets: inherit
```

That's it. No other files needed in the consuming repo for the review
to work.

## Per-repo customization

Three axes of customization:

**1. Content customization — edit `CLAUDE.md` / `AGENTS.md` /
`README.md` in the consuming repo.** Optional. If they exist, the
agents read them.

**2. Workflow customization — pass `with:` inputs in the stub.**
Available inputs:

| Input             | Default           | Purpose                     |
| ----------------- | ----------------- | --------------------------- |
| `foundry-version` | `nightly`         | Foundry release tag         |
| `install-soldeer` | `true`            | Run `forge soldeer install` |
| `claude-model`    | `claude-opus-4.7` | Claude model identifier     |
| `codex-model`     | `gpt-5.3-codex`   | Codex model identifier      |
| `run-claude`      | `true`            | Enable Claude reviewer      |
| `run-codex`       | `true`            | Enable Codex reviewer       |

**3. Switching setups — change which folder the stub points at.** To
use a specialized setup (future), change
`.../default/ai-review.yml@main` to `.../zk-circuits/ai-review.yml@main`
or whatever the setup folder is called.

## Branching model

- `dev` — open all PRs here. Test changes.
- `main` — protected. Only merges from `dev` with required approvals.
  Stubs in consuming repos point at `@main`, so anything landing here
  goes live everywhere on the next PR.

### Testing changes before merge

Before merging `dev` → `main`, point one consuming repo's stub at your
branch temporarily:

```yaml
uses: 0xPolygon/smart-contracts-agentic-review/.github/workflows/default/ai-review.yml@my-feature-branch
```

Open a test PR there, verify, revert the stub change, merge
`dev` → `main`.

## Adding a new setup

1. Create `.github/workflows/<setup-name>/` in this repo.
2. Put `ai-review.yml`, `CLAUDE_PROMPT.md`, and `CODEX_PROMPT.md`
   inside it. Start by copying `default/` and editing.
3. Update the prompt paths inside the new `ai-review.yml` to point at
   its own folder (search for `.github/workflows/default/` and replace
   with `.github/workflows/<setup-name>/`).
4. Consuming repos that want the new setup change their stub's
   `uses:` to point at the new folder.

## Secrets

`CLAUDE_API_KEY` and `OPENAI_API_KEY` are 0xPolygon org-level secrets
with visibility set to all repositories. Consuming repos pass them
through with `secrets: inherit`.
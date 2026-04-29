# Smart Contracts Agentic Review

Centralized agentic PR review for Solidity / Foundry repositories in
the 0xPolygon organization. Every Solidity repo calls a workflow
here via a thin stub. Improvements ship to every repo on its next
PR.

## Concept

The system separates **environment** (how the runner is set up) from
**agent** (what the AI is told to look for). Each consuming repo
picks both:

- **Workflow** — picks the environment. `default.yml` provides
  Foundry + Soldeer + Bun. Future workflows (e.g. `legacy.yml`)
  could provide alternative toolchains.
- **Agent** — picks the review specialty. `general` is the default
  elite Solidity security reviewer. Future agents might specialize
  in LayerZero integrations, ERC-4626 vaults, governance audits, etc.

A consuming repo's stub calls a workflow and (optionally) selects a
non-default agent for Claude and/or Codex. Both AIs always run; they
just point at potentially different prompts.

## Repository structure

```
smart-contracts-agentic-review/
├── README.md
├── .github/workflows/
│   └── default.yml             ← Foundry + Soldeer + Bun environment
└── agents/
    └── general/                ← elite generic security review
        ├── CLAUDE_PROMPT.md
        └── CODEX_PROMPT.md
```

GitHub Actions only discovers workflow files at the top level of
`.github/workflows/`, so workflows are flat. Agent directories under
`agents/` can be organized freely.

## Stub for a consuming repo

`.github/workflows/agentic-review-stub.yml`:

```yaml
name: Agentic Review Stub
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
    uses: 0xPolygon/smart-contracts-agentic-review/.github/workflows/default.yml@main
    permissions:
      contents: read
      pull-requests: write
      issues: write
      actions: read
    secrets: inherit
```

That runs `general` on both Claude and Codex. To change the agent on
either or both:

```yaml
    with:
      claude-agent: "layerzero"   # use the layerzero agent for Claude
      codex-agent: "general"      # keep general on Codex
```

If `agents/<name>/CLAUDE_PROMPT.md` (or `CODEX_PROMPT.md`) doesn't
exist, that AI's job fails fast with a clear error. The other AI
still runs.

## Available inputs

| Input             | Default           | Purpose                           |
| ----------------- | ----------------- | --------------------------------- |
| `foundry-version` | `nightly`         | Foundry release tag               |
| `install-soldeer` | `true`            | Run `forge soldeer install`       |
| `claude-model`    | `claude-opus-4-7` | Claude model identifier           |
| `codex-model`     | `gpt-5.3-codex`   | Codex model identifier            |
| `claude-agent`    | `general`         | Folder under `agents/` for Claude |
| `codex-agent`     | `general`         | Folder under `agents/` for Codex  |
| `run-claude`      | `true`            | Enable Claude reviewer            |
| `run-codex`       | `true`            | Enable Codex reviewer             |

## Per-repo customization (recap)

Three axes:

1. **Repo context** — drop `AGENTS.md`, `CLAUDE.md` (with
   `@AGENTS.md` import), `README.md` in the consuming repo. The
   agents read them. Optional but recommended.
2. **Agent choice** — `claude-agent` / `codex-agent` inputs. Pick a
   different specialty per AI if it makes sense.
3. **Workflow inputs** — `foundry-version`, `install-soldeer`, etc.

## What an agent looks like

An agent is a folder under `agents/` containing:

- `CLAUDE_PROMPT.md` — appended to Claude Code's system prompt via
  `--append-system-prompt-file`.
- `CODEX_PROMPT.md` — passed to the Codex action via `prompt-file`.

Both prompts share methodology (Orient → Map → Verify → Hunt →
Devil's Advocate → Report) and adversarial framing, but the Claude
prompt also covers inline-comment posting and the summary-comment
header format, while the Codex prompt covers terse markdown output
that the workflow wraps with a header.

## Adding a new agent

1. `mkdir -p agents/<agent-name>`
2. Create `agents/<agent-name>/CLAUDE_PROMPT.md`. Start by copying
   `agents/general/CLAUDE_PROMPT.md` and adding/replacing the Hunt
   categories with the agent's specialty.
3. Create `agents/<agent-name>/CODEX_PROMPT.md` similarly. (Optional
   — if missing, Codex jobs that try to use this agent will fail
   fast.)
4. PR against `dev`. After approval, merge to `main`.
5. Consuming repos opt in by setting `claude-agent: "<agent-name>"`
   and/or `codex-agent: "<agent-name>"` in their stub.

## Adding a new workflow (environment)

If a repo uses a non-Foundry-Soldeer-Bun toolchain (e.g. Hardhat,
plain `forge install` instead of Soldeer), copy `default.yml` to
`<env-name>.yml`, adjust the dependency-installation steps, and
publish. Consuming repos point their stub at the new file:
`uses: .../workflows/<env-name>.yml@main`.

The agent inputs (`claude-agent`, `codex-agent`) work the same way
across all workflows — they're independent of environment.

## Branching model

- `dev` — open PRs here. Test changes on a feature branch by pointing
  one consuming repo's stub at that branch temporarily.
- `main` — protected. Stubs in consuming repos point at `@main`.
  Anything landing here goes live everywhere on the next PR.

## Why public

The default `GITHUB_TOKEN` in consuming repos cannot check out other
private/internal repos in the org, even when reusable-workflow
access is enabled. Keeping this repo public sidesteps the
GitHub App / PAT detour. Workflow scaffolding and agent prompts are
not sensitive.

## Codex sandbox configuration (heads-up)

The Codex action has two orthogonal knobs:

- `safety-strategy` — privilege control (`drop-sudo` default).
- `sandbox` — filesystem boundary (`workspace-write` default).

`forge build` and `forge test` need `workspace-write` because they
write `cache/` and `out/`. The previous bug we hit:
`safety-strategy: read-only` (privilege knob) was misread as a
filesystem knob and blocked builds. `default.yml` now sets both
explicitly to the correct values.

## Secrets

`CLAUDE_API_KEY` and `OPENAI_API_KEY` are 0xPolygon org-level
secrets, visibility = all repositories. Consuming repos pass them
through with `secrets: inherit`. Rotate centrally.

## Cost expectations

Per PR with the `general` agent on both AIs:

- Claude Opus 4.7 with the elite prompt: ~$0.20–0.40
- GPT-5.3-Codex: ~$0.10–0.20

Combined: ~$0.30–0.60 per PR. ~100 PRs/month → $30–60/month total.
The new prompt is more thorough than the previous one, which is the
reason for the bump.

## What this is NOT

- Not a replacement for human security review or professional audits.
- Not a replacement for `forge fmt` in a separate CI job — that's
  deterministic and cheaper.
- Not interactive on standalone issues — `@claude` works only in PR
  comments and review threads.

## License

This codebase is licensed under Source Available License.

See [`LICENSE.txt`](./LICENSE.txt).

Your use of this software constitutes acceptance of these license terms.
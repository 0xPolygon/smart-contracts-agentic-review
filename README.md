# Smart Contracts: Agentic Review

Centralized agentic PR review for Solidity / Foundry repositories in
the 0xPolygon organization. Every Solidity repo calls a workflow
here via a thin stub. Improvements here ship to every repo on its
next PR.

On every PR opened in a consuming repo:

1. **Claude Opus 4.7** runs a deep security and gas review. Posts
   inline comments on specific diff lines for 🔴/🟡 findings, plus a
   PR-level summary with build/test status, severity counts, and
   🟢/📝/⛽ sections.
2. **GPT-5.3-Codex** runs an independent diff-focused review in
   parallel. Posts a single PR-level summary comment with P0/P1
   findings and `path/File.sol:LINE` citations.

Additionally, any team member can write `@claude <question>` in a PR
comment or review to get an interactive, read-only response from
Claude with full `forge` / `git` / `gh` access.

Both agents are instructed to produce reviews, not edits. Claude
Code runs with a read-only tool set (no `Edit`/`Write`/`git push`).
Codex runs with a `workspace-write` sandbox (needed so `forge build`
can write `cache/` and `out/`) but has no git credentials, so any
workspace changes are ephemeral.

## Repository structure

```
smart-contracts-agentic-review/
├── README.md
├── .github/workflows/
│   └── default.yml                 ← reusable workflow for the default profile
└── prompts/
    └── default/                    ← prompts for the default profile
        ├── CLAUDE_PROMPT.md
        └── CODEX_PROMPT.md
```

Each **profile** is a pair: a workflow file
(`.github/workflows/<profile>.yml`) plus a prompts directory
(`prompts/<profile>/`). The workflow file hardcodes which profile
it uses via the top-level `env.PROFILE`.

GitHub Actions only discovers workflow files at the top level of
`.github/workflows/`, so workflow files cannot be nested into
subfolders. Prompt directories, however, are free to be organized
under `prompts/`.

## How a consuming repo uses this

Drop this stub at `.github/workflows/agentic-review-stub.yml`:

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
      id-token: write
    secrets: inherit
```

That's the entire per-repo setup. No other files are required.

## Per-repo customization

Three axes:

**1. Review content — edit `CLAUDE.md` / `AGENTS.md` / `README.md`
in the consuming repo.** Optional. If they exist, the agents read
them and use them to inform the review. They're repository-context
files, not CI configuration.

**2. Workflow behavior — pass `with:` inputs in the stub.**
Available inputs (from `default.yml`):

| Input             | Default           | Purpose                     |
| ----------------- | ----------------- | --------------------------- |
| `foundry-version` | `nightly`         | Foundry release tag         |
| `install-soldeer` | `true`            | Run `forge soldeer install` |
| `claude-model`    | `claude-opus-4-7` | Claude model identifier     |
| `codex-model`     | `gpt-5.3-codex`   | Codex model identifier      |
| `run-claude`      | `true`            | Enable Claude reviewer      |
| `run-codex`       | `true`            | Enable Codex reviewer       |

**3. Profile choice — change which workflow the stub calls.** For a
future specialized profile (e.g. `audit.yml`), the consuming repo's
stub changes `.../default.yml@main` to `.../audit.yml@main`.

## Branching model

- `dev` — open all PRs here. Test changes.
- `main` — protected. Only merges from `dev` with required
  approvals. Stubs in consuming repos point at `@main`, so anything
  landing here goes live everywhere on the next PR.

### Testing changes before merge

Before merging `dev` → `main`, point one consuming repo's stub at
your branch temporarily:

```yaml
uses: 0xPolygon/smart-contracts-agentic-review/.github/workflows/default.yml@my-feature-branch
```

Open a test PR there, verify, revert the stub change, merge
`dev` → `main`.

## Adding a new profile

1. Create `prompts/<profile-name>/CLAUDE_PROMPT.md` and
   `prompts/<profile-name>/CODEX_PROMPT.md`. Start by copying
   `prompts/default/` and editing.
2. Create `.github/workflows/<profile-name>.yml`. Start by copying
   `default.yml`. Change the `env.PROFILE` value at the top of the
   file to `<profile-name>`. Change the Codex `prompt-file` path
   and the Claude `--append-system-prompt-file` path to point at the
   new profile's prompts.
3. Update the workflow's `name:` field to reflect the profile.
4. Consuming repos that want the new profile change their stub's
   `uses:` to point at `<profile-name>.yml` instead of `default.yml`.

## Visibility

This repo is **public**. The default `GITHUB_TOKEN` in consuming
repos cannot check out other private/internal repos in the org,
even when reusable-workflow access is enabled. Keeping this repo
public sidesteps the GitHub App / PAT detour. The content
(workflow scaffolding + prompts) is not sensitive.

## Prompt delivery mechanism

Prompts are delivered to the agents **by file path**, never as
inline strings:

- **Claude Code** accepts `--append-system-prompt-file <path>` in
  `claude_args`. The CLI reads the file itself.
- **Codex action** accepts a `prompt-file` input that takes a
  repository-relative path.

This sidesteps the fragile pattern of reading a file into a bash
variable and passing it through `$GITHUB_OUTPUT`, which fails when
prompt content happens to match the heredoc delimiter or contains
sequences the GitHub runner interprets specially.

## Codex sandbox configuration

The Codex action has two orthogonal knobs that are easy to confuse:

- **`safety-strategy`** controls privilege: `drop-sudo` (default,
  recommended), `unprivileged-user`, `read-only`, `unsafe`. This is
  about sudo and the user identity Codex runs as — not about
  filesystem writability.
- **`sandbox`** controls filesystem access: `workspace-write`
  (default), `read-only`, `danger-full-access`.

For a review workflow that runs `forge build` and `forge test`, you
need `sandbox: workspace-write` because those commands write
`cache/` and `out/` in the workspace. If you mistakenly set
`safety-strategy: read-only` thinking it's the sandbox knob (easy
mistake — the name is misleading), Codex can't run the build and
all findings will be annotated with "build could not run due to
read-only filesystem".

## Secrets

`CLAUDE_API_KEY` and `OPENAI_API_KEY` are 0xPolygon org-level
secrets with visibility set to all repositories. Consuming repos
pass them through with `secrets: inherit`. Rotate centrally.

## Cost expectations

Claude Opus 4.7: ~$0.10–0.20 per typical PR review.
GPT-5.3-Codex: ~$0.05–0.15 per typical PR review.

For ~100 PRs/month across all consuming repos, expect ~$20–35/month
combined API spend.

## What this is NOT

- Not a replacement for human security review or professional
  audits.
- Not a replacement for `forge fmt` in a separate CI
  job — those are deterministic and cheaper. Run them too.
- Not interactive on standalone issues — `@claude` only works in PR
  comments and review threads (intentional; standalone-issue
  conversations tend to be open-ended and burn API credits).
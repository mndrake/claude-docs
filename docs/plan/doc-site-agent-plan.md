# Plan: a self-maintaining Claude consulting doc site

Status: draft for review. Open questions are in the last section. Everything else
is written under the stated defaults so work can start as soon as you confirm or
override them.

## 1. What we are building

A documentation site for Claude consulting content with four content types:

| Type | Purpose | Example |
|---|---|---|
| Cookbook | Short, tested "how to do X with Claude" recipes | "Stream tool calls with the Agent SDK in Python" |
| Runbook | Step-by-step operational procedures with checks and rollback | "Rotate an Anthropic API key across Bedrock and direct API" |
| Playbook | Opinionated engagement guidance for consultants | "First 30 days of a Claude Code rollout" |
| Plugin | A Claude Code plugin that ships the playbooks as skills and commands | `/runbook rotate-key` |

Two agents keep it current:

1. **Issue agent.** Reads GitHub issues (bug reports, gaps, requests), plans, implements the change, and opens a pull request. Humans merge.
2. **Vendor watch agent.** Polls vendor documentation on a schedule, detects meaningful changes, maps them to the site pages that depend on those sources, and files issues for the issue agent to act on.

The design principle: **the repository is the agent's memory and the pull request is the only write path to the live site.** No agent ever pushes to `main`.

## 2. Architecture

```
                 ┌──────────────────────────────┐
  vendor docs ──▶│ vendor-watch (scheduled)      │──▶ issues (label: vendor-change)
  llms.txt,      │ fetch → normalize → diff →    │          │
  changelogs     │ summarize → map to pages      │          ▼
                 └──────────────────────────────┘   ┌──────────────────────────┐
  humans ──────▶ issues (bug / gap / request) ────▶│ triage (on issue open)    │
                                                    │ classify, label, plan     │
                                                    │ comment, ask questions    │
                                                    └─────────────┬────────────┘
                                                                  │ label: approved
                                                                  ▼
                                                    ┌──────────────────────────┐
                                                    │ implement (on label)      │
                                                    │ branch → edit → lint →    │
                                                    │ build → PR "Closes #N"    │
                                                    └─────────────┬────────────┘
                                                                  ▼
                              ┌──────────────┐      ┌──────────────────────────┐
  GitHub Pages ◀── deploy ◀───│ main         │◀─────│ PR: CI gates + review     │
                              └──────────────┘      │ agent + human merge       │
                                                    └──────────────────────────┘
```

### 2.1 Runtime choice

Default: **GitHub Actions running Claude Code** (`anthropics/claude-code-action@v1`),
one workflow per job. The action runs in two modes: interactive, triggered by an
`@claude` mention on an issue or PR, and automation, driven by a `prompt` input on any
GitHub event including `schedule`. Auth is an `ANTHROPIC_API_KEY` secret, or a
`CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token` if you would rather bill a
Claude subscription than the API. Reference: https://code.claude.com/docs/en/github-actions

Why this over a standalone Agent SDK service:

- Zero infrastructure. Triggers, secrets, logs, and permissions already exist in GitHub.
- Every agent action is a commit, comment, issue, or PR, so it is auditable and reversible.
- Cost is bounded per run with `max_turns` and job timeouts.

When to move a job to the **Claude Agent SDK** (`claude-agent-sdk` on PyPI,
`@anthropic-ai/claude-agent-sdk` on npm) later: if a job
needs long-lived state, more than roughly 30 minutes of work, custom tools that are
awkward as MCP servers, or if Actions minutes become the bottleneck. The repo layout
below keeps prompts and scripts separate from the workflow glue so this swap is cheap.

Claude Code on the web Routines (https://claude.ai/code/routines, research preview)
are a third option. They run a fresh cloud session on a cron, on an HTTP call, or on
a GitHub pull request or release event, with the repo cloned and no permission
prompts. They are simpler to set up than Actions but the prompt lives outside the
repo, so they are harder to version and audit. Use them as the fallback, or for the
vendor watch job if the Actions 60-day rule below becomes a nuisance.

One Actions gotcha to remember if the repo is ever made public: GitHub disables
scheduled workflows on public repositories after 60 days without activity. It does
not apply to this private repo. The snapshot commits in §4.2 would keep it active anyway.

### 2.2 Repository layout

```
claude-docs/
├── docs/                     # site content (Markdown, one folder per content type)
│   ├── cookbooks/
│   ├── runbooks/
│   ├── playbooks/
│   ├── plugin/               # user docs for the plugin
│   └── plan/                 # this document and future ADRs
├── plugin/                   # the Claude Code plugin (see §5)
│   ├── .claude-plugin/plugin.json
│   ├── skills/
│   ├── commands/
│   ├── agents/
│   └── hooks/
├── .claude-plugin/marketplace.json   # lets users `claude plugin marketplace add mndrake/claude-docs`
├── agent/                    # everything the agents use, versioned like code
│   ├── prompts/              # triage.md, implement.md, vendor-watch.md, review.md
│   ├── sources.yaml          # vendor source registry (see §4)
│   ├── snapshots/            # normalized text of each watched source, last seen
│   └── scripts/              # fetch_sources.py, diff_sources.py, coverage.py, check_frontmatter.py
├── .github/
│   ├── workflows/            # triage, implement, vendor-watch, pr-review, quality, deploy
│   ├── ISSUE_TEMPLATE/       # bug, gap, request, vendor-change (agent-filed)
│   └── labels.yml
├── CLAUDE.md                 # house rules every agent run loads
├── STYLE.md                  # writing style guide, referenced from CLAUDE.md
└── mkdocs.yml                # or astro.config.mjs, see question 2
```

### 2.3 Page metadata: the contract between the two agents

Every page carries frontmatter. This is what makes "vendor changed X, which pages care?"
answerable without reading the whole site.

```yaml
---
title: Rotate an Anthropic API key
type: runbook            # cookbook | runbook | playbook | reference
status: current          # draft | current | stale | deprecated
last_verified: 2026-10-03
sources:                 # ids from agent/sources.yaml
  - anthropic-api-keys
  - bedrock-claude-auth
owners: [mndrake]
---
```

A script (`check_frontmatter.py`) fails CI when a page is missing fields, cites a
source id that is not in the registry, or has `status: current` with a
`last_verified` older than the newest change to any of its sources.

## 3. Issue agent

### 3.1 Triage (on `issues: opened, edited`)

1. Load `CLAUDE.md`, the issue, and the site index (titles plus frontmatter).
2. Classify: `bug`, `gap`, `request`, `vendor-change`, or `question`. Apply labels.
3. Find affected pages by searching the site. List them in a plan comment:
   what will change, which pages, estimated size (`size:xs` to `size:l`).
4. If the issue is ambiguous, post up to three clarifying questions and apply
   `needs-info` instead of a plan. Re-plan when the issue is edited or answered.
5. Never implements in this step. Output is labels plus one comment.

### 3.2 Implement (on label `approved`, or on comment `@claude implement`)

1. Create branch `agent/issue-<n>-<slug>` from `main`.
2. Make the change following `STYLE.md`. Each factual claim about vendor behavior
   links to a source in `sources.yaml` and updates the page's `last_verified`.
3. Run the fast checks locally: markdownlint, vale, frontmatter check, site build,
   link check on changed pages.
4. Open a PR titled from the issue, body: summary, pages touched, "Closes #n",
   and a checklist of what a human should verify.
5. Post a one-line comment on the issue linking the PR.

Autonomy default: `size:xs` and `size:s` issues from repository collaborators skip the
`approved` gate and go straight to a PR. Everything else waits for the label. See
question 4.

### 3.3 PR review (on `pull_request`)

A review agent checks the diff against `STYLE.md`, verifies that every new claim
has a source, runs the same checks as CI, and leaves a review. It does not merge.
Branch protection requires CI green and one human approval.

### 3.4 Guardrails

- Tool allowlist per workflow: triage gets read plus GitHub comment and label tools only;
  implement gets file edits, the lint commands, and git push to `agent/*` branches.
- `max_turns` and a job timeout on every workflow. Budget alert on the Anthropic console.
- One PR per issue. If a PR already exists for the issue, the agent updates it instead of opening another.
- Concurrency group per issue so two edits of the same issue cannot race.
- Agents never touch `.github/workflows/`, `CLAUDE.md`, or `agent/prompts/`.
  Changes there are human-only and enforced with CODEOWNERS.

## 4. Vendor watch agent

### 4.1 Source registry (`agent/sources.yaml`)

```yaml
sources:
  - id: claude-code-changelog
    url: https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md
    kind: changelog           # changelog | page | llms-txt | sitemap | release-feed
    cadence: daily
    severity_hint: high       # changelog entries usually need action
  - id: claude-code-releases
    url: https://github.com/anthropics/claude-code/releases.atom
    kind: release-feed
    cadence: daily
  - id: claude-code-docs-index
    url: https://code.claude.com/docs/llms.txt
    kind: llms-txt            # index of doc pages; a changed index triggers fetch of the listed pages
    cadence: daily
  - id: claude-platform-docs-index
    url: https://platform.claude.com/docs/llms.txt
    kind: llms-txt
    cadence: daily
  - id: claude-code-release-notes
    url: https://docs.anthropic.com/en/release-notes/claude-code
    kind: page
    selector: main
    cadence: daily
  - id: bedrock-claude-models
    url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    kind: page
    selector: main
    cadence: weekly
```

Scope default: Anthropic sources first. The verified machine-readable entry points
are the Claude Code CHANGELOG and releases Atom feed, the two `llms.txt` indexes
above, and the release-notes HTML page. There is no documented sitemap or docs RSS
feed, so the `llms-txt` kind works in two stages: diff the index, then fetch and
diff each page the index lists that is also cited by at least one site page.
Deprecation and model pages are added as `page` sources once the first pass is quiet. Cloud vendor pages are added per
question 3. Each source lists nothing about which pages depend on it; that
mapping is the reverse index built from page frontmatter, so adding a page
automatically subscribes it.

### 4.2 Pipeline (scheduled daily, plus manual dispatch)

1. **Fetch.** Pull each source. HTML is reduced to readable text with the
   `selector`; Markdown is kept as is. Normalize whitespace and strip timestamps
   and nonces so cosmetic churn does not diff.
2. **Diff.** Compare against `agent/snapshots/<id>.txt`. No change means stop for that source.
3. **Summarize.** Claude receives the diff and returns structured JSON:
   changed topics, kind (`new-feature`, `behavior-change`, `deprecation`,
   `doc-only`), severity, and a one-paragraph summary with quotes.
4. **Map.** `coverage.py` joins changed source ids to pages via frontmatter, and
   also asks Claude whether a `new-feature` change has no page at all (a gap).
5. **File.** One issue per meaningful change, templated, labeled `vendor-change`
   plus severity, listing affected pages and the gap verdict. Deduplicated by a
   fingerprint stored in the issue body so re-runs update rather than duplicate.
   `doc-only` changes below a severity threshold are batched into one weekly digest issue.
6. **Commit** the new snapshots to `main` via a PR that auto-merges when CI is green.
   Snapshots are the only agent writes that bypass human review, and they
   touch nothing under `docs/`.

### 4.3 Closing the loop

`vendor-change` issues flow into triage like any other issue. High-severity
ones are auto-approved for `size:xs`/`size:s` changes (for example, bump a
version number, mark a flag deprecated). Larger ones wait for a human.

A weekly `stale-sweep` job lists pages whose `last_verified` predates a source
change and files one digest issue, so drift is visible even when the summarizer
missed something.

## 5. The plugin

The plugin is the distribution channel for the playbooks and runbooks, and a
dogfooding target: the agents that maintain the site use the same skills.

- `skills/`: one skill per playbook, generated from `docs/playbooks/*.md` by a
  build script so docs and skills cannot drift.
- `commands/`: `/runbook <name>` renders a runbook as a checklist and walks through it.
- `agents/`: a `doc-reviewer` subagent that applies `STYLE.md`, reused by the PR review workflow.
- `hooks/`: optional, for example a `SessionStart` hook that prints the plugin version and the docs URL.
- `.claude-plugin/marketplace.json` at repo root (`name`, `owner`, `plugins[]`) so install is
  `claude plugin marketplace add mndrake/claude-docs` then `claude plugin install <name>@<marketplace>`.
- Evals: `claude plugin eval` suites for the top skills run in CI on changes under `plugin/`.
  Graders can be regex, tool-use, file-exists, or LLM judge, and `--ablation with-without`
  measures whether a skill actually helps versus no skill, which is the test that matters
  for a playbook.
- Versioning: semver in `plugin.json`, a GitHub Release per version, release notes
  drafted by the implement agent from merged PR titles.

## 6. Quality gates (CI on every PR)

| Check | Tool | Fails when |
|---|---|---|
| Markdown lint | markdownlint-cli2 | formatting rules broken |
| Prose lint | vale with a house style | banned words, passive voice over limit, missing admonitions |
| Frontmatter | `check_frontmatter.py` | missing fields, unknown source id, stale `current` page |
| Links | lychee | broken internal or external link (external cached, retried) |
| Build | mkdocs build --strict (or astro build) | warnings, missing nav entries |
| Plugin | `claude plugin validate` and eval suite | manifest invalid, eval regressions |
| Agent output | PR body contains `Closes #n` and a verification checklist | agent-opened PR missing either |

## 6a. Cost model: subscription or API

Every runtime in this plan can bill either way. The choice is made per workflow by
which secret you hand it, so the two can be mixed.

| Runtime | Subscription (Pro, Max, Team, Enterprise) | API (Claude Console) |
|---|---|---|
| GitHub Action | `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`. Docs: "If you authenticate with an OAuth token, runs use your Claude subscription instead of API billing." | `ANTHROPIC_API_KEY`, billed per token at list rates |
| Routines (claude.ai/code) | Only option. "Routines draw down subscription usage the same way interactive sessions do." | Not available |
| Agent SDK service | OAuth supported | API key, per token |

What that means in practice:

- **Subscription path.** Agent runs come out of the same 5-hour and weekly usage
  windows as your interactive Claude Code and chat use. The OAuth token is tied to
  the one person who ran `claude setup-token`, so a shared repo should use an API key
  or a Console service account via OIDC instead. When the window is exhausted, runs
  fail unless usage credits are turned on, in which case they continue at metered
  API rates. A cap on daily scheduled jobs is the main lever.
- **API path.** Pay per token, no interaction with your personal quota, hard caps via
  Console workspace spend limits. Predictable and auditable, and the right default
  for anything scheduled.
- **GitHub Actions minutes** are charged separately by GitHub on either path. This is a
  private repo, so runs draw on the plan's free monthly allowance and then bill per
  minute. Agent runs are long-tailed, so set `timeout-minutes` on every job, and
  expect Actions minutes to be a visible line item next to token spend.

Rough API-path estimate for the default design, assuming Opus 5.5 for implement and
review and Sonnet 5.5 for triage and summaries. These are order-of-magnitude figures,
not quotes; measure in M1 and revise.

| Job | Frequency | Est. per run | Est. per month |
|---|---|---|---|
| Triage | per issue, say 20/month | $0.10 to $0.50 | $2 to $10 |
| Implement | per approved issue, say 10/month | $1 to $5 | $10 to $50 |
| PR review | per PR, say 15/month | $0.30 to $1.50 | $5 to $25 |
| Vendor watch | daily, scripts do the fetching | $0.05 to $0.50 on change days, near zero otherwise | $2 to $10 |
| Stale sweep | weekly | $0.20 to $1 | $1 to $4 |
| **Total** | | | **roughly $20 to $100** |

Recommended split: run the vendor watch, stale sweep and PR review on an API key so
they never touch your subscription and have a hard spend limit. Run triage and
implement on whichever you prefer. If you already pay for Max, point them at the
subscription first and watch the `/usage` breakdown; move them to the API key if they
start crowding out your interactive work.

Sources: https://code.claude.com/docs/en/github-actions (Manage costs),
https://code.claude.com/docs/en/routines (Usage and limits),
https://code.claude.com/docs/en/costs.

## 7. Milestones

Durations assume one person part time with the agents doing most typing.
Each milestone ends with something usable.

### M0. Foundations (week 1)
- Decide the open questions in §8.
- Static site generator, `mkdocs.yml` or Astro config, deploy workflow to private
  hosting (GitHub Pages with access control on Enterprise Cloud, else Cloudflare
  Pages behind Access or Vercel with SSO).
- `CLAUDE.md`, `STYLE.md`, frontmatter schema, `check_frontmatter.py`.
- Labels, issue templates, CODEOWNERS, branch protection on `main`.
- Seed content: three to five real pages, at least one of each type, so the
  agents have patterns to imitate.
- Exit: site live at a URL, a hand-written PR passes all CI gates.

### M1. Issue agent v1 (week 2)
- `triage.yml` and `implement.yml` with the Claude Code action, prompts under `agent/prompts/`.
- Tool allowlists, `max_turns`, concurrency groups, one-PR-per-issue logic.
- Exit: file an issue "typo on page X", get a mergeable PR within minutes with no human steps
  other than merge. File a vague issue, get clarifying questions.

### M2. Quality gates and review agent (week 3)
- markdownlint, vale, lychee, build, all wired into CI and into the implement agent's local checks.
- `pr-review.yml` review agent using the shared `doc-reviewer` definition.
- Exit: an agent PR with an unsourced claim is blocked by CI and flagged by review.

### M3. Vendor watch v1 (weeks 4 to 5)
- `sources.yaml` with Anthropic sources, fetch and normalize scripts, snapshots committed.
- Diff, summarize with structured output, reverse-index mapping, issue filing with dedupe.
- Exit: edit a snapshot by hand to simulate a change, run the workflow, get one
  correctly mapped `vendor-change` issue. Then run live for a week and tune noise.

### M4. Plugin and marketplace (week 6)
- `plugin/` skeleton, skills generated from playbooks, `/runbook` command, marketplace manifest.
- Eval suite for the top three skills, `claude plugin validate` in CI.
- First tagged release and an install page on the site.
- Exit: fresh machine installs the plugin from the marketplace and runs a runbook.

### M5. Close the loop (week 7)
- Auto-approve small `vendor-change` issues, `stale-sweep` weekly digest.
- Metrics committed as a JSON file and rendered on a site page: issue-to-PR time,
  PRs merged without edits, vendor changes detected versus manually noticed, cost per run.
- Exit: a real upstream change becomes a merged PR with at most one human click (merge).

### M6. Hardening and expansion (ongoing)
- Add cloud vendor sources, more plugins or skills, a site feedback widget that files issues.
- Revisit runtime: move long jobs to the Agent SDK if Actions limits bite.
- Quarterly prompt review using the metrics page.

## 8. Sources verified for this plan

- GitHub Action, modes and auth: https://code.claude.com/docs/en/github-actions
- Agent SDK: https://code.claude.com/docs/en/agent-sdk/overview
- Plugin manifest and marketplace: https://code.claude.com/docs/en/plugins/manifest-reference,
  https://code.claude.com/docs/en/plugins/marketplace-reference
- Plugin evals: https://code.claude.com/docs/en/plugin-evals
- Routines: https://code.claude.com/docs/en/routines
- Doc indexes: https://code.claude.com/docs/llms.txt, https://platform.claude.com/docs/llms.txt
- Claude Code changelog and feed: https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md,
  https://github.com/anthropics/claude-code/releases.atom

## 9. Open questions

Each one changes the plan materially. Defaults are what the plan above assumes.

1. **Audience and visibility.** Answered: internal, private repository. Consequences
   folded into the plan: private GitHub Pages (needs GitHub Enterprise Cloud; otherwise
   Cloudflare Access or Vercel with SSO in front of the static build), issues come only
   from collaborators so the approval gate can be looser, Actions minutes are billed
   after the free allowance, and the 60-day scheduled-workflow shutoff does not apply.
   Still to decide: which hosting option, and whether external clients ever get access.
2. **Site generator.** MkDocs Material (Python, pure Markdown, simplest for agents to edit)
   or Astro Starlight (MDX, more design freedom)? Default: MkDocs Material.
3. **Vendor scope.** Anthropic docs only to start, or also AWS Bedrock, GCP Vertex,
   Azure AI Foundry, Databricks, others? Default: Anthropic only in M3, others in M6.
   Each extra vendor adds roughly a day of selector tuning and noise filtering.
4. **Autonomy level.** Should small changes skip the `approved` label and go straight to PR?
   Should any PRs auto-merge? Default: small issues from collaborators skip approval;
   nothing auto-merges except snapshot commits.
5. **Runtime and budget.** See §6a. Which plan do you have, and do you want agent runs on
   your subscription, on an API key, or split as recommended? Is there a monthly
   ceiling? That sets `max_turns`, polling cadence, and whether review runs on every PR.
6. **Who files issues.** Only you, also clients, also the public? Default: collaborators
   plus the vendor watch agent. Public filing needs spam controls and a stricter approval gate.
7. **Plugin shape.** One plugin bundling all playbooks, or several focused plugins
   (for example `claude-ops`, `claude-consulting`)? Default: one plugin until it has
   more than about ten skills.
8. **Scripting language.** Python or TypeScript for `agent/scripts/`? Default: Python,
   matching MkDocs. If Starlight wins question 2, TypeScript.
9. **Existing material.** Do you have notes, gists, or client docs to import as seed content?
   This decides whether M0 is writing or migrating.
10. **Models.** Default: the most capable current model for implement and vendor summarize,
    a cheaper model for triage and link-level checks. Any constraint on which models can be used?

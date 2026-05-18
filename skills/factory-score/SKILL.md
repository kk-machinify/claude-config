---
name: factory-score
description: Score a repository against the "Software Factory" maturity rubric — measures how ready a repo is for AI-driven development. Produces both a 0–18 Score and a gated 0–3 Level (the two can disagree; level wins for agentic-readiness decisions). Use when triaging repos, prioritizing AI-readiness investment, or checking whether a codebase is amenable to autonomous agent workflows.
version: 0.3.0-wip
---

> ⚠️ **WORK IN PROGRESS — NOT YET VETTED**
>
> This skill is being iterated on actively. The rubric has gone through three revisions:
> - **v1** counted *presence* of artifacts (over-scored large monorepos)
> - **v2** added coverage adjustment for scale + code-health + CI-speed modifiers
> - **v3** (current) added **gated Levels** because the additive score doesn't capture agentic-readiness on its own — a repo can score 16/18 and still be Level 1 if the points are distributed wrong
>
> Known limitations: infrastructure repos (Terraform, build-images), generated code, editor extensions, training content, and CI-only tooling all fit awkwardly into this rubric and may need separate variants. Branch-protection detection assumes GitHub rulesets and may underreport on classic protections. Some gates (especially G11, the auto-feedback loop) are hard to detect without runtime observation. Treat output as a starting point for conversation, not a final assessment.

---

# Factory Score — Software Factory Rubric

A lightweight, mechanically checkable scorecard inspired by the [Building the Software Factory](https://www.notion.so/machinify/Building-the-Software-Factory-31b5356b3971817e8ceac1eaf6a9253a) framing.

Scores a single repo on two dimensions:

- **Score** — 0–18 across 6 categories (0–3 each), additive. Useful for "how much investment does this repo need."
- **Level** — 0/1/2/3, gated by specific capabilities. Answers "is this repo agentic-ready."

When Score and Level disagree, **Level wins** for any agentic-readiness decision.

Every signal is a file existence check, file-size/age check, CI config key, grep pattern, or a `git log` / `wc -l` calculation — no human judgement required.

## Invocation

```
/factory-score                                    # current dir
/factory-score /path/to/local/repo                # local repo
/factory-score https://github.com/org/repo        # GitHub URL (auto-clones)
```

Output saved to `~/factory-score/<repo>-YYYY-MM-DD.md`.

---

## Step 0: Repo Scale preamble (compute once)

| Metric | How to compute | Use |
|--------|---------------|-----|
| `loc_total` | `cloc --json . \| jq .SUM.code` (or fallback: find pipe through wc -l for source extensions) | Scale tier |
| `file_count` | `git ls-files \| wc -l` | Scale tier |
| `repo_size_mb` | `du -sm . \| cut -f1` (exclude .git) | Scale tier |
| `module_dirs` | `find . -maxdepth 1 -type d -not -name '.*' -not -name 'node_modules' -not -name 'target'` | Coverage denominators |
| `median_file_size` | find pipe \| `wc -l` \| `sort -n` \| middle value | Code health |
| `pct_files_over_500` | `awk '{c++; if($1>500)l++} END{print l/c*100}'` over the wc -l output | Code health |
| `top_5_hotspots` | `git log --since='1 year ago' --name-only --pretty=format: \| sort \| uniq -c \| sort -rn \| head -5` | Code health |
| `ci_duration_min` | `gh run list --workflow=<main> --status=success -L 5 --json startedAt,updatedAt --jq '...'` | CI speed signal |

### Scale tier (used by Context + Orchestration scoring)

| Tier | Criteria |
|------|----------|
| **Small** | `file_count < 500` AND `repo_size_mb < 100` |
| **Medium** | `file_count` 500–2,000 OR `repo_size_mb` 100–500 |
| **Large** | `file_count > 2,000` OR `repo_size_mb > 500` |

---

## Score: 6 categories × 0–3

### 1. Context Engineering

| Score | Signal |
|-------|--------|
| 0 | No `CLAUDE.md` / `AGENTS.md` / `.cursor/rules` / `.cursorrules` / `.github/copilot-instructions.md` |
| 1 | One exists, ≥50 lines, mtime <180 days |
| 2 | Above + at least one of: `docs/adr/`, `docs/conventions.md`, `ARCHITECTURE.md` + machine-readable API spec (`openapi.yaml`, `*.proto`, GraphQL schema) checked in |
| 3 | Above + custom lint rules or codegen schemas enforce conventions + **scale-adjusted coverage gate** (below) |

**Scale-adjusted gate for Score 3:**
- Small: every top-level source dir has `README.md` or CLAUDE.md
- Medium: ≥70% of top-level source dirs have per-area context, OR root CLAUDE.md names them
- Large: ≥80% of top-level source dirs have per-module context, OR structured `.claude/docs/` with ≥5 topic files AND a clear mapping doc

### 2. Guardrails

| Score | Signal |
|-------|--------|
| 0 | No CI workflow file; no typechecker config |
| 1 | CI exists; typechecker config present (`tsconfig.json`, `mypy.ini`/`pyproject.toml`, `go.mod`, `Cargo.toml`) |
| 2 | + linter referenced in CI + `.pre-commit-config.yaml` or husky/lefthook + branch protection + secret scanning |
| 3 | + architectural fitness functions (import-linter, dependency-cruiser, ArchUnit) + SAST in CI (CodeQL, Semgrep) + mutation testing config + perf regression gate |

### 3. Feedback Loops

| Score | Signal |
|-------|--------|
| 0 | No test directory; or test LOC < 10% of source LOC |
| 1 | Tests exist, run on PR |
| 2 | + integration or e2e test dir + visual regression for UI + `.claude/hooks/` running tests after edits |
| 3 | + adversarial reviewer (`.claude/agents/` review subagent or `claude-pr-review.yml`) + mutation testing in CI **AND CI speed gate** (below) |

**CI speed gate:**
- `ci_duration_min < 5` required for Score 3
- `ci_duration_min < 15` required for Score 2 (else cap at 1)
- < 2 min = bonus signal (the Software Factory threshold for agent productivity)

### 4. Agent Orchestration

| Score | Signal |
|-------|--------|
| 0 | No `.claude/`, `.cursor/`, or `.github/copilot-*` |
| 1 | Flat `CLAUDE.md` only (no skills, no subagents) |
| 2 | `.claude/skills/` or `.claude/commands/` populated + MCP config + at least one subagent |
| 3 | + planning/dispatch skill (controller pattern visible) + documented agent-to-agent handoffs + **scale-adjusted skill density** (below) |

**Scale-adjusted skill density for Score 3:**
- Small: ≥3 skills or commands
- Medium: ≥1 skill per ~5 top-level source dirs
- Large: ≥1 skill per ~10 top-level source dirs AND skills mention major module names

### 5. Knowledge Persistence

| Score | Signal |
|-------|--------|
| 0 | No changelog, lessons, or decision log |
| 1 | `CHANGELOG.md` or session-notes file |
| 2 | + `docs/lessons/`, `docs/decisions/`, or per-area knowledge + postmortems in repo |
| 3 | Self-curating: a scheduled job updates the knowledge base from merged PRs / incidents |

### 6. CI / Workflow Integration

| Score | Signal |
|-------|--------|
| 0 | No CI |
| 1 | CI runs tests on PR |
| 2 | + agent-runnable entrypoints (`make ci`, `make test`, `make lint`, or `package.json` scripts, or `justfile`) + bot/PR integrations (Renovate, Dependabot, custom PR bots) |
| 3 | + cron-triggered or event-triggered cloud agents (GitHub Action invoking Claude/Anthropic API/Bedrock) + ticket→PR pipeline + scheduled entropy/sweeper agent |

---

## Code Health modifier (apply after scoring)

| Condition | Modifier |
|-----------|----------|
| `median_file_size > 500` lines | Cap Context + Feedback at 2 |
| `pct_files_over_500 > 20%` | Cap Context + Feedback at 2 |
| Top-5 hotspots include files with >100 commits in last year | Subtract 1 from raw total |
| `repo_size_mb > 500` AND no per-module context coverage ≥50% | Cap Context at 1 |

**Reasoning:** AI-generated code on unhealthy code raises defect risk by ~60% (CodeScene research). High-presence repos shouldn't receive Level-2 confidence if the substrate is unhealthy.

---

## Level: gated by specific capabilities

A repo's Level is **not** its score band. Each level has explicit gate capabilities. If any gate is missing, the repo is below that level — regardless of total score.

### Level 0 → Level 1 gate (Individual Productivity)

Single gate:
- **G0:** ≥1 machine-readable context file (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.cursor/rules/`, or `.github/copilot-instructions.md`)

Below this, agents can't even start. *Markdown docs in `docs/` don't count* — the gate is specifically about agent-loadable context.

### Level 1 → Level 2 gate (Managed Automation)

All 6 must hold:

| Gate | Check | Why it's a gate |
|------|-------|-----------------|
| **G1: Context maps to modules** | Per-area README/CLAUDE/AGENTS in ≥50% of top-level source dirs, OR `.claude/docs/` with ≥3 topic files | Without it, agent gets lost in any non-trivial repo |
| **G2: Type checker + linter in CI** | Both present AND referenced in a workflow | Agent gets fast structural feedback |
| **G3: Branch protection** | Active ruleset on default branch (require PR + status checks) | Agent can't bypass verification |
| **G4: Behavior-verifying tests on PR** | Tests directory exists AND CI runs them. UI repos: include e2e or visual-regression | "Did it work?" floor of agent self-check |
| **G5: Specialist skills or subagents** | `.claude/skills/` ≥1 skill, OR `.claude/agents/` ≥1 subagent, OR `.claude/commands/` ≥3 commands | Agent can dispatch to focused context |
| **G6: Adversarial reviewer** | `.claude/agents/<review-or-verify>.md`, OR `claude-pr-review.yml` (any workflow invoking Claude API or Bedrock-Claude on PRs) | Anthropic's #1 recommendation: "give Claude a way to verify its work" via a separate context |

### Level 2 → Level 3 gate (Autonomous Pipeline)

All 6 Level 2 gates PLUS all 5 below:

| Gate | Check | Why it's a gate |
|------|-------|-----------------|
| **G7: Event-triggered cloud Claude** | Workflow with `on: pull_request` invoking Claude/Anthropic API/Bedrock, with `pull-requests: write` permission | PR review without waiting on a human |
| **G8: Cron-triggered cloud agent** | Workflow with `schedule:` cron trigger that runs a Claude/automation job (sweeper, knowledge curator, type-audit) | Factory maintains itself |
| **G9: Quality verification beyond "tests pass"** | At least one of: mutation testing in CI, property-based tests for invariants, or merge-blocking Claude review (not advisory-only) | Catches AI's self-correction blind spot |
| **G10: Self-curating knowledge** | Scheduled job updates docs/lessons/knowledge automatically from PRs/incidents (not just deploys) | No human-bottlenecked knowledge debt |
| **G11: Feedback loop into agent execution** | Evidence of self-iteration: test fails → agent fixes → re-runs without human prompt | The agentic-pipeline gate — automation vs autonomy |

---

## Process

1. Resolve the target repo (current dir, local path, or GitHub URL → clone to `/tmp/factory-score-<repo>`)
2. Compute Repo Scale preamble (file count, size, modules, code health, CI duration)
3. Run "How to check" commands for each category in parallel
4. For each category: pick the highest level whose signals AND scale gate are satisfied — that's the Score
5. Apply Code Health modifier to total
6. Evaluate gates in order: L0→L1 → L1→L2 → L2→L3. Stop at first failed gate
7. Output **Score · Level** with explicit list of which gates passed/failed
8. Save the scorecard

## Output format

Save to `~/factory-score/<repo>-YYYY-MM-DD.md`:

- **Header:** repo name, commit SHA, date, Repo Scale preamble
- **Score**: 6-category table (0–3 each, total /18)
- **Level**: gated assessment with explicit gate-pass/fail table
- **Why Level X (not X+1)**: the specific failing gate(s) — these are the agentic-readiness gaps
- **Highest-leverage next move**: sized to closing the next gate, not adding a category point
- **Code Health modifier applied**: ± n with reasoning
- **Notable observations**: anything surprising

For an at-a-glance summary in a database or table, use the compact format: `N/18, LX` (e.g., `15/18, L2`, `16/18, L1`).

Keep output to a single page where possible. This is fast triage, not a deep audit.

---

## Common patterns and disagreement cases

When Score and Level disagree, the most common reason is a missing **G6 (adversarial reviewer)**: a repo with great docs, great CI, and great tests but no `claude-pr-review.yml` workflow caps at Level 1. The fix is one workflow file.

The second-most-common is **G5 (specialist skills)**: high-doc, high-test repos with only a flat `CLAUDE.md` and no `.claude/skills/` directory. The fix is populating skills mapping to the codebase's major modules.

For monorepos > 500MB / > 2,000 files, even a substantive root CLAUDE.md is often insufficient; per-module context becomes essential and the scale-adjusted gates reflect this.

---

## References

- Anthropic 2026 Agentic Coding Trends Report — context as infrastructure
- Stack Overflow 2026: bugs and incidents with AI coding agents (1.7× more bugs, 75% more logic errors)
- CodeScene — code health predicts AI effectiveness (defect risk +60% on unhealthy code)
- GitClear — AI doubles code duplication, halves refactoring across 211M lines
- Anthropic Code Review pattern — multi-agent cross-verification (94% detection vs 45% LLM-only)

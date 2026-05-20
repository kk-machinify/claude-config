---
name: factory-score
description: Score a repository against the "Software Factory" maturity rubric — measures how ready a repo is for AI-driven development. Produces both a 0–18 Score (with half-points) and a gated 0–3 Level (the two can disagree; level wins for agentic-readiness decisions). Use when triaging repos, prioritizing AI-readiness investment, or checking whether a codebase is amenable to autonomous agent workflows.
version: 0.7.0-wip
---

> ⚠️ **WORK IN PROGRESS — NOT YET VETTED**
>
> This skill is being iterated on actively. Rubric history:
> - **v1** counted *presence* of artifacts (over-scored large monorepos)
> - **v2** added coverage adjustment for scale + code-health + CI-speed modifiers
> - **v3** added **gated Levels** because the additive score doesn't capture agentic-readiness on its own — a repo can score 16/18 and still be Level 1 if the points are distributed wrong
> - **v4** added **half-point scores**, **parallel-subagent dispatch** for assessment, and a **formalized effort tiebreaker** for the recommended next move
> - **v5** added a **read-only repo state check** that aborts if the repo isn't on the default branch with a clean working tree
> - **v6** added **ADR / decision-log quality checks** — counting `docs/adr/` files isn't enough; v6 verifies structure, code-anchoring, cross-linking, and (critically) that the ADRs aren't byte-identical clones from another repo. Caught one real case (s3-castore had 28 ADRs that were verbatim copies of Pika's)
> - **v7** (current) tightened **G9's "merge-blocking Claude review"** check — a `claude-pr-review.yml` that just posts comments is **advisory**, not blocking. v7 requires both (a) the workflow exits non-zero when Claude flags blocking concerns AND (b) the workflow's status check is required by branch protection. Inspired by [StrongDM's framework](https://www.strongdm.com/blog/the-strongdm-software-factory-building-software-with-ai) where "humans don't review code" is the endpoint — that bar requires actually-gating reviewers, not advisory ones. Closed-loop execution (propose → fix → re-assess) is intentionally NOT included yet — this skill assesses only.
>
> Known limitations: infrastructure repos (Terraform, build-images), generated code, editor extensions, training content, and CI-only tooling all fit awkwardly into this rubric and may need separate variants. Branch-protection detection assumes GitHub rulesets and may underreport on classic protections. Some gates (especially G11, the auto-feedback loop) are hard to detect without runtime observation. Treat output as a starting point for conversation, not a final assessment.

---

# Factory Score — Software Factory Rubric

A lightweight, mechanically checkable scorecard inspired by the [Building the Software Factory](https://www.notion.so/machinify/Building-the-Software-Factory-31b5356b3971817e8ceac1eaf6a9253a) framing.

Scores a single repo on two dimensions:

- **Score** — 0–18 across 6 categories (0–3 each, **half-points allowed**), additive. Useful for "how much investment does this repo need."
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

## Step -1: Repo state check (run before everything else)

The skill assesses **the canonical state of the repo**, not whatever your local workspace happens to look like. Before scoring, verify the repo is on the default branch with a clean working tree. **Read-only — never modifies your state.**

If the repo was passed as a GitHub URL and freshly cloned to `/tmp/factory-score-<repo>`, this step is a no-op (the fresh clone is already on the default branch with no local changes).

For local paths (including current dir):

1. **Detect the default branch.** Try `gh api repos/<owner>/<repo> --jq .default_branch` if origin is a GitHub remote. Fall back to `git symbolic-ref refs/remotes/origin/HEAD --short` and strip the `origin/` prefix. If neither works, ask the user.
2. **Verify HEAD matches the default branch.** `git branch --show-current` must equal the default branch.
   - If not: **abort** with `"Assessment must run on the default branch (<X>). You're on <Y>. Switch branches or pass a GitHub URL instead."`
3. **Verify a clean working tree.** `git status --porcelain` must be empty.
   - If not: **abort** with `"Uncommitted changes detected. Commit or stash before scoring (or pass a GitHub URL to score the remote state)."`
4. **Warn if local is behind origin (don't abort).** `git fetch origin <default> 2>/dev/null && git rev-list --count HEAD..origin/<default>`.
   - If > 0: print `"⚠️ Local <default> is N commits behind origin. Score will reflect your local state, not the latest remote."`

Only if checks 2 and 3 pass, proceed to Step 0.

**Why read-only:** earlier drafts auto-stashed and switched branches. That works for repo-internal skills (where the workflow is "always run on develop"), but for a portable skill it's too invasive — the stash/pop cycle has too many failure modes (merge conflicts on pop, untracked files, detached HEAD, submodules). Refusing to run on a dirty/wrong-branch repo is louder, safer, and teaches the user to put state in order first.

**Override flag (optional):** if the user passes `--allow-dirty` or `--allow-non-default-branch`, skip the corresponding check. Useful for ad-hoc exploration but the resulting score should be marked `(unreliable: dirty tree)` or `(unreliable: feature branch)` in the output header.

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

## Process (v4): parallel-subagent dispatch

After Step 0, dispatch **6 parallel subagents** using the `Agent` tool with `subagent_type: "Explore"`. Each subagent scores one of the 6 categories independently.

Each subagent returns:

```
Score: <0, 0.5, 1, 1.5, 2, 2.5, or 3>
Evidence:
- <bullet per existing item found>
Gaps for next level:
- <bullet per missing item>
```

Why parallel: cheaper on main-context tokens, faster wall-clock, and each subagent focuses narrowly on one category's signals.

After collecting all 6 reports:
1. Sum into a raw 0–18 score
2. Apply Code Health modifier
3. Evaluate gates in order (L0→L1 → L1→L2 → L2→L3); stop at first failing gate to determine Level
4. Produce the scorecard + gap list + highest-leverage next move (ranked by effort tiebreaker)

---

## Score: 6 categories × 0–3 (half-points allowed)

### Half-point rule

A category scores X.5 when **most but not all** next-level criteria are met. Example: Context Engineering scores 2.5 if the repo has CLAUDE.md + ADRs + API spec (= Score 2) AND custom lint rules (= 1 of 2 Score-3 criteria) but is missing the per-module coverage gate.

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

## ADR / decision-log quality check (apply where ADRs are credited)

Counting files in `docs/adr/` or `docs/adrs/` is *not* enough — outdated, generic, or borrowed-from-another-repo ADRs are worse than no ADRs ("outdated context is worse than no context"). When ADRs are claimed for **Context Engineering Score ≥ 2** or **Knowledge Persistence Score ≥ 2**, run these mechanical quality checks. ADRs that fail multiple checks **don't count toward the score** for the category that's crediting them.

### Per-ADR structural checks

For each `*.md` file in `docs/adr/` or `docs/adrs/` (excluding `README.md` and `template.md`):

| Signal | How to check |
|--------|--------------|
| **Has structure** | Contains all of: `Status`, `Context` (or `Context and Problem Statement`), `Decision` (or `Decision Outcome`), `Consequences`. Grep for these section headers. |
| **Status is real** | `grep '^[*]*Status[*]*:' <adr>` returns a non-empty value (`Accepted`, `Superseded by X`, `Deprecated`, `Proposed`, etc.) |
| **Anchored to this repo's code** | `grep -E '<this-repo-name>/\|src/\|crates/\|apps/\|packages/\|\.cs\|\.rs\|\.py\|\.ts' <adr>` returns ≥1 hit referencing paths that exist in *this* repo |
| **Considers alternatives** | Contains `Considered Options`, `Alternatives`, or similar section listing ≥2 options |
| **Documents consequences both ways** | The Consequences section has both positive items AND tradeoffs/bad items (grep for `Bad:` `Tradeoff:` `Cons:` or similar) |

### Cross-cutting checks (across the ADR set)

| Signal | How to check |
|--------|--------------|
| **Status diversity** | Not 100% `Status: Accepted`. At least one ADR has `Superseded` / `Deprecated` / `Supersedes` / `Replaced by` — indicates the log is alive |
| **Cross-linking** | `grep -lE 'ADR [0-9]+\|0[0-9]{3}-' docs/adr*/*.md` returns ≥30% of ADRs |
| **Referenced from CLAUDE.md** | `grep -lE 'adr\|decision-log\|docs/adr' CLAUDE.md AGENTS.md .claude/docs/*.md` returns at least one match |
| **No cross-repo duplication** | For each ADR in this repo, check sibling repos in the same org for byte-identical files: `diff -r docs/adrs/ <sibling-repo>/docs/adrs/`. If ≥50% of ADRs are byte-identical to another repo's, **flag as borrowed** — the score for ADRs in this repo should be set to 0 unless the team has explicitly designated this as a shared-decisions pattern |

### Scoring impact

Compute `adr_quality_pct = mean across ADRs of (structural checks passed) / 5` and check the cross-cutting signals separately.

| Outcome | Effect on score |
|---------|-----------------|
| ≥80% structural + cross-cutting checks all pass | ADRs count fully toward Context Engineering and Knowledge Persistence |
| 50–79% structural OR 1 cross-cutting signal fails | ADRs count at half-weight — round category score down by 0.5 |
| <50% structural OR cross-repo duplication detected | ADRs **don't count** — score the category as if `docs/adr*/` didn't exist |

**Real case caught by v6 (2026-05-18):** s3-castore had 28 files in `docs/adrs/` byte-identical to Pika's. The structural checks would have passed (because the structure is Pika's — and Pika's is excellent) but the cross-repo-duplication check fails decisively. With v5, s3-castore scored Knowledge Persistence 2.5; with v6, it scores 1.5 (CHANGELOG only, ADRs excluded). The repo was subsequently marked `N/A: Superseded by Pika` in the org's tracking database.

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

A repo's Level is **not** its score band. Each level has explicit gate capabilities. If any gate is missing, the repo is below that level — regardless of total score. Half-point scores **do not change gate logic**; the gates are binary (met / not-met).

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
| **G9: Quality verification beyond "tests pass"** | At least one of: (a) mutation testing config with CI score gate (Stryker / `cargo-mutants` / mutmut / PIT), (b) property-based tests for invariants (`proptest` / `hypothesis` / `fast-check`), or (c) **merge-blocking** Claude review — see verification below | Catches AI's self-correction blind spot (64.5% per Anthropic research) |

**Verifying "merge-blocking" Claude review (v7).** A `claude-pr-review.yml` workflow that posts review comments is **advisory** — Claude's findings *suggest* but don't *gate* the merge. To count for G9, both of the following must hold:

1. **The workflow exits non-zero when Claude flags blocking concerns.** Check: grep the workflow for `exit 1`, `core.setFailed`, `:: error ::`, `--fail-on-error`, or equivalent gating logic in the Claude review step. A workflow that only calls `gh pr review --comment` is advisory.
2. **The workflow's status check is required by branch protection.** Check: `gh api repos/<owner>/<repo>/branches/<default>/protection/required_status_checks --jq '.contexts'` (or the rulesets equivalent) must include the workflow's check name. Without this, a passing/failing Claude review doesn't gate merge.

**Default behavior of every `claude-pr-review.yml` in the org as of 2026-05-18 is advisory** — they post comments but don't `exit 1` and aren't wired into branch protection. This is consistent with how cob-assistant, prompt-hub, drgrouper, document_processing_pipeline, mac-ui-presentation, and onlineDataAnalysis use the pattern. They satisfy **G6** (an adversarial reviewer exists — the L1→L2 gate) but **do not satisfy G9** (quality verification beyond tests — an L2→L3 gate).

The two gates were always distinct but I was implicitly conflating them. v7 makes the distinction explicit: G6 needs *any* adversarial reviewer (advisory is fine); G9 needs *merge-blocking* verification (advisory is not enough). For a repo to reach L3, the Claude reviewer has to actually be able to refuse to merge.
| **G10: Self-curating knowledge** | Scheduled job updates docs/lessons/knowledge automatically from PRs/incidents (not just deploys) | No human-bottlenecked knowledge debt |
| **G11: Feedback loop into agent execution** | Evidence of self-iteration: test fails → agent fixes → re-runs without human prompt | The agentic-pipeline gate — automation vs autonomy |

---

## Highest-leverage next move (with effort tiebreaker)

After the scorecard + gate analysis, recommend the **single highest-leverage next move** — the one that closes the next gate or completes a half-point.

### Ranking algorithm

1. **Category priority order** (matches the Software Factory phase progression):
   Context Engineering > Guardrails > Feedback Loops > Agent Orchestration > Knowledge Persistence > CI/Workflow Integration

2. **Within a category:** closing a failing **gate** (e.g., L1→L2 G6) takes priority over completing a half-point (e.g., 2 → 2.5 in score). Completing a whole-level advance (N → N+1) takes priority over completing a partial (N.5 → N+1).

3. **Effort tiebreaker** at the same leverage tier:
   - **Low** (<1 hour): config change, prompt edit, adding a single file (e.g., `claude-pr-review.yml`)
   - **Medium** (1–4 hours): new project/script, moderate code addition (e.g., add `.claude/skills/` with 3 skills)
   - **High** (>4 hours): cross-cutting infrastructure, multi-layer change (e.g., adopt mutation testing across a Rust workspace)

   Prefer **lower-effort** gaps at the same leverage tier.

### Output format for the recommendation

```
## Highest-Leverage Next Move

**Gap:** <which gate or half-point this closes>
**Category:** <name>
**Current:** <score & level> → **Target:** <new score & level>
**What to build:** <concrete files, config changes, or code needed>
**Effort:** <Low / Medium / High>
**Why this one:** <1 sentence: category priority + effort rationale>
```

---

## Process (full sequence)

1. **Resolve target repo** (current dir, local path, or GitHub URL → clone to `/tmp/factory-score-<repo>`)
2. **Step -1:** Repo state check — abort if not on default branch or working tree is dirty (skip if fresh clone)
3. **Step 0:** Compute Repo Scale preamble (file count, size, modules, code health, CI duration)
4. **Dispatch 6 parallel subagents** (one per category) with `Agent` tool, `subagent_type: "Explore"`. Each returns Score (0–3 with half-points) + Evidence + Gaps. Each subagent should exclude these paths: `node_modules`, `.git`, `target`, `dist`, `build`, `.next`, `.fastembed_cache`, `bin`, `obj`
5. **Collect all 6 reports** and sum to raw total
6. **Apply Code Health modifier**
7. **Evaluate gates** in order (L0→L1 → L1→L2 → L2→L3); stop at first failed gate to determine Level
8. **Rank gaps** by category priority + gate-over-half-point + effort tiebreaker
9. **Output** the scorecard + level analysis + highest-leverage next move

## Output format

Save to `~/factory-score/<repo>-YYYY-MM-DD.md`:

- **Header:** repo name, commit SHA, date, Repo Scale preamble
- **Score**: 6-category table (0–3 each with half-points, total /18)
- **Level**: gated assessment with explicit gate-pass/fail table
- **Why Level X (not X+1)**: the specific failing gate(s) — these are the agentic-readiness gaps
- **Highest-leverage next move**: ranked output (see format above)
- **Code Health modifier applied**: ± n with reasoning
- **Notable observations**: anything surprising

For an at-a-glance summary in a database or table, use the compact format: `N/18, LX` or `N.5/18, LX` (e.g., `15/18, L2`, `16.5/18, L1`).

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
- Provider Portal Core's `maturity-gap.md` — repo-tailored variant with closed-loop execution; inspired v4's parallel-subagent dispatch, half-points, and effort tiebreaker

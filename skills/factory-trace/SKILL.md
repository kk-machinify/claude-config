---
name: factory-trace
description: Use when the user asks to trace, audit, or verify that AI-repo guardrails are actually firing (not just present). Companion to ai-repo-readiness factory-score — answers "did the guardrails catch anything in the last 90 days?" rather than "do the guardrails exist?". Produces a per-gate trace report with evidence URLs and a `L<n> nominal vs L<m> effective` verdict.
version: 0.1.0
disable-model-invocation: true
context: fork
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, WebFetch
---

> ⚠️ **NOT YET VETTED** — This skill is a draft (v0.1) that has not been reviewed or approved by the CTO. Thresholds, signals, and verdict bands are provisional. Treat output as exploratory until the open questions at the bottom of this file are resolved. Do not cite results as authoritative repo assessments.

# factory-trace — Spec Draft v0.1

**Status:** Draft for CTO review. Standalone skill, separated from `ai-repo-readiness`.
**Companion to:** Section 5 of `ai-repo-readiness/SKILL.md` (factory-score).

---

## Why this exists

`factory-score` checks that guardrails *exist* (tests directory present, typecheck in CI, branch protection ruleset configured, adversarial reviewer subagent file checked in). It does not check whether those guardrails *catch anything*. A repo can score 16/18, L2 nominal, while shipping AI-authored code through:

- tests that assert on data they constructed (vacuous, always pass)
- a typecheck job that runs against `any`-littered files
- branch protection routinely bypassed by admin override
- an "adversarial reviewer" that emits `LGTM 👍` on every PR

## Strategy: same gates, different question

factory-score answers *"could this repo be safe?"* (does the mechanism exist?). factory-trace answers *"in the last 90 days, did the guardrails fire?"* — same six categories and ten gates, different question per gate.

Roughly 80% of v0.1's signals look at the merge log, CI history, and comment threads rather than file existence. Examples with no equivalent in factory-score:

- % merges via admin override
- % direct pushes that bypassed PR
- hotfix-within-7-days rate on AI-authored merges
- % CI runs that *actually failed* in window (zero failures in 90d strongly suggests trivially-passing tests or `|| true`-wrapped jobs)
- `--no-verify` commit count
- reviewer address rate (substantive comments → commits before merge)
- cron PR merge-vs-ignore fate

**Headline deliverable factory-score can't produce:** the `L2 nominal, L1 effective` gap with 3 evidence URLs per failed verdict. You can click through and verify the claim instead of trusting a presence check.

### What v0.1 still doesn't catch (honest)

- **Vacuous tests that happen not to be exercised by recent bugs** — mutation testing is the only authoritative check and it's `[future]` because it executes repo code.
- **LGTM-bot calibration** — the 20-comment sample is directionally useful, not authoritative; the human-agreement signal that would settle it is also `[future]`.
- **Agent session quality** — no telemetry on what humans/agents actually asked or were misled by, so G5 stays a weak proxy.

### Validation criterion

If on the first 5–10 repos the cheap signals routinely agree with factory-score (i.e., nominal == effective on every gate), that's evidence v0.1 isn't earning its keep and we should fast-track the `[future]` heavy signals rather than keep iterating on cheap ones.

## Code execution & data egress

**v0.1 default mode is read-only.** No code from the target repo is executed. Concretely:

| Operation | Reads repo | Executes repo code | Egress |
|---|---|---|---|
| `gh api repos/...`, `gh run list`, `gh pr list` | — | no | GitHub API |
| `git log`, `git ls-files`, `git diff` | yes | no | none |
| `find` / `wc -l` / grep for pragma strings | yes | no | none |
| `gh repo clone` (URL mode) | writes to `/tmp/` | no | GitHub |
| File mtime / size checks | yes | no | none |

**Medium signals add one egress path:**

- **G6 comment-substance rating** ships sampled PR comment text (≤20 comments per run) to the Claude API for classification. Repo code is *not* sent. Skip with `--no-llm-rate` if PR comments may contain secrets.

**`[future]` heavy signals execute repo code — gated behind explicit flag:**

- **G4 mutation kill rate** runs the target repo's test suite via `mutmut` / `stryker` / `cargo-mutants`. This executes arbitrary code: test fixtures, conftest hooks, build scripts, anything the suite imports. **Requires `--allow-execute` flag, never on by default.** Recommendation: only run inside a sandbox (container, ephemeral VM, or worktree-isolated environment) and only against repos whose code you'd already run locally.

**Optional integrations (egress only, no execution):**

- FireHydrant API for incident-touching-repo signal — egress to FireHydrant, opt-in via token presence.
- Notion MCP for cross-referencing the "Building the Software Factory" doc — egress to Notion, opt-in.

**Output safety:** the report contains PR URLs, commit SHAs, file paths, and pragma counts. It does not include source code excerpts. Sampled PR comments are stored in `~/ai-repo-readiness/factory-trace-<repo>-YYYY-MM-DD.md` only if `--keep-evidence` is set; default is to log URLs only.

## Invocation

```
/factory-trace [repo-path-or-url] [--window=90d] [--since-pr=N]
```

- `--window`: time window for trace signals (default 90d, mtime-drift signals use 180d regardless)
- `--since-pr`: alternative to time window — last N merged PRs (default 100)

## Output

`~/ai-repo-readiness/factory-trace-<repo>-YYYY-MM-DD.md` with:

1. **Data sufficiency banner** — which signals had enough data, which were skipped, why
2. **Per-gate trace table** — for each G1–G10 gate from factory-score: `presence` (from static), `trace verdict`, `evidence (3 PR URLs or git refs)`, `what this does NOT measure`
3. **Effective Level** — lowest level for which every required trace verdict is ≥ `weak`. If insufficient data, declare `insufficient-data`, not a default-pass
4. **Notion compact form:** `15/18, L2 nominal, L1 effective` (or `L2 (trace ok)` when nominal == effective)

## Minimum data requirements

| Requirement | Why |
|---|---|
| `gh auth status` succeeds | All GitHub-API signals |
| ≥30 merged PRs in `--window` OR ≥6 months of git history | Stat signals need a denominator |
| Default branch resolvable | Branch-protection + merge-path signals |
| **Skip with `insufficient-data` if below threshold** — do NOT degrade to a default-pass |

Optional integrations (degrade gracefully if missing):

- FireHydrant token → "incidents touching this repo in window" signal
- Mutation-test runner (`mutmut`/`stryker`/`cargo-mutants`) installed in repo → kill-rate signal
- LLM rater (Claude API) → comment-substance signal

## Per-gate trace signals

Three cost tiers: **cheap** (gh API + git log only), **medium** (sampling + LLM rating), **heavy** (running tools against the repo). v0.1 ships cheap + medium; heavy signals are marked `[future]`.

### G1 — Context maps to modules

| Field | Value |
|---|---|
| **Presence (factory-score)** | per-module README/CLAUDE/AGENTS in ≥50% top-level dirs |
| **Trace signal** | **mtime drift vs code churn.** For each per-module doc, compute (commits to that module in 180d) / (days since doc mtime). High ratio = doc is fiction. |
| **Verdict** | `pass`: ≥70% of per-module docs have ratio <0.5 commits/day. `weak`: 40–70%. `fail`: <40%. |
| **Cost** | cheap |
| **Does NOT measure** | Whether doc *content* is correct — only that someone touched it as the surrounding code evolved. |

### G2 — Type checker + linter in CI

| Field | Value |
|---|---|
| **Presence** | both present + referenced in `.github/workflows/*.yml` |
| **Trace signal A** | **Ignore-pragma density.** Count `any` / `# type: ignore` / `@ts-ignore` / `eslint-disable*` / `# noqa` per kloc. Trend over window. |
| **Trace signal B** | **Has the job actually failed?** `gh run list --workflow=<typecheck>` — % of runs failed in window. If 0%, either trivially passing or job is wrapping `\|\| true`. |
| **Trace signal C** | **`--no-verify` commits.** `git log --grep="no-verify"` + check commit messages for hook bypass markers. |
| **Verdict** | `pass`: density <2/kloc AND ≥1 failed CI run in window. `weak`: one of the two. `fail`: density >5/kloc OR zero failed runs in 90 days. |
| **Cost** | cheap |
| **Does NOT measure** | Whether the type system models the domain — only whether it's being honored. |

### G3 — Branch protection

| Field | Value |
|---|---|
| **Presence** | active ruleset on default branch |
| **Trace signal A** | **% merges via admin override.** `gh api repos/<owner>/<repo>/pulls?state=closed` filtered to `merged_at` in window, cross-ref with rule-bypass events. |
| **Trace signal B** | **% direct pushes to default branch** (bypass PR entirely). `git log origin/main --merges` vs `git log origin/main` count. |
| **Trace signal C** | **% PRs merged with required checks failing.** Inspect PR review/check history. |
| **Verdict** | `pass`: all three <2%. `weak`: any one 2–10%. `fail`: any >10%. |
| **Cost** | cheap |
| **Does NOT measure** | Whether the *rules being enforced* are the right ones. |

### G4 — Behavior-verifying tests on PR

| Field | Value |
|---|---|
| **Presence** | tests directory + CI runs on PR |
| **Trace signal A (primary)** | **Hotfix-within-7-days rate.** Of PRs merged in window, % followed by a PR within 7d whose title/body matches `fix\|hotfix\|revert\|rollback\|broken\|regression` and touches files overlapping ≥50% with the original PR. Subset: same metric restricted to AI-authored PRs (heuristic: Claude/Cursor/Copilot mentions in PR description, or `[bot]` author). |
| **Trace signal B** | **% PRs where CI passed first try.** Vacuous tests pass first time, every time. A healthy repo should have *some* PRs that needed a CI fix. |
| **Trace signal C** `[future]` | **Mutation kill rate.** If mutation runner is configured, run it; require ≥60% kill rate for `pass`. |
| **Verdict** | `pass`: hotfix rate <5% AND first-try-pass rate 60–95%. `weak`: hotfix 5–15% OR first-try-pass outside 50–98%. `fail`: hotfix >15% OR first-try-pass >98% (suspicious). |
| **Cost** | cheap (A, B); heavy (C) |
| **Does NOT measure** | Test *quality* directly — only the downstream behavior. A repo with no AI activity can't be assessed on the AI subset; report `insufficient-data` for that slice. |

### G5 — Specialist skills / subagents / commands

| Field | Value |
|---|---|
| **Presence** | `.claude/skills/` ≥1 OR `.claude/agents/` ≥1 OR `.claude/commands/` ≥3 |
| **Trace signal** | **Edit recency vs repo activity.** For each skill/subagent/command file, mtime within window? If 0/N have been touched in 90d while the repo had >50 commits, the assets are likely abandoned. **Weak proxy only** — we don't have invocation telemetry from agent sessions. |
| **Verdict** | `pass`: ≥1 asset touched in 90d (or repo itself had <20 commits, no expected churn). `weak`: assets touched 90–180d. `fail`: assets >180d while repo active. |
| **Cost** | cheap |
| **Does NOT measure** | Whether agents/humans actually *use* these — that requires session telemetry we don't have. Flag this honestly in output. |

### G6 — Adversarial reviewer

| Field | Value |
|---|---|
| **Presence** | review subagent file OR `claude-pr-review.yml`-style workflow |
| **Trace signal A** | **Activation rate.** % of PRs in window where the reviewer left ≥1 comment. |
| **Trace signal B** | **Blocking rate.** Of those, % where the comment was substantive (not pure approval). Sample N=20 comments; LLM-rate as `lgtm-only` / `nit` / `substantive` / `blocking`. |
| **Trace signal C** | **Address rate.** % of substantive comments followed by a commit before merge (vs dismissed/ignored). |
| **Trace signal D** `[future]` | **Human-agreement rate.** When the bot blocked and a human approved (or vice versa), who was right per follow-up history. |
| **Verdict** | `pass`: A ≥80%, B (substantive+blocking) ≥30%, C ≥60%. `weak`: any two of the three. `fail`: ≤1 of the three. |
| **Cost** | cheap (A, C); medium (B); heavy (D) |
| **Does NOT measure** | Whether the bot is calibrated correctly — only that it's emitting non-trivial output that humans engage with. |

### G7 — Event-triggered cloud Claude

| Same trace signals as G6 if it's the same bot. Otherwise score independently. |
|---|

### G8 — Cron-triggered cloud agent

| Field | Value |
|---|---|
| **Presence** | workflow with `schedule:` invoking Claude |
| **Trace signal A** | **Last successful run.** `gh run list --workflow=<cron>` — most recent success within window? |
| **Trace signal B** | **PR fate.** Of PRs opened by this workflow, % merged vs % closed-without-merge vs % stale-open (>30d). |
| **Trace signal C** | **Run failure rate.** % failed runs in window. |
| **Verdict** | `pass`: A within 14d, B merge rate ≥50%, C <20% failures. `weak`: A within 30d. `fail`: A >30d OR merge rate <20% (the bot is being ignored). |
| **Cost** | cheap |
| **Does NOT measure** | Whether the bot's *suggestions* are valuable — only whether they land. |

### G9 — Quality verification beyond "tests pass"

| Field | Value |
|---|---|
| **Presence** | mutation config OR property tests OR merge-blocking Claude review |
| **Trace signal** | Depends on which one:<br>• Mutation: actual kill rate from last run (`[future]` — heavy)<br>• Property: count of property tests, % invoked in CI<br>• Merge-blocking Claude: % PRs the bot actually blocked vs merged anyway via override |
| **Verdict** | Per the active mechanism. |
| **Cost** | medium / heavy |
| **Does NOT measure** | Beyond stated. |

### G10 — Self-curating knowledge

| Field | Value |
|---|---|
| **Presence** | scheduled job that updates docs/changelog/lessons |
| **Trace signal A** | **Did the curation job actually run and commit?** Look for commits in window authored by the workflow's bot identity. |
| **Trace signal B** | **Knowledge freshness vs commit volume.** Has `CHANGELOG.md` / `docs/lessons/` been updated within window, while repo had >50 commits? |
| **Verdict** | `pass`: A ≥1 commit in window AND B fresh. `weak`: one of two. `fail`: neither. |
| **Cost** | cheap |
| **Does NOT measure** | Whether the curated content is *useful* — could be auto-generated noise. |

## Verdict synthesis

For each Level, the gates required for the *nominal* level are re-evaluated using trace verdicts:

- **Effective L2** requires: G1–G6 all have trace verdict ≥ `weak` (not `fail` or `insufficient-data`).
- **Effective L3** requires: L2 effective + G7–G10 all ≥ `weak`.

If any required trace signal is `insufficient-data`, the effective level is reported as `L<n-1> + <gate> insufficient` — never silently promoted.

**Disagreement is itself a finding.** A repo at `L2 nominal, L1 effective` is the most useful signal this skill produces — it tells the team exactly which guardrail to harden. The output should highlight disagreements at the top, not bury them.

## Honest-framing rules

These are non-negotiable for the output:

1. **`insufficient-data` is visible, not hidden.** Never default-pass a signal we couldn't compute.
2. **Every verdict carries 3 evidence URLs** (PR links, git SHAs, or `gh run` URLs). No claim without a trace.
3. **Every signal declares what it does NOT measure.** Already in the per-gate tables — keep it in the output too.
4. **AI-authored slice is reported separately** when computable. The aggregate hotfix rate can hide an AI-specific problem.
5. **No effectiveness number that combines presence + trace into a single hidden formula.** Report both, let the reader weigh.

## Out of scope for v0.1

- Mutation-test execution (heavy, language-specific) — track as `[future]`
- LLM-based comment-substance rating beyond a 20-sample audit — needs evaluation harness
- Cross-repo correlation (org-wide trends)
- Prod-incident → repo attribution (FireHydrant integration is best-effort, optional)
- Agent session telemetry (we don't have it)

## Open questions

1. Is **hotfix-within-7-days** the right primary signal for G4, or should we anchor on revert/rollback specifically?
2. Should we restrict the AI-authored subset using `[bot]` author + co-author trailers, or expand the heuristic (PR description keyword scan)?
3. For G6 substance rating: 20-sample audit per run feels right for cost, but we could go to 50 with batching. Preference?
4. Acceptable to require `gh` CLI auth, or do we need a token-only path for CI use?
5. Should the verdict band `L2 nominal, L1 effective` be the *primary* output (top of doc) and the static score secondary, given your framing?

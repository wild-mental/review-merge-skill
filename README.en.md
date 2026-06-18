![AI Skills for Everyone](author/wildmental-bjpark.png)

# review-merge
> Skill for Cursor, Claude, Codex agents

**Language / 언어:** [한국어](README.md) · [English](README.en.md)

**A human-in-the-loop sequential review Skill that reviews PRs one at a time and merges each one only after it passes a verification gate (REVIEW → MERGE). Use it when PRs are independent or each one must clear an explicit gate before shipping, when you need per-PR review records, and when a human merges remotely on every approval. It formalizes a CLI-authoritative verification gate, the no-cascade push rule, and defenses against stack-merge pitfalls such as auto-delete-head, draft PRs, and git push credentials. Supports Cursor, Claude Code, and Codex.**

Review is fundamentally **behavior-based**, and behavior lives per feature. If PRs are independent of one another, or *each must pass a gate before shipping*, there's no reason to bundle and merge them together first. Rather, a flow of **review one at a time → verification gate → human merges remotely** leaves a clear record of the review and the blocking points.

`review-merge` formalizes this flow. The *review and fixes* are committed by the agent to the branch and handled with a single push, but **the actual merge decision and the remote merge are done by a human** (AI fixes are never auto-merged before approval). It runs in exactly the opposite direction of its sibling skill [merge-review](https://github.com/wild-mental/merge-review-skill) (MERGE → REVIEW); the mode-selection table below decides which of the two to use.

---

## When to use this skill (mode selection)

> Core thesis: Review is **behavior-based**, and behavior lives per feature. Therefore human-reviewer efficiency is maximized when the review unit = *one cohesive review surface*. The two modes diverge from this principle.

| Signal | **REVIEW → MERGE (this skill)** | MERGE → REVIEW ([merge-review](https://github.com/wild-mental/merge-review-skill)) |
| --- | --- | --- |
| Relationship between PRs | **Independent** (don't call each other, each can ship on its own) or coupled but a gate is required before shipping | Dependency-coupled (an earlier PR is dead code until a later one) |
| Review surface | Each PR is reviewable on its own surface | Must converge onto a single user surface for the behavior to *exist* |
| Risk profile | **Different** (don't mix security vs. presentation in one merge) | Similar within the bundle |
| Requirements | Per-PR blocking and review records needed before shipping | Live review only possible after integration |

**Use this skill:** when each PR must be gated independently / before shipping, when you need per-PR review records, and when the flow is a human merging remotely after each approval.

> One-line verdict: **"Does this change have to pass on a per-PR basis *before* merge?"** — if yes, this skill.

---

## What this skill solves

### 1. Put a blocking gate on every PR before merge

No PR is merged before it passes the verification gate (§4), which covers static, unit, build, and DB integration. The gate's **authoritative basis is the CLI result, not the editor LSP**. It isn't fooled by stale Prisma/module errors right after a branch switch.

### 2. The agent reviews; the human decides the merge

The *review and fixes* are committed by the agent to the owning branch only and handled with a single push. **The actual merge decision and the remote merge are done by a human.** AI fixes are never auto-merged before approval.

### 3. Don't break the stack invariant

A parent must remain an ancestor of its child. When you merge a parent into main, **retarget** the child's base to main; never break the ancestor relationship by merging main directly into the child.

### 4. No-Cascade — prevent a flood of preview deploys

Don't batch-push multiple branches (cascade) to synchronize the stack. A branch with an open PR generates **one preview deploy per push** (e.g., Vercel). Push **only** the one branch you changed, and let the behind state of the rest reconcile naturally at merge time.

### 5. Defend the common stack-merge pitfalls with procedure

A downstream PR getting **CLOSED instead of merged** by `auto-delete-head`, marking a draft PR as ready, an unauthenticated `git push`, stale artifacts — it formalizes a prevention and recovery procedure for each.

### 6. Provide per-domain repeat-check surfaces

It organizes the checkpoints to verify every time for security/auth (token, session cookie, authz matrix, existence non-disclosure), render/content (sanitize boundary, RSC props identification), and embed/headers (`X-Frame-Options`, CSP `frame-src`).

---

## Why you need this skill

| Limits of gateless sequential merging | review-merge's prescription |
|-----------------|-----------------|
| Misjudging because you're fooled by stale editor LSP errors | Gate with **CLI tsc/test** as the authoritative basis |
| AI fixes get merged before approval | Agent reviews/fixes, **human decides the merge** |
| Merging main directly into a child collapses the ancestor relationship | Retarget the child's base to main after merging the parent |
| Batch-pushing branches for stack sync → preview flood | **no-cascade** — push only the changed branch |
| auto-delete-head silently CLOSES a downstream PR | Move the base first + recovery procedure (§7.1) |
| The integration gate env doesn't take, so all tests skip | Verify the exact string match `RUN_DB_TESTS=true` |

---

## Verification gate (required before fixes and merge)

The gate's **authoritative basis is the CLI** result. Order of passing:

```
0) Normalize the toolchain (unset version-manager functions + prioritize the node path)
1) Clean branch contamination + regenerate artifacts (rm -rf .next lib/generated → db:generate → manifest/lessons build)
2) Static + unit (typecheck && lint && test)
3) Build (inject secrets only at the OS level — don't read them from context)
4) DB integration (RUN_DB_TESTS=true if this PR touches the DB)
```

**Gate gotcha:** the integration gate env requires an **exact value match** (the string `RUN_DB_TESTS=true`; `=1` won't take → suspect it if "all skipped") · hardcoded assertions should be content-derived · a missing-artifact `Cannot find module ...registry` is the generation script not having run, not the code · a `| tail` pipe masks the exit code (check `${pipestatus}`).

---

## Quick start

### Prerequisites

- Cursor, Claude Code, or Codex
- GitHub `gh` CLI authentication, the **PR stack/set to review**, and the **verification gate** to pass (typecheck/lint/test/build, plus DB integration if needed)
- A **human reviewer** to perform the remote merge (the merge decision-maker)

### Installing the skill

Install the skill by choosing one of two scopes: **personal** or **project**. Both scopes fetch only `SKILL.md` with `curl`. **Do not clone this entire repository into your working repo.**

| | Personal skill | Project skill |
|---|----------|--------------|
| **Scope** | Every project I open | Only in the current repo |
| **Path** | `~/…/skills/review-merge/` | `<repo-root>/.cursor/skills/review-merge/`, etc. |
| **Git impact** | No files added to the working repo | Skill files can be committed to the repo (team sharing) |
| **When to use** | When using it alone across all projects | When pinning and sharing the skill in a team repo |

| Tool | Personal path | Project path |
|------|-----------|---------------|
| Cursor | `~/.cursor/skills/review-merge/` | `.cursor/skills/review-merge/` |
| Claude Code | `~/.claude/skills/review-merge/` | `.claude/skills/review-merge/` |
| Codex | `~/.agents/skills/review-merge/` | `.agents/skills/review-merge/` |

#### Personal skill (recommended — no change to the Git repo)

```bash
# Cursor
mkdir -p ~/.cursor/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.cursor/skills/review-merge/SKILL.md \
  -o ~/.cursor/skills/review-merge/SKILL.md

# Claude Code
mkdir -p ~/.claude/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.claude/skills/review-merge/SKILL.md \
  -o ~/.claude/skills/review-merge/SKILL.md

# Codex
mkdir -p ~/.agents/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.agents/skills/review-merge/SKILL.md \
  -o ~/.agents/skills/review-merge/SKILL.md
```

#### Project skill (install into the project path)

Use this only when you want to include and share the AI skill setup in the repo.

```bash
# Cursor
mkdir -p .cursor/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.cursor/skills/review-merge/SKILL.md \
  -o .cursor/skills/review-merge/SKILL.md

# Claude Code
mkdir -p .claude/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.claude/skills/review-merge/SKILL.md \
  -o .claude/skills/review-merge/SKILL.md

# Codex
mkdir -p .agents/skills/review-merge
curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.agents/skills/review-merge/SKILL.md \
  -o .agents/skills/review-merge/SKILL.md
```

Just pick and run the tool(s) you need (Cursor / Claude Code / Codex).

#### After installing

- **Cursor**: **Reload Window** once
- **Claude Code**: skill edits reflect live during a session; a new top-level `.claude/skills/` after a session has started may require a restart
- **Codex**: if the skill doesn't appear, restart Codex

### How to use

Tell it the PR stack/set to review and ask "let's review with a flow that passes each one through a gate one at a time and a human merges," and the Agent applies the skill automatically.

| Tool | Manual invocation |
|------|-----------|
| Cursor | `/review-merge` |
| Claude Code | `/review-merge` |
| Codex | `/skills` or `$review-merge` |

**Example requests that trigger it:**

- "Review this PR stack one at a time from the bottom, and I'll merge only the ones that pass the verification gate"
- "Each PR is independent, so they need to be gated separately before shipping. Look at them one at a time"
- "It's a security PR, so verify the token hash, session cookie, and authz matrix with code as evidence"
- "Don't push all the branches just to sync the stack. Push only what you changed and reconcile the rest at merge time"

---

## OUTPUT: what remains

The deliverable is **what remains**, not the conversation.

| Deliverable | Contents |
|--------|------|
| **Per-PR review record** | Orientation, code-evidenced responses, runtime verification, and PR-body updates remain on a per-PR basis |
| **Gate-passing history** | Each merged item ships in a state where it has passed the typecheck/lint/test/build (+DB integration) gate |
| **Reconciled local/remote state** | After merge: main FF, local branch cleanup, retarget the next PR's base to main |
| **Harness reflection** | Pin the no-cascade, CLI-authoritative gate, and child-base-retarget-to-main invariants in the project harness (`CLAUDE.md` / `AGENTS.md` / `.cursor/rules`) |

---

## Workflow

| Step | Contents |
|------|------|
| 3. Single-PR review cycle | Orientation → interactive review (code-evidenced) → fixes (owning branch only) → PR-body update → human merges remotely |
| 4. Verification gate | Normalize toolchain → regenerate artifacts → static/unit → build → DB integration (CLI authoritative) |
| 5. Post-merge local cleanup | fetch --prune → main FF → delete merged branch → retarget the next PR's base to main |
| 6. No-Cascade | Push only the changed branch, reconcile the rest at merge, pin in the harness |
| 7. gotcha | auto-delete-head recovery, draft ready, git push credentials, stale artifacts |

---

## Skill structure

```
.cursor/skills/review-merge/SKILL.md   # for Cursor
.claude/skills/review-merge/SKILL.md   # for Claude Code
.agents/skills/review-merge/SKILL.md   # for Codex
```

| Section | Contents |
|------|------|
| Role | REVIEW → MERGE — a human merges after each one passes the gate, one at a time |
| Mode selection | The mode-selection table vs. merge-review |
| Mental model | Topology, role split, stack invariant |
| Single-PR review cycle | Orientation → interactive review → fixes → body update → remote merge |
| Verification gate | CLI authority + gate gotcha |
| Post-merge cleanup / No-Cascade | Local cleanup and base retargeting, no batch push |
| Known gotchas | auto-delete-head, draft, push credentials, stale artifacts |
| Per-domain checks / checklist / environment-adaptation notes | Security/render/embed surfaces, PR/post-merge checks, per-repo bindings |

---

## Recommended for

- Those whose PRs are **independent, or who must gate each one before shipping**
- Those who want to judge whether to merge using the **CLI as the authoritative basis**, not the editor LSP
- Those who want to delegate review and fixes to the AI but **keep the merge decision with a human**
- Those who want to use no-cascade to prevent a **flood of preview deploys** from stack synchronization
- Those who want to handle pitfalls like `auto-delete-head`, draft, and push credentials **with procedure**
- Those who want to **choose, depending on the situation,** between this and [merge-review](https://github.com/wild-mental/merge-review-skill), which integrates the bundle first

---

## References

- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)
- Sibling skill: [merge-review](https://github.com/wild-mental/merge-review-skill) (MERGE → REVIEW)

---

## For LLMs

<!-- install-only; operational rules live in SKILL.md — load skill after install -->

```
skill=review-merge
repo=https://github.com/wild-mental/review-merge-skill
sibling=merge-review  # inverse MERGE → REVIEW mode
install.method=curl-only  # never clone this repo into user's project

flow:
  1. ask scope: "개인 스킬(~/, 모든 프로젝트) vs 프로젝트 스킬(./, 현재 repo만) 중 어디에 설치할까요?"
  2. ask tools: "Cursor / Claude Code / Codex 중 어떤 도구에 설치할까요?" (복수 선택 가능)
  3. run matching curl commands below; do not install both scopes unless user asks
  4. post_install steps; then load SKILL.md — do not infer rules from README

scope.user.paths:
  cursor=~/.cursor/skills/review-merge/SKILL.md
  claude=~/.claude/skills/review-merge/SKILL.md
  codex=~/.agents/skills/review-merge/SKILL.md

scope.project.paths:
  cursor=.cursor/skills/review-merge/SKILL.md
  claude=.claude/skills/review-merge/SKILL.md
  codex=.agents/skills/review-merge/SKILL.md

install.user.cursor=mkdir -p ~/.cursor/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.cursor/skills/review-merge/SKILL.md -o ~/.cursor/skills/review-merge/SKILL.md
install.user.claude=mkdir -p ~/.claude/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.claude/skills/review-merge/SKILL.md -o ~/.claude/skills/review-merge/SKILL.md
install.user.codex=mkdir -p ~/.agents/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.agents/skills/review-merge/SKILL.md -o ~/.agents/skills/review-merge/SKILL.md

install.project.cursor=mkdir -p .cursor/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.cursor/skills/review-merge/SKILL.md -o .cursor/skills/review-merge/SKILL.md
install.project.claude=mkdir -p .claude/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.claude/skills/review-merge/SKILL.md -o .claude/skills/review-merge/SKILL.md
install.project.codex=mkdir -p .agents/skills/review-merge && curl -fsSL https://raw.githubusercontent.com/wild-mental/review-merge-skill/main/.agents/skills/review-merge/SKILL.md -o .agents/skills/review-merge/SKILL.md

install.project.note=run from repo root; adds tracked files — confirm user chose project scope

invoke.cursor=/review-merge
invoke.claude=/review-merge
invoke.codex=/skills|$review-merge

post_install.cursor=Reload Window
post_install.claude=live reload; restart if new top-level .claude/skills/ after session start
post_install.codex=restart if skill not detected

contract:
  mode=REVIEW → MERGE (review each PR one at a time, gate before merge, human merges remotely)
  use_when=[PRs independent or must each clear an explicit gate before shipping, per-PR review records needed, human merges after each approval]
  use_sibling_instead_when=[earlier PRs dead code until wired up, group converges on one surface, batch shares one risk profile, rollback cheap] -> merge-review
  gate_test=must this change clear a PR-level gate BEFORE merge?
  gate_authority=CLI result (ignore stale editor LSP diagnostics)
  gate_steps=[normalize toolchain, rm -rf .next lib/generated + regenerate, typecheck && lint && test, build (secrets at OS level), RUN_DB_TESTS=true integration]
  role_split=agent reviews/edits on owning branch + single push; HUMAN decides and performs remote merge (never auto-merge AI fixes pre-approval)
  stack_invariant=parent stays ancestor of child; on parent merge retarget child base to main; never merge main into child
  no_cascade=push only the changed branch; never batch-push branches (each open-PR push = 1 preview deploy); behind branches reconcile at merge time
  post_merge=[fetch --prune, main ff, delete merged local branches, retarget next PR base to main]
  gotchas=[auto-delete-head closes downstream PR (CLOSED not merged) -> restore base ref/reopen/retarget/redelete, draft PR needs gh pr ready, git push unauth -> gh auth setup-git, stale LSP/artifacts -> CLI tsc authoritative + regenerate]
  domain_surfaces=[security/auth (token hash/single-use/expiry, session cookie attrs, authz matrix 200/403/404, existence non-disclosure), render/content (sanitize boundary, RSC props identity not type), embed/headers (X-Frame-Options, CSP frame-src)]
  harness_fixation=[no-cascade, CLI-authoritative gate, retarget child base to main] -> CLAUDE.md | AGENTS.md | .cursor/rules/*.mdc
```

---

## License

[MIT License](LICENSE)

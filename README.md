![AI Skills for Everyone](author/wildmental-bjpark.png)

# review-merge
> Skill for Cursor, Claude, Codex agents

**PR을 한 건씩 검토하고, 매 건이 검증 게이트를 통과한 뒤에만 머지하는 human-in-the-loop 순차 리뷰 Skill입니다(REVIEW → MERGE). PR이 독립이거나 출고 전 각 건이 명시적 게이트를 통과해야 할 때, PR별 리뷰 레코드가 필요할 때, 사람이 승인마다 원격 머지하는 흐름에 씁니다. CLI를 권위 기준으로 한 검증 게이트, no-cascade push 규칙, auto-delete-head·draft PR·git push 자격 같은 스택 머지 함정의 방어를 정형화합니다. Cursor, Claude Code, Codex 모두 지원합니다.**

리뷰는 본질적으로 **행위(behavior) 기반**이고, 행위는 기능별로 삽니다. PR들이 서로 독립이거나 *출고 전에 각각 게이트를 통과*해야 한다면, 묶어서 먼저 머지할 이유가 없습니다. 오히려 **한 건씩 리뷰 → 검증 게이트 → 사람이 원격 머지**하는 흐름이 리뷰 레코드와 차단 지점을 명확히 남깁니다.

`review-merge`는 이 흐름을 정형화합니다. *리뷰·수정*은 에이전트가 브랜치에 커밋하고 단일 push로 처리하되, **실제 머지 결정과 원격 머지는 사람**이 합니다(AI 수정안을 승인 전 자동 머지하지 않음). 자매 스킬 [merge-review](https://github.com/wild-mental/merge-review-skill)(MERGE → REVIEW)와 정확히 반대 방향이며, 둘 중 무엇을 쓸지는 아래 모드 선택 표로 가릅니다.

---

## 언제 이 스킬인가 (모드 선택)

> 핵심 명제: 리뷰는 **행위 기반**이고 행위는 기능별로 산다. 따라서 리뷰 단위 = *하나의 응집된 검토 표면*일 때 인간 리뷰어 효율이 최대다. 이 원칙에서 두 모드가 갈린다.

| 신호 | **REVIEW → MERGE (이 스킬)** | MERGE → REVIEW ([merge-review](https://github.com/wild-mental/merge-review-skill)) |
| --- | --- | --- |
| PR 간 관계 | **독립**(서로 호출 안 함, 각자 출고 가능) 또는 출고 전 게이트가 필수인 결합 | 의존-결합(앞 PR이 뒤 PR 전엔 dead code) |
| 검토 표면 | PR마다 자기 표면에서 검토 가능 | 하나의 사용자 표면으로 수렴해야 동작이 *존재* |
| 위험 프로파일 | **상이**(보안 vs 표현을 한 머지에 섞지 않음) | 묶음 내 유사 |
| 요구사항 | 출고 전 PR별 차단·리뷰 레코드 필요 | 통합 후에야 라이브 검토 가능 |

**이 스킬을 쓴다:** 각 PR을 독립적으로/출고 전에 게이트해야 할 때, PR별 리뷰 기록이 필요할 때, 사람이 매 승인 후 원격 머지하는 흐름일 때.

> 한 줄 판정: **"이 변경을 머지 *전에* PR 단위로 통과시켜야 하는가?"** — 예면 이 스킬.

---

## 이 스킬이 해결하는 것

### 1. 머지 전에 PR마다 차단 게이트를 둔다

매 PR이 정적·단위·빌드·DB 통합을 포함한 검증 게이트(§4)를 통과하기 전에는 머지하지 않습니다. 게이트의 **권위 기준은 에디터 LSP가 아니라 CLI 결과**입니다. 브랜치 전환 직후의 stale Prisma/모듈 오류에 속지 않습니다.

### 2. 리뷰는 에이전트, 머지 결정은 사람

*리뷰·수정*은 에이전트가 owning 브랜치에만 커밋하고 단일 push로 처리합니다. **실제 머지 결정과 원격 머지는 사람**이 합니다. AI 수정안을 승인 전에 자동 머지하지 않습니다.

### 3. 스택 불변식을 깨지 않는다

부모가 자식의 조상으로 남아야 합니다. 부모를 main에 머지하면 자식 base를 main으로 **재타게팅**하고, 절대 자식에 직접 main을 머지해 조상 관계를 깨지 않습니다.

### 4. No-Cascade — preview 배포 폭증을 막는다

스택 동기화를 위해 여러 브랜치를 일괄 push(cascade)하지 않습니다. PR이 열린 브랜치는 push마다 **preview 배포가 1건씩** 생깁니다(예: Vercel). 변경한 브랜치 **하나만** push하고, 나머지의 behind 상태는 머지 시점에 자연 reconcile되게 둡니다.

### 5. 스택 머지의 흔한 함정을 절차로 방어한다

`auto-delete-head`로 하위 PR이 **머지가 아니라 CLOSED**되는 사고, draft PR ready 처리, `git push` 미인증, stale 생성물 — 각각의 예방·복구 절차를 정형화합니다.

### 6. 도메인별 반복 확인 표면을 제공한다

보안/인증(토큰·세션 쿠키·인가 매트릭스·존재 비노출), 렌더/콘텐츠(sanitize 경계·RSC props 식별), 임베드/헤더(`X-Frame-Options`·CSP `frame-src`)에서 매번 확인할 체크포인트를 정리합니다.

---

## 왜 skill 이 필요한가

| 게이트 없는 순차 머지의 한계 | review-merge의 처방 |
|-----------------|-----------------|
| 에디터 LSP의 stale 오류에 속아 잘못 판단 | **CLI tsc/test**를 권위 기준으로 게이트 |
| AI 수정안이 승인 전에 머지됨 | 리뷰·수정은 에이전트, **머지 결정은 사람** |
| 자식에 main을 직접 머지해 조상 관계 붕괴 | 부모 머지 후 자식 base를 main으로 재타게팅 |
| 스택 동기화로 여러 브랜치 일괄 push → preview 폭증 | **no-cascade** — 변경 브랜치 하나만 push |
| auto-delete-head로 하위 PR이 조용히 CLOSED | 사전 base 이동 + 복구 절차(§7.1) |
| 통합 게이트 env가 안 먹어 테스트가 전부 skip | `RUN_DB_TESTS=true` 문자열 정확 일치 확인 |

---

## 검증 게이트 (수정·머지 전 필수)

게이트의 **권위 기준은 CLI** 결과입니다. 통과 순서:

```
0) 툴체인 정규화 (버전 매니저 함수 unset + node 경로 우선)
1) 브랜치 오염 제거 + 생성물 재생성 (rm -rf .next lib/generated → db:generate → manifest/lessons build)
2) 정적 + 단위 (typecheck && lint && test)
3) 빌드 (비밀값은 OS 레벨로만 주입 — 컨텍스트로 읽지 않음)
4) DB 통합 (해당 PR이 DB를 건드리면 RUN_DB_TESTS=true)
```

**게이트 gotcha:** 통합 게이트 env는 **정확한 값 일치**(`RUN_DB_TESTS=true` 문자열, `=1`은 안 먹음 → "all skipped"면 의심) · 하드코딩 어서션은 content-derived로 · 생성물 누락 `Cannot find module ...registry`는 코드가 아니라 생성 스크립트 미실행 · `| tail` 파이프는 종료코드를 가림(`${pipestatus}` 확인).

---

## 빠른 시작

### 사전 요구사항

- Cursor, Claude Code, 또는 Codex
- GitHub `gh` CLI 인증, **검토할 PR 스택/집합**과 통과시킬 **검증 게이트**(typecheck/lint/test/build, 필요시 DB 통합)
- 원격 머지를 수행할 **사람 리뷰어**(머지 결정 주체)

### 스킬 설치

스킬은 **개인** 또는 **프로젝트** 범위 중 하나를 골라 설치합니다. 두 범위 모두 `curl`로 `SKILL.md`만 받습니다. **이 저장소 전체를 작업 repo에 clone하지 마세요.**

| | 개인 스킬 | 프로젝트 스킬 |
|---|----------|--------------|
| **적용 범위** | 내가 여는 모든 프로젝트 | 현재 repo에서만 |
| **경로** | `~/…/skills/review-merge/` | `<repo-root>/.cursor/skills/review-merge/` 등 |
| **Git 영향** | 작업 repo에 파일 추가 없음 | repo에 스킬 파일 commit 가능 (팀 공유) |
| **언제 쓰나** | 혼자 모든 프로젝트에서 쓸 때 | 팀 repo에 스킬을 고정·공유할 때 |

| 도구 | 개인 경로 | 프로젝트 경로 |
|------|-----------|---------------|
| Cursor | `~/.cursor/skills/review-merge/` | `.cursor/skills/review-merge/` |
| Claude Code | `~/.claude/skills/review-merge/` | `.claude/skills/review-merge/` |
| Codex | `~/.agents/skills/review-merge/` | `.agents/skills/review-merge/` |

#### 개인 스킬 (권장 — Git repo 무변경)

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

#### 프로젝트 스킬 (프로젝트 경로에 설치)

AI 스킬 설정을 repo에 포함·공유하려는 경우에만 사용하세요.

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

필요한 도구(Cursor / Claude Code / Codex)만 골라 실행하면 됩니다.

#### 설치 후

- **Cursor**: **Reload Window** 한 번
- **Claude Code**: 스킬 수정은 세션 중 live 반영; 세션 시작 후 새 top-level `.claude/skills/`는 재시작 필요할 수 있음
- **Codex**: 스킬이 안 보이면 Codex 재시작

### 사용 방법

검토할 PR 스택/집합을 알려주고 "한 건씩 게이트 통과시키고 사람이 머지하는 흐름으로 리뷰하자"라고 요청하면 Agent가 스킬을 자동으로 적용합니다.

| 도구 | 수동 호출 |
|------|-----------|
| Cursor | `/review-merge` |
| Claude Code | `/review-merge` |
| Codex | `/skills` 또는 `$review-merge` |

**적용되는 요청 예시:**

- "이 PR 스택을 아래부터 한 건씩 리뷰하고, 검증 게이트 통과한 것만 내가 머지할게"
- "각 PR이 독립이라 출고 전에 따로 게이트해야 해. 한 건씩 봐줘"
- "보안 PR이라 토큰 해시·세션 쿠키·인가 매트릭스를 코드 근거로 확인해줘"
- "스택 동기화한다고 브랜치 다 push하지 마. 바꾼 것만 push하고 나머진 머지 때 reconcile"

---

## OUTPUT: 무엇이 남는가

대화가 아니라 **남는 것**이 산출물입니다.

| 산출물 | 내용 |
|--------|------|
| **PR별 리뷰 레코드** | 오리엔테이션·코드 근거 응답·런타임 확인·PR 본문 최신화가 PR 단위로 남음 |
| **게이트 통과 이력** | 머지된 각 건이 typecheck/lint/test/build(+DB 통합) 게이트를 통과한 상태로 출고 |
| **정합된 로컬·원격 상태** | 머지 후 main FF·로컬 브랜치 정리·다음 PR base 재타게팅 |
| **하네스 반영** | no-cascade·CLI 권위 게이트·자식 base 재타게팅 불변식을 프로젝트 하네스(`CLAUDE.md` / `AGENTS.md` / `.cursor/rules`)에 고정 |

---

## Workflow

| 단계 | 내용 |
|------|------|
| 3. PR 한 건 리뷰 사이클 | 오리엔테이션 → 대화형 리뷰(코드 근거) → 수정(owning 브랜치만) → PR 본문 최신화 → 사람이 원격 머지 |
| 4. 검증 게이트 | 툴체인 정규화 → 생성물 재생성 → 정적·단위 → 빌드 → DB 통합 (CLI 권위) |
| 5. 머지 후 로컬 정리 | fetch --prune → main FF → 머지 브랜치 삭제 → 다음 PR base를 main으로 재타게팅 |
| 6. No-Cascade | 변경 브랜치 하나만 push, 나머진 머지 시 reconcile, 하네스에 고정 |
| 7. gotcha | auto-delete-head 복구·draft ready·git push 자격·stale 생성물 |

---

## 스킬 구성

```
.cursor/skills/review-merge/SKILL.md   # Cursor용
.claude/skills/review-merge/SKILL.md   # Claude Code용
.agents/skills/review-merge/SKILL.md   # Codex용
```

| 섹션 | 내용 |
|------|------|
| 역할 | REVIEW → MERGE — 한 건씩 게이트 통과 후 사람이 머지 |
| 모드 선택 | merge-review와의 모드 선택 표 |
| 멘탈 모델 | 토폴로지·역할 분담·스택 불변식 |
| PR 한 건 리뷰 사이클 | 오리엔테이션 → 대화형 리뷰 → 수정 → 본문 최신화 → 원격 머지 |
| 검증 게이트 | CLI 권위 + 게이트 gotcha |
| 머지 후 정리 / No-Cascade | 로컬 정리·base 재타게팅, 일괄 push 금지 |
| 알려진 gotcha | auto-delete-head·draft·push 자격·stale 생성물 |
| 도메인별 체크 / 체크리스트 / 환경 적응 메모 | 보안·렌더·임베드 표면, PR/머지 후 체크, repo별 바인딩 |

---

## 이런 분께 추천합니다

- PR이 **독립이거나 출고 전 각 건을 게이트**해야 하는 분
- 에디터 LSP가 아니라 **CLI를 권위 기준**으로 머지 여부를 판단하고 싶은 분
- AI에게 리뷰·수정은 맡기되 **머지 결정은 사람이** 쥐고 싶은 분
- 스택 동기화로 **preview 배포가 폭증**하는 걸 no-cascade로 막고 싶은 분
- `auto-delete-head`·draft·push 자격 같은 함정을 **절차로** 다루고 싶은 분
- 묶음을 먼저 통합하는 [merge-review](https://github.com/wild-mental/merge-review-skill)와 **상황에 따라 골라 쓰고** 싶은 분

---

## 참고

- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)
- 자매 스킬: [merge-review](https://github.com/wild-mental/merge-review-skill) (MERGE → REVIEW)

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

## 라이선스

[MIT License](LICENSE)

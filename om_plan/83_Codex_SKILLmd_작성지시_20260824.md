# Codex 지시 — om-plan·om-verify SKILL.md 작성 (스킬 포장 통일)

작성일: 2026-08-24. 배경: 목표는 4개 명령 각각을 스킬로 만드는 것. 현재 clean-export에 SKILL.md는 om-apply 하나뿐(plan은 스킬 형식 도입 전 제작, verify는 v1 범위 밖). 사람 결정: 세 명령을 같은 스킬 형식으로 통일 후 MR !2에 합류.

## 안전 경계

- 대상: clean-export 작업트리(브랜치 `codex/om-plan-verified-gates-20260820`, HEAD `82be68e733`).
- **엔진 코드 수정 금지**(plancore/applycore/verifycore/CLI 불변). 스킬 문서 + wiring 테스트만.
- 커밋·push·MR 금지(작업트리만 — 검증·커밋·push는 Claude가 대행).

## 작업 1: `.claude/skills/om-plan/SKILL.md`

- 형식: om-apply SKILL.md 패턴(frontmatter name/description → Non-negotiable boundary → Procedure, 실제 CLI 명령 인용).
- 반드시 담을 경계(현행 코드·문서에서 근거 확인 후 작성):
  - 4모드(initial/feature/change/upgrade)와 각 모드의 결과는 **제안일 뿐** — 승인은 사람(intent_review)·검사기(plan check)의 몫.
  - 게이트 우회 금지: 등록자료·관리파일을 계획 단계에서 수정하지 않는다. exit code(0/1/2/3)를 재해석하지 않는다.
  - upgrade 모드의 공식문서 2차 독해는 LLM 교차확인일 뿐, 결정적 진실은 검사기.
  - 기존 훅(run_om_plan_hook)·에이전트(om-plan-official-doc-reviewer)와의 관계 명시(대체 아님, 병행).
- 실제 CLI: `plan start`·`plan check`·`plan-session-start`·`plan-preflight`·`plan-validate`·`plan-resume` 중 스킬 사용자 여정에 필요한 것만, 코드에서 인자 실측 후 기재.

## 작업 2: `.claude/skills/om-verify/SKILL.md`

- 반드시 담을 경계:
  - verify는 **코드·관리파일을 절대 바꾸지 않는다** — 실행·판정·기록만.
  - 최종 상태는 verified(0)/failed(1)/infra_error(3)뿐이며 **실행불가·skip·부분실행·WARN은 절대 통과 아님**.
  - 인계(apply-result 전체)와 build receipt(local-issued+source_clean) 없이는 시작 불가. receipt는 읽기전용·재사용 금지(run-dir 새로 생성).
  - retries=0. 테스트 목록은 등록 계약에서 재계산되며 인계값과 다르면 중단.
  - trust_limitation(digest는 무결성이지 발행자 진위 아님)을 사람 보고에 그대로 전달.
  - UI 부품(test-agent)은 표시 일치 증거로 한정, meta.run_id 계약 충족 시에만 결속.
- 실제 CLI: `verify run REQUEST --run-dir NEW_RUN_DIR` (인자 실측).

## 작업 3: wiring 테스트 보강

- `harness/tests/test_claude_wiring.py`에 apply와 동일 방식으로: 두 SKILL.md 존재 + 핵심 경계 문구 존재 단언(예: verify의 "never" 계열 문구, plan의 제안-승인 분리 문구). 전체 테스트 green 유지.

## 참고 기준

- 실물 패턴: `.claude/skills/om-apply/SKILL.md`.
- 저작 표준 문서 2종(01 공통 작성규칙·02 Claude Code 전용규칙, 2026-08-19)은 현재 디스크에서 못 찾음 — 사용자가 다시 제공하면 그 규칙(필요 규칙만 근거 있게 선택, 자기완결)을 우선 적용하고, 없으면 om-apply 패턴 준수로 갈음.
- 문장별 근거: 코드·정본 문서(61·76 등)를 재열람해 확인한 것만 기재. 추측 문구 금지.

## 결과

`skill_develop/om_plan/84_Codex_SKILLmd_작성결과_20260824.md`: 작성 파일 목록·각 경계 문구의 근거(파일:줄)·테스트 결과. 이후 Claude가 검증 → 커밋 → MR !2 push.

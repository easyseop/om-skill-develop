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

---

## 부록 A (보강 2026-08-24): I/O 양식·호출 체크리스트 — 구체 지시

> 사용자 지적으로 보강: SKILL.md는 사용설명서이므로 아래를 **막연히 쓰지 말고 그대로 반영**할 것.

### A-1. I/O 양식의 정본 = 스키마 파일 (전부 이미 저장소에 있음)

SKILL.md에서 언급하는 **모든 입출력 파일은 해당 스키마 경로를 명시**하고, 필드 목록은 스키마에서 실측해 요약 기재한다(추측 금지):

| 명령 | 입력(사람/LLM 작성) | 스키마 | 주요 산출 | 스키마 |
|---|---|---|---|---|
| plan | `run-request.yaml` | `plancore/schema/plan-run-request.schema.json` | `input-lock.yaml`·`discovered-facts.json`·`proposal/`·`validation-result.json` | plancore/schema/ 나머지 4종 |
| apply | `apply-request.yaml` (실물 필드: schema_version·run_id·plan_run_dir·expected_plan_digest·repositories.product/checker·registration_path·start_ref) | `applycore/schema/apply-request.schema.json` | `apply-context.yaml`(실행계획)·`execution-report.yaml`(LLM 기록)·`apply-result.json` | applycore/schema/ 3종 |
| verify | `verify-request.json` (실물 필드: apply_result_path·build_receipt_path·run_id·retries(항상 0)·prior_infra_error_count·runtime{base_url·container_id·expected_compose_project·expected_compose_config_paths·expected_volume_names·fixture_digest·fixture_evidence_path·mode}·test_environment_names·timeout_seconds) | `verifycore/schema/verify-request.schema.json` | `verify-receipt.json`(canonical_payload+receipt_digest)·`pytest/` 증거 | build-receipt·fixture-receipt·ui-component·waiver 스키마 4종 |

### A-2. 각 호출은 4단 체크 형식으로 기술 (Procedure 강제 양식)

모든 CLI 호출 단계는 **[호출 전 확인 → 호출 → 성공 판정 → 실패 시 행동]** 4단으로 쓴다. 최소 포함:

- **plan check**: 전=run-request가 스키마 유효·모드별 필수 입력 존재. 후=exit 0/1/2/3 의미(2=사람 검토 준비 상태이며 CI에선 성공 취급[Q9], **재해석 금지**), validation-result의 게이트별 판정 읽는 법. 실패 시=제안 수정 후 재검증이지 게이트 우회 아님.
- **apply start**: 전=승인된 plan run 실재·expected_plan_digest 일치·start_ref 40자 SHA. 후=apply-context.yaml의 DAG·management_files 확인. 실패 시=계획 문제면 재계획(수기 보정 금지).
- **apply check**: 전=모든 unit의 start/end SHA가 execution-report에 기록됨. 후=exit 0이면 `static_consistent_awaiting_verify`(**유일 성공 상태**)·verify_handoff를 **그대로** verify에 전달, exit 1=STOP(scope_variances 사유 작성→사람 승인 대기). 실패 시=매니페스트 확장으로 통과시키기 절대 금지.
- **verify run**: 전=**새** run-dir(재사용은 VERIFY_RUN_ALREADY_EXISTS로 거부됨)·build receipt(local-issued+source_clean)·fixture receipt·서버 기동 상태(container_id 실측). 후=status 3상태만 존재(verified 0/failed 1/infra_error 3), **skip·부분실행·WARN은 어느 것도 통과 아님**, receipt의 gates[].reason_codes로 원인 확인, trust_limitation 문구를 사람 보고에 그대로 포함. 실패 시=infra_error 반복이면 3회째 escalation(사람), receipt·산출물 수정 금지.

### A-3. 케이스별 지시 (각 SKILL.md에 표 또는 절로)

- **plan**: 4모드별로 [필수 입력 차이·필수 산출물 차이·대표 실패 게이트]를 표로. upgrade는 3층 검증·path-remap 필수 산출을 명시.
- **apply**: 정상 / 민감경로 이탈(즉시 STOP) / 일반 이탈(scope_variances 4요소 사유 작성→사람) / 병합 실패(3단 복구 순서) — 각각 "LLM이 해도 되는 일·절대 안 되는 일"을 분리 기술.
- **verify**: verified / failed / infra_error 각각에서 다음 행동(사람에게 무엇을 보고하고 무엇을 기다리는지). 대표 거부 사례 4종(다른 서버·등록자료 불일치·전부 skip·run-dir 재사용)은 "이건 고장이 아니라 차단이 작동한 것"임을 명시.

### A-4. 근거 규율

- 위 실물 필드명은 리허설 산출물에서 실측한 것이나, **최종 기재는 반드시 스키마·코드 재열람으로 재확인**(스키마와 이 부록이 다르면 스키마가 정본).
- 결과서 84에 각 SKILL.md 문장 → 근거(스키마/코드 파일:줄) 대응표 포함.

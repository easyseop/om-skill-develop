# A′ 명세 + Codex 지시 — change 테스트 누락 방지 (계획 단계, 검사기만)

작성일: 2026-08-20. 작성자: Claude(검증자·기록자). 구현: Codex.
근거 결정: `24_누락감사_사람결정과_기록_20260820.md` "확정된 결정 → A-3" 및 "★ 재검토 대상 R-1·R-2".

> ★ **GitLab 반영 보류 중**: 현재 apply·GitLab push·MR은 전면 보류다(`43_Codex_검증완료분_반영_지시서` 참조). 이유: (1) 실제 GitLab 파이프라인·4모드 end-to-end 미검증, (2) Q9(exit2 성공처리) GitLab 미반영. 따라서 이 A′ 구현도 **push·MR 금지, 로컬 작업 트리 변경만**. 반영은 나중에 Q9 반영 + 파이프라인 초록불 + 사람 승인 후 몰아서 한다.

## 0. 한 줄 정의 / 범위

**A′ = change 모드에서 "그 커스텀에 등록된 테스트가 제안서 실행목록에 전부 있는지"만 계획 단계에서 강제(없으면 block). 그 외 판정 없음.**

- **하는 것**: 요청 `customization_id`(=X)의 등록 테스트(`customization-relations[X].tests`)가 제안서 실행목록에 **전부 포함**되는지 검사. 하나라도 빠지면 block.
- **하지 않는 것(중요, v2 붕괴 교훈)**:
  - **동작변경 자동판정 안 함** — doc 45의 관측분기(O_req/O_surface/O_outside, 규칙1·2)는 **구현하지 않는다**(비교기준이 운영자 저작이라 못 믿음).
  - **기준선 잠금 안 함** — custom_baseline==릴리즈잠금 강제는 A(별개, apply 개방 후. R-1·R-2). 여기서 손대지 마라.
  - **테스트 실제 실행·통과 안 봄** — 실행은 verify(downstream). 계획 단계는 "목록에 있나"까지.
  - `change_kind` 자기신고 필드 도입 금지.

## 1. 대상 저장소 / 위치

clean-export `work/kb-datacatalog-upgrade-checker-om-plan-cli`.
- 검사: `harness/acgh/integrations/om/collectors.py::OmAdapter.validate_proposal`(L539~). change 등록여부 검사(L556-557) 근처에 규칙 추가.
- 소비 fact: `customization-relations`(A-4, 이미 존재 — L493-495에서 노출, per-item에 `tests`). `fact_values`로 읽음(L546-549 방식).

## 2. 입력 (이미 있는 것)

- `fact_values["customization-relations"]` = ID로 정렬된 리스트. 각 항목 `{customization_id, changed_paths, required_changed_paths, contracts, tests}`. **`tests` = 그 ID 한정 검사기 selector**(그 ID contracts→required_tests ∪ 매니페스트 assurance.direct_tests). A-4에서 검증 완료.
- 요청 `request["customization_id"]` = X.
- 제안서 실행목록: `validate_proposal`이 이미 모으는 `required_test_claims`(각 document의 `decisions[].required_tests`, L586) + (있으면) 제안서 `contracts-to-run`류 필드. **정확한 제안서 필드명은 Codex가 `references/proposal-format*.md`·`example-change-proposal.md`와 대조해 정본 하나로 확정**하고 결과서에 명시한다. 실행목록 집합 = 그 필드가 담은 test selector들의 합집합.

## 3. 규칙 (검사기 결정론 — validate_proposal에 추가)

`mode == "change"`일 때만 진입(다른 모드는 진입 금지 — 가드).

1. `relations = fact_values.get("customization-relations")`에서 `customization_id == X`인 항목을 찾는다.
   - **없거나 fact 자체가 없으면 fail-closed**: `issues.append(...)`로 block(예: "change target has no relation facts: X"). 조용히 통과 금지.
2. `required = set(그 항목의 tests)`. `run_list = set(제안서 실행목록 selector들)`.
3. `missing = required - run_list`. **`missing`이 비어있지 않으면 block**: `issues.append(f"registered tests for {X} missing from run list: {sorted(missing)}")`.
4. `required`가 빈 집합(`[]`)이면 규칙 통과(요구할 게 없음). **크래시·거짓 block 금지.**

## 4. 정직한 한계 (결과서에 명시할 것)

- 이 규칙은 **테스트가 실제로 실행·통과했는지 안 본다**(verify 몫).
- `tests` 목록 자체는 매니페스트(운영자 저작) 파생이다 → **등록 시점에 테스트를 적게 등록하면** 이 규칙은 못 잡는다(등록 완전성 문제, B안과 동일한 잔여 위험). A′는 "이미 등록된 걸 빼먹는 것"만 막는다.
- 동작변경 여부는 판정하지 않는다 — 그건 verify+사람.

## 5. 수용 기준 (Claude가 반례로 독립 검증)

| # | 정상(통과) | 반례(block/일치) |
|---|---|---|
| 1 | change 제안서가 X의 등록 테스트를 실행목록에 **전부** 실음 → 통과 | 하나라도 누락 → block |
| 2 | **X 것만 요구** — `customization-relations[X].tests`만 사용(전역 `registered-tests` union 아님) | ID-B 테스트를 X에 요구하면 실패(거짓 block) |
| 3 | X의 tests가 `[]` → 규칙 통과, 크래시 없음 | 빈 목록에서 block/크래시 시 실패 |
| 4 | `customization-relations` fact 부재/‌X 항목 부재 → **fail-closed block** | 조용히 통과 시 실패 |
| 5 | 다른 모드(initial/feature/upgrade)는 이 규칙 **미진입** | 다른 모드에서 규칙 발동 시 실패 |
| 6 | 기존 change 검사(등록여부·owner·shared·required_test_claims)·기존 테스트 **회귀 0** | 회귀 시 실패 |
| 7 | pre_plan/post_change_reconcile 적용 범위가 결과서에 명시되고 그대로 동작 | 명시 없이 임의 적용 시 실패 |

> #7 주의: pre_plan(변경 전 계획)에도 실행목록이 있는지 Codex가 확인해 **적용 범위를 명시**하라. 실행목록 개념이 없으면 post_change_reconcile 한정으로 두고 그 사실을 결과서에 적는다(임의 확대 금지).

## 6. 금지사항

- doc 45의 관측분기(규칙1·2)·기준선 잠금 구현 금지(A, 별개·미래).
- 기존 flat fact·기존 검증규칙·apply·제품코드·활성 등록자료·`/om-plan` SKILL·references 수정 금지.
- **commit·push·MR 금지**(GitLab 반영 보류 중 — 상단 ★ 참조). 로컬 작업 트리 변경만. 확인 못 한 것 단정 금지 — 불확실하면 결과서에 질문으로.

## 7. 결과

`skill_develop/om_plan/47_Codex_Aprime_구현_결과_20260820.md`: 반영 위치(파일:줄)·확정한 제안서 실행목록 필드명과 근거·적용 범위(pre_plan 포함 여부)·수용기준별 반례 테스트 결과·남은 위험. 구현 후 Claude가 반례로 독립 검증한다.

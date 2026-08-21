# A′ change 테스트 누락 방지 구현 결과

작성일: 2026-08-20  
구현 대상: `work/kb-datacatalog-upgrade-checker-om-plan-cli`  
작업 브랜치: `codex/om-plan-verified-gates-20260820`

현재 A′ 변경은 기존 HEAD `df970075f4` 위의 미커밋 작업 트리 변경이다. 앞선 43 작업의 로컬 커밋과 A′ 변경을 같은 상태로 오해하면 안 된다.

## 1. 결과

A′를 지시 범위대로 구현했다.

- `change` 모드에서만 대상 커스터마이징 ID의 등록 테스트를 확인한다.
- 제안서 실행목록에서 등록 테스트가 하나라도 빠지면 `block` 사유를 추가한다.
- 대상 ID의 테스트만 요구한다. 전역 `registered-tests` 합집합을 요구하지 않는다.
- 대상 관계 fact가 없거나 대상 ID 항목이 없으면 fail-closed로 `block`한다.
- 테스트 목록이 빈 ID는 추가 block 없이 통과한다.
- 테스트 실행·통과, 동작변경 판정, 기준선 잠금은 구현하지 않았다.
- commit·push·MR은 수행하지 않았다.

## 2. 반영 위치

### 검사 코드

`work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/acgh/integrations/om/collectors.py:605`

- `mode == "change"` 가드 안에서만 A′를 실행한다.
- `customization-relations`에서 요청 `customization_id`와 같은 항목을 찾는다.
- 관계 fact/대상 항목이 없으면 다음 사유를 추가한다.

```text
change target has no relation facts: <CUSTOMIZATION-ID>
```

- `target_relation.tests - proposal_run_list`를 계산하고 누락 시 다음 사유를 추가한다.

```text
registered tests for <CUSTOMIZATION-ID> missing from run list: [<TEST-ID>...]
```

### 테스트 코드

`work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/tests/test_plan_workflow.py:841`

- `pre_plan`·`post_change_reconcile` 양쪽의 정상/누락 반례
- 두 테스트 중 하나만 넣었을 때 정확히 빠진 하나를 block하는 반례
- 다른 ID의 전역 테스트를 대상 ID에 요구하지 않는 반례
- 대상 ID의 테스트가 빈 목록일 때 통과하는 반례
- 관계 fact 자체 또는 대상 항목 부재 시 fail-closed 반례
- `initial`·`feature`·`upgrade`에서 A′가 발동하지 않는 반례
- upgrade용 최상위 `required_tests`를 change 실행목록으로 잘못 인정하지 않는 반례
- 기존 C20 승인 테스트가 새 A′ 입력 요건도 만족하도록 proposal fixture를 보정했다.

## 3. 확정한 제안서 실행목록

정본은 모든 proposal 문서의 다음 필드 합집합이다.

```text
decisions[].required_tests[].id
```

근거:

- `proposal-format.md:56`은 `required_tests`를 decision 내부 필드로 정의한다.
- `example-change-proposal.md`도 `decisions[].required_tests` 구조를 사용한다.
- `change.md:31-33`은 `pre_plan`과 `post_change_reconcile` 모두 재실행할 계약·테스트를 proposal 산출물로 요구한다.
- 기존 검사 코드도 모든 문서의 `decisions[].required_tests`를 `required_test_claims`로 합친다.

기존 하위호환에 따라 문자열 claim도 selector로 읽지만, 문서 정본은 `{id, status, required, result}` 매핑이다.

최상위 `required_tests` 또는 `contracts-to-run`은 change 실행목록으로 인정하지 않았다. 해당 별칭은 upgrade 산출물 수집용이며 change reference에는 정의되어 있지 않다. 이를 잘못 인정하지 않는 반례도 추가했다.

## 4. 적용 범위

A′는 change의 두 경로에 모두 적용한다.

| change 경로 | 적용 | 근거 |
|---|---:|---|
| `pre_plan` | 예 | 변경 전 계획에도 재실행할 테스트 목록이 proposal 산출물로 정의돼 있다. |
| `post_change_reconcile` | 예 | 변경 후 정합 proposal에도 동일한 decision `required_tests` 형식을 사용한다. |

두 경로의 차이는 변경 diff 수집 여부이며, 등록 테스트를 제안서 실행목록에 빠뜨리지 않는 규칙에는 영향을 주지 않는다.

## 5. 수용 기준별 검증

| 기준 | 검증 결과 | 반례 근거 |
|---|---|---|
| 1. 등록 테스트 전부 포함/일부 누락 | 통과 | 두 테스트를 모두 넣으면 `approval`, 하나만 넣으면 빠진 테스트 하나를 명시해 `block`했다. |
| 2. 대상 ID의 테스트만 요구 | 통과 | 전역에는 ID-A 테스트가 있지만 테스트가 빈 ID-B change proposal은 `approval`이었다. |
| 3. 빈 테스트 목록 | 통과 | `customization-relations[ID-B].tests == []`에서 크래시·거짓 block이 없었다. |
| 4. 관계 fact/대상 항목 부재 | 통과 | fact 부재와 빈 relation 목록 모두 `change target has no relation facts`로 block했다. |
| 5. 다른 모드 미진입 | 통과 | `initial`·`feature`·`upgrade`에서 A′ 사유가 생성되지 않았다. |
| 6. 기존 change·전체 회귀 | 통과 | 기존 C20 목적을 유지하도록 fixture를 보정한 뒤 전체 clean-export 테스트가 통과했다. |
| 7. 두 change 경로 적용 범위 | 통과 | 정상/누락 반례를 `pre_plan`과 `post_change_reconcile` 각각 실행했다. |

실행 결과:

```bash
PYTHONPATH=.:harness python3.11 -m pytest -q \
  harness/tests/test_plan_workflow.py -k 'aprime'
# 11 passed

PYTHONPATH=.:harness python3.11 -m pytest -q \
  harness/tests/test_plan_workflow.py \
  harness/tests/test_om_registration_facts.py
# 115 passed

PYTHONPATH=.:harness python3.11 -m pytest -q harness/tests
# 184 passed

git diff --check
# 통과
```

테스트 개수만을 완료 근거로 삼지 않았다. 위 표의 누락·잘못된 ID·fact 부재·모드 오발동 반례가 실제 판정을 뒤집는지 확인했다.

## 6. 변경 범위

검사기 작업 트리에서 수정한 파일은 두 개다.

1. `harness/acgh/integrations/om/collectors.py`
2. `harness/tests/test_plan_workflow.py`

다음 항목은 수정하지 않았다.

- doc 45 관측분기와 `change_kind`
- 기준선 잠금
- 테스트 실제 실행·통과 판정
- 기존 flat fact
- apply·제품 코드·활성 등록자료
- `/om-plan` `SKILL.md`와 references
- commit·push·MR

## 7. 정직한 한계와 남은 질문

### 그대로 남는 한계

- A′는 테스트 selector가 계획 목록에 있는지만 본다. 실제 실행·통과는 downstream verify가 확인해야 한다.
- `customization-relations.tests`는 운영자가 작성한 Manifest·Contract에서 파생된다. 처음부터 테스트를 적게 등록한 경우 A′는 누락을 발견하지 못한다.
- 코드의 동작이 바뀌었는지 판정하지 않는다. 의미 판단은 verify와 사람이 맡는다.
- 기준선이 올바른 릴리스인지 잠그지 않는다. 이는 별도 A 범위다.

### 후속 결정이 필요한 기존 비대칭

`customization-relations.tests`에는 Manifest의 `assurance.direct_tests`가 포함되지만, 기존 flat fact `registered-tests`에는 Contract의 `required_tests`만 포함된다. 따라서 direct-only 테스트를 proposal에서 `status: existing`으로 표시하면 기존 검증이 "미등록"으로 볼 수 있다.

이번 지시서는 기존 flat fact와 기존 검증규칙 변경을 금지하므로 손대지 않았다. 후속 단계에서 다음 중 무엇을 정본으로 할지 결정해야 한다.

1. `registered-tests`에 direct tests도 포함한다.
2. direct test 전용 status 의미를 문서와 검사기에 명시한다.
3. Contract tests만 change 실행목록으로 인정하도록 A-4 관계 fact 정의를 축소한다.

권고는 1번이다. Manifest에 이미 등록된 direct test를 전역 등록 목록에서도 같은 의미로 취급해야 제안서 작성자와 기존 검증기의 해석이 일치한다.

## 8. 다음 단계

Claude가 이 문서의 수용 기준 1~7을 독립 반례로 재검증한다. 재검증 결과와 위 direct-test 정책 결정이 나오기 전에는 A′ 변경을 commit·push·MR하지 않는다.

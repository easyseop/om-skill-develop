# direct_tests 관계 fact 제거 결과

작성일: 2026-08-21  
구현 대상: `work/kb-datacatalog-upgrade-checker-om-plan-cli`  
작업 브랜치: `codex/om-plan-verified-gates-20260820`  
선행 변경: A′ 테스트 누락 방지 미커밋 변경 및 `47_Codex_Aprime_구현_결과_20260820.md`

## 1. 결과

사람 결정 D-directtest 옵션 3을 반영했다.

- `customization-relations[].tests`는 해당 ID가 등록한 Contract의 `required_tests`만 담는다.
- Manifest의 `assurance.direct_tests`는 관계 fact 수집에서 읽지 않는다.
- 기존 전역 flat fact `registered-tests`와 per-ID 관계 `tests`가 모두 Contract 테스트를 뜻하게 됐다.
- 합성 Manifest와 실제 등록자료의 `assurance.direct_tests` 필드는 삭제하지 않았다.
- A′는 변경 후 관계 fact를 소비해 Contract 테스트 누락을 계속 block한다.
- commit·push·MR은 수행하지 않았다.

현재 변경은 기존 HEAD `df970075f4` 위의 미커밋 작업 트리 변경이다. A′ 변경과 이번 direct-test 제거 변경이 함께 존재하지만, 이번 작업에서 새 commit은 만들지 않았다.

## 2. 반영 위치

### 관계 fact 생성

`work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/acgh/integrations/om/collectors.py`

- `107-112`: 관계 조립용 `parts`에서 사용하지 않는 `direct_tests` 집합을 제거했다.
- `146-152`: Manifest의 `assurance.contracts` 읽기는 유지하고, `assurance.direct_tests` 읽기만 제거했다.
- `156-169`: 관계 `tests`를 빈 집합에서 시작해 등록된 Contract의 `required_tests`만 합친다.

다음 로직은 변경하지 않았다.

- Contract catalog 수집
- `registered-tests`
- `id-contract-consistency`
- Manifest `assurance.contracts` 교차검증
- 기존 flat fact

### A-4 테스트 개정

`work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/tests/test_om_registration_facts.py`

- 합성 Manifest에는 `tests/direct_a.py::test_guard`를 계속 넣어 둔다.
- 관계 `tests` 예상값에서는 해당 direct test를 제거했다.
- direct test selector가 어떤 관계 `tests`에도 나타나지 않는다고 명시적으로 단언한다.
- 모든 per-ID 관계 `tests`가 전역 `registered-tests`의 부분집합인지 단언한다.
- 입력 순서를 뒤집어도 digest가 같은 기존 안정성 테스트는 유지했다.

## 3. A′ 상호작용

A′의 실행목록 정본은 그대로 `decisions[].required_tests[].id`다. 이번 변경은 A′ 검사 코드를 수정하지 않고, A′가 소비하는 `customization-relations[].tests`의 의미만 Contract 테스트로 단일화했다.

검증 결과:

- 대상 ID의 Contract 테스트를 모두 넣으면 `approval`
- Contract 테스트 하나를 빠뜨리면 정확한 selector를 표시해 `block`
- 다른 ID의 Contract 테스트는 요구하지 않음
- 대상 관계 fact가 없으면 fail-closed `block`
- 빈 Contract 테스트 목록은 추가 block 없이 통과
- `pre_plan`·`post_change_reconcile` 모두 동일하게 동작

## 4. 수용 기준별 반례 결과

| 기준 | 결과 | 확인 내용 |
|---|---|---|
| 1. 관계 tests = per-ID Contract required_tests | 통과 | ID-A와 ID-B의 관계 `tests`가 각 Contract의 `required_tests`와 정확히 일치했다. |
| 2. 합성 direct_tests 미노출 | 통과 | Manifest에 비어 있지 않은 direct test를 넣어도 관계 `tests`에는 나타나지 않았다. |
| 3. 전역/관계 의미 일치 | 통과 | 모든 관계 `tests`가 전역 `registered-tests`의 부분집합임을 단언했다. |
| 4. A′ 반례 유지 | 통과 | 누락 block, 대상 ID 한정, 빈 목록, fact 부재, 다른 모드 미진입 반례 11개가 통과했다. |
| 5. A-4·전체 회귀 | 통과 | A-4 테스트 8개와 clean-export 전체 테스트 184개가 통과했다. |
| 6. Manifest 필드 유지 | 통과 | 실제 1.13.1 Manifest 7개에 `direct_tests: []`가 그대로 있고 등록자료 diff는 없다. |

실행 명령과 결과:

```bash
PYTHONPATH=.:harness python3.11 -m pytest -q \
  harness/tests/test_om_registration_facts.py
# 8 passed

PYTHONPATH=.:harness python3.11 -m pytest -q \
  harness/tests/test_plan_workflow.py -k 'aprime'
# 11 passed

PYTHONPATH=.:harness python3.11 -m pytest -q harness/tests
# 184 passed

git diff --check
# 통과
```

테스트 개수뿐 아니라 합성 direct test가 실제로 관계 fact에서 사라지는지, 관계 테스트가 전역 Contract 테스트 집합을 벗어나지 않는지, A′ 누락 반례가 판정을 뒤집는지를 확인했다.

## 5. 변경 범위

이번 후속으로 새로 수정한 대상은 다음 두 파일이다.

1. `harness/acgh/integrations/om/collectors.py`
2. `harness/tests/test_om_registration_facts.py`

작업 트리에는 선행 A′ 테스트 파일 `harness/tests/test_plan_workflow.py` 변경도 함께 남아 있다. 이번 후속은 그 파일을 추가로 수정하지 않았다.

다음 항목은 수정하지 않았다.

- Manifest와 활성 등록자료
- Contract와 `registered-tests`
- `id-contract-consistency`
- apply와 제품 코드
- `/om-plan` `SKILL.md`와 references
- commit·push·MR

## 6. 남은 死필드와 위험

Manifest 스키마와 실제 Manifest의 `assurance.direct_tests`는 이제 관계 fact 생성에서 아무도 읽지 않는 死필드로 남았다. 이번 범위는 등록자료와 스키마 수정을 금지하므로 필드를 삭제하지 않았다.

후속 R-3에서 다음을 별도로 결정해야 한다.

1. Manifest 스키마에서 `assurance.direct_tests`를 제거할지
2. 기존 Manifest 7개의 빈 필드를 함께 정리할지
3. 이전 버전 등록자료를 읽을 때 해당 필드를 무시하는 호환 정책을 명시할지

현재 동작상 위험은 direct test가 Contract에 등록되지 않으면 A-4 관계 fact와 A′ 검사 대상에서 완전히 제외된다는 점이다. 이는 이번 사람 결정의 의도다. 기능 검증이 필요하면 반드시 Contract의 `required_tests`에 등록해야 한다.

## 7. 다음 단계

Claude가 수용 기준 1~6을 독립 반례로 재검증한다. GitLab 파이프라인·4모드 end-to-end와 Q9 반영이 확인되고 사람이 승인하기 전에는 현재 작업 트리를 commit·push·MR하지 않는다.


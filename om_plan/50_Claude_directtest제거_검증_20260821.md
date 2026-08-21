# Claude 독립 검증 — direct_tests 제거(옵션3) + A′

검증일: 2026-08-21. 대상: `work/kb-datacatalog-upgrade-checker-om-plan-cli` 브랜치 `codex/om-plan-verified-gates-20260820`(미커밋). 근거 결과서: `47`(A′), `49`(direct_tests 제거).

## 판정: 통과 (P0/P1 없음)

"테스트 N개 통과"가 아니라 **반례가 판정을 뒤집는지**로 검증.

### 1. 코드 실측 (collectors.py)
- `parts`에서 `direct_tests` 집합 제거 확인, `assurance.get("direct_tests")` 읽기 삭제 확인, `assurance.get("contracts")` 유지 확인.
- 관계 `tests = set()`에서 시작해 **등록 계약 required_tests만** union(약 L156-169).
- collectors.py에 `direct_test` 문자열 잔재 **0건**.

### 2. 내가 직접 실행
- `pytest harness/tests` = **184 passed**(내 실행). `test_om_registration_facts.py`·A′(`-k aprime`) 포함.

### 3. 실데이터 독립 계산
- 등록 7개 ID 각각 `entry.contracts → required_tests`를 내가 별도 계산 → 관계 fact의 `tests`와 정확히 일치. 전 매니페스트 `direct_tests=[]`.

### 4. 결정적 주입 반례 (빈 데이터 착시 배제)
- 등록자료를 temp로 복사 후 **BANK-OM-001 매니페스트에 `direct_tests: ["tests/FAKE_direct.py::test_injected"]` 주입** → 실제 `_registration_metadata` 실행.
- 결과: `BANK-OM-001.tests = [계약 테스트 1개]`, **주입한 FAKE 미포함**(leaked=False). `registered-tests` flat(9개)에도 FAKE 없음.
- → direct_tests 무시가 **코드로 실증**(데이터가 비어서가 아님).

### 5. §7 해소 확인
- 관계 `tests`·`registered-tests` **둘 다 계약 테스트만** → direct-only 비대칭 원천 소멸. A′는 계약 테스트 기준으로 누락 block 유지(주입 후에도 A′ 동작 불변).

## 잔여(위험 아님, 기록)
- 매니페스트 `assurance.direct_tests: []` = 死필드로 잔존(등록자료·스키마 미변경 원칙). 정리는 R-3(스키마·등록자료 몰아서 할 때).

## 상태
- A′(doc 46/47) + direct_tests 제거(doc 48/49) 모두 검증 통과. 같은 브랜치 미커밋. **push·MR 안 함**(GitLab 보류: 파이프라인·4모드 미검증 + Q9 미반영).

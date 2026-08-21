# Codex 지시 — direct_tests를 관계 fact의 tests에서 제거 (D-directtest 옵션3)

작성일: 2026-08-20. 작성자: Claude(검증자·기록자). 구현: Codex.
근거 결정: `24_누락감사_사람결정과_기록_20260820.md` "신규 사람 결정 → D-directtest" = **옵션(3) 확정(사람 결정)**. 관련 명세: `40_A4_facts_ID관계보존_명세` §2.1·§6-#4(이 결정으로 개정됨).

> ★ **GitLab 반영 보류 중**(`43_...` 참조): commit·push·MR 금지. 로컬 작업 트리 변경만. A′(doc 46)와 같은 브랜치 `codex/om-plan-verified-gates-20260820`의 미커밋 작업에 얹는다.

## 0. 결정 배경 (왜 빼나)

- 이 시스템의 테스트 등록 정본은 **Contract**(invariant + required_tests)다. 매니페스트 `assurance.direct_tests`는 계약을 우회해 테스트만 나열하는 통로인데 **1.13.1 전 매니페스트에서 `[]`(빈 값), 아무도 안 쓴다**(Claude 데이터 확인).
- A-4가 이 빈 통로를 관계 fact의 `tests`에 union하면서, 기존 flat fact `registered-tests`(계약 테스트만)와 **비대칭(§7 D-directtest)**이 생겼다.
- **사람 결정: 통로를 하나로.** A-4가 `direct_tests`를 `tests`에 넣던 것을 **제거**한다 → 관계 `tests` = 그 ID의 **계약 required_tests만**. 그러면 `registered-tests`(전역 계약 테스트)와 의미가 일치해 §7이 원천 소멸.

## 1. 대상 / 위치

clean-export `work/kb-datacatalog-upgrade-checker-om-plan-cli`.
- `harness/acgh/integrations/om/collectors.py::_registration_metadata`
  - `parts` 초기화의 `"direct_tests": set()` (약 L112)
  - `parts["direct_tests"].update(_sorted_strings(assurance.get("direct_tests")))` (약 L153-155)
  - 관계 tests 조립: `tests = set(parts["direct_tests"])` (약 L164) → contract union 시작점
- 테스트: `harness/tests/test_om_registration_facts.py` (direct_tests 합성·포함 단언: L71·L82·L93·L142 부근)

## 2. 할 일

1. **collectors.py**: 관계 `tests`가 **contract required_tests만** 담게 한다.
   - `tests = set(parts["direct_tests"])` → `tests: set[str] = set()`로 변경(계약만 union).
   - 더 이상 안 쓰는 `parts["direct_tests"]` 초기화·`assurance.get("direct_tests")` 읽기 **제거**(死코드 방지). `assurance.get("contracts")` 읽기(manifest_contracts_by_id)는 **유지**(정합 크로스체크에 필요).
2. **매니페스트·활성 등록자료는 건드리지 마라.** `assurance.direct_tests: []` 필드는 매니페스트에 **그대로 둔다**(무시되는 빈 필드가 됨). 등록자료 수정 금지 원칙 유지. (그 死필드 자체의 스키마 제거는 별도·나중 — 아래 §5 기록.)
3. **A-4 테스트 개정**: `direct_tests`가 `tests`에 **포함되지 않음**을 검증하도록 뒤집는다.
   - 합성 `direct_tests: ["tests/direct_a.py::test_guard"]`를 넣은 케이스는, 그 selector가 관계 `tests`에 **나타나지 않음**을 단언(기존엔 나타남을 단언 = A-4 수용#4). 
   - digest 안정성(경로 reverse) 테스트는 유지하되 direct_tests 의존 부분만 조정.
4. **A′(doc 46)과의 상호작용 회귀 확인**: A′는 관계 `tests`를 소비한다. 이제 그 값이 계약 테스트만이므로, A′ 반례(누락→block 등)가 **여전히 계약 테스트 기준으로** 동작하는지 재실행. change 제안서가 계약 테스트만 실행목록에 요구받으면 정상.

## 3. 하지 말 것

- 계약 경로(①)·`registered-tests`·`id-contract-consistency`·기존 flat fact **로직 변경 금지**(이번은 관계 `tests`에서 direct_tests만 뺀다).
- 매니페스트/활성 등록자료 수정 금지. `/om-plan` SKILL·references·apply·제품코드 수정 금지.
- commit·push·MR 금지. 확인 못 한 것 단정 금지 — 불확실하면 결과서에 질문.

## 4. 수용 기준 (Claude가 반례로 검증)

| # | 정상 | 반례(실패로 잡혀야) |
|---|---|---|
| 1 | 관계 `tests`가 그 ID **계약 required_tests와 정확히 일치** | direct_tests selector가 tests에 남으면 실패 |
| 2 | 매니페스트에 합성 `direct_tests` 넣어도 관계 `tests`·A′ 요구에 **안 나타남** | 나타나면 실패 |
| 3 | `registered-tests`(전역 계약 테스트)와 관계 `tests`(per-ID 계약 테스트)가 **의미 일치**(전역 ⊇ per-ID) | direct-only가 한쪽에만 남으면 실패 |
| 4 | A′ 반례(누락→block, 다른 ID 테스트 안 요구, fail-closed) **계약 테스트 기준으로 유지** | 회귀 시 실패 |
| 5 | 기존 A-4 다른 수용기준(관계 per-ID 보존·정합 크로스체크·정렬 digest)·전체 회귀 0 | 회귀 시 실패 |
| 6 | 매니페스트 `assurance.direct_tests: []` 필드는 **그대로 존재**(등록자료 미변경) | 매니페스트를 수정하면 실패 |

## 5. 기록 (남은 死필드)

- 매니페스트 스키마의 `assurance.direct_tests`는 이제 **아무도 안 읽는 死필드**로 남는다. 스키마에서 완전 제거할지는 **별도 정리 대상**(등록자료·스키마 변경이라 이번 범위 밖). 결과서에 "死필드로 남김"을 명시. → 결정기록 R-3에 연결.

## 6. 결과

`skill_develop/om_plan/49_Codex_directtest제거_결과_20260820.md`: 반영 위치(파일:줄)·개정한 A-4 테스트·A′ 회귀 결과·수용기준별 반례 결과·남은 위험(死필드 포함). Claude가 반례로 독립 검증한다.

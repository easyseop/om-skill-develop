# Codex 지시 — A-2(a) decision 필드 타입 검사 추가

작성일: 2026-08-21. 작성자: Claude(검증자·기록자). 구현: Codex.
근거: 트랙 A(doc 38) 1항목 "proposal 스키마"의 **실측 재범위**. 조사 결과 형식 핵심(6필드 필수·decision_source enum·evidence_refs·floor·no_change)은 **이미 강제됨**. **유일한 실질 구멍 = decision 필드 값의 타입 미검사.** 이것만 보강하고 트랙 A를 닫는다.

> ★ **GitLab 반영 보류 중**(`43_...`): commit·push·MR 금지. 로컬 작업 트리 변경만. 같은 브랜치 `codex/om-plan-verified-gates-20260820`의 미커밋 작업에 얹는다.

## 0. 배경 (왜 이것만)

- `_validate_required_decision_fields`(validate.py:198~)는 6키 **존재**와 `decision_source` **enum**만 본다. 값이 **타입이 틀려도**(예: `affected_customization_ids`를 리스트 아닌 문자열로) 통과한다.
- 다른 형식 규칙은 이미 강제(존재/enum/refs/floor/no_change). **중복 구현 금지.**

## 1. 대상 / 위치

clean-export `work/kb-datacatalog-upgrade-checker-om-plan-cli`.
- **수정 함수 하나**: `harness/acgh/plancore/validate.py::_validate_required_decision_fields`.
- 이 함수는 각 decision이 dict이고 6키가 있을 때, **아래 타입 규칙을 추가 검사**한다(어기면 issue append = block).

## 2. 타입 규칙 (present일 때만 검사 — 부재는 기존 missing이 이미 처리)

각 `decision`(dict)에서, 해당 키가 **있을 때**:

| 필드 | 요구 타입 | 위반 예 |
|---|---|---|
| `subject` | 비어있지 않은 `str` | 리스트/빈 문자열 |
| `decision` | 비어있지 않은 `str` | 매핑/빈 문자열 |
| `evidence_refs` | `list` | 문자열 하나로 씀 |
| `affected_customization_ids` | `list`이고 **모든 원소가 `str`** | `"BANK-OM-001"`(문자열) |
| `required_follow_up` | `list` **또는** 리터럴 문자열 `"none"` | 매핑/정수 |

- `decision_source`: **손대지 마라**(이미 enum 강제).
- `evidence_refs`의 **원소 형식**(문자열/매핑·포인터)은 **기존 `_validate_refs`가 담당** — 여기선 "list인가"만 본다(중복 금지).
- issue 메시지는 기존 스타일 유지: `f"{path.name}: decisions[{index}].<field> must be <type>"`.
- **present일 때만 검사**: 부재 필드를 여기서 또 지적하지 말 것(기존 `missing` 로직과 이중보고 금지).

## 3. 범위 밖 (하지 말 것)

- `no_change` 블록 타입: **기존 floor가 이미 검사** — 손대지 마라.
- 모드별 필수파일·RDB식 스키마·jsonschema 파일 신설: **현재 제안서 설계(자유형 문서 묶음)와 안 맞음. 하지 마라.**
- decision_source enum 재구현, evidence_ref 원소·포인터 재구현: 중복 금지.
- 기존 flat fact·apply·제품코드·활성 등록자료·`/om-plan` SKILL·references·A′·A-4 로직 수정 금지.
- commit·push·MR 금지.

## 4. 수용 기준 (Claude가 반례로 검증)

| # | 정상(통과) | 반례(block) |
|---|---|---|
| 1 | `affected_customization_ids`가 문자열 리스트 | 문자열 하나(`"BANK-OM-001"`) → block |
| 2 | `subject`/`decision`이 비어있지 않은 문자열 | 빈 문자열·리스트·매핑 → block |
| 3 | `evidence_refs`가 리스트 | 문자열/매핑 하나 → block |
| 4 | `affected_customization_ids` 원소가 전부 문자열 | 원소에 정수/매핑 섞임 → block |
| 5 | `required_follow_up`가 리스트 또는 `"none"` | 매핑/정수 → block |
| 6 | 필드 **부재**는 여기서 추가 지적 없음(기존 missing만) | 부재를 타입검사가 이중 지적하면 실패 |
| 7 | 기존 정상 제안서(4모드 예시·기존 fixture)·전체 회귀 0 | 회귀 시 실패 |
| 8 | decision_source enum·evidence_ref 원소검사 **동작 불변** | 중복/약화 시 실패 |

> #7 필수: 기존 통과하던 실제 제안서/테스트 fixture가 새 타입검사로 **거짓 block되지 않아야** 한다(정상 데이터 전부 통과 재확인).

## 5. 결과

`skill_develop/om_plan/52_Codex_A2_구현_결과_20260821.md`: 반영 위치(파일:줄)·추가 타입규칙·수용기준별 반례 결과·기존 제안서 거짓block 없음 확인·회귀 결과. Claude가 반례로 독립 검증한다.

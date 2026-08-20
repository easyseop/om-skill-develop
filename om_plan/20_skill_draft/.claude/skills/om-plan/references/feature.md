# feature 모드 — 신규 커스터마이징 추가 계획

## 목적

기존 등록에 없는 새 커스터마이징을 추가하는 계획을 만든다.

## 요청에 필요한 것

- `customization_id`: 새로 추가할 ID. 형식은 `CUSTOMIZATION_ID_PATTERN`을 따른다.
- 새 ID의 근거·후보를 별도 파일로 제출할 때는 `plan` 명령의 `--new-id-input` 경로로 준다.
- `owner`: 없으면 `null`.

## 검사기가 수집하는 사실 (discovered-facts)

- 공통: 등록된 ID 목록(`registered-customizations`), 등록된 테스트, 공유 경로 소유자, 중복 등록 ID.
- feature 모드는 initial처럼 커밋 인벤토리를 자동 수집하는 별도 분기가 없다. 새 기능의 변경 범위는 요청 입력과 근거로 제시한다.

## LLM이 만들 proposal 산출물

> 완성 예시: `example-feature-proposal.md`. 문서 필수 필드·evidence_refs 형식은 `proposal-format.md`.

- 새 매니페스트 초안: 새 `customization_id`의 `changed_paths`와 목적.
- 명단 항목 초안: 새 ID의 `title`, `status`, `criticality`. `owner`는 요청에 없으면 비운다.
- 재사용 판단: 기존 커스터마이징·공유 코드와 겹치는지 확인하고 겹치면 `shared_impact`로 지분을 선언한다.
- 계약 초안: 새 기능이 지켜야 할 동작과 확인 테스트.
- 미해결 질문: 담당자, 재사용 범위 등.

## 이 모드 특유 규칙

- [필수] 새 `customization_id`가 이미 등록돼 있으면 검사기가 거부한다(block). 등록 목록에 없는 ID여야 한다.
- 새 기능이 기존 공유 코드의 포인터를 옮기는 경우, "기능 손실" 주장 대신 공유 코드 정의 delta로 표현한다.
- 관찰 주장에는 `evidence_refs`를 붙인다.

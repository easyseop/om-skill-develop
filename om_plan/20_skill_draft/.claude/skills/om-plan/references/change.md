# change 모드 — 기존 등록 변경 계획

## 목적

이미 등록된 커스터마이징의 변경을 반영하는 계획을 만든다.

## 두 경로 (Q1=C)

change 모드는 `request.change_path`로 두 경로를 가진다.

- `pre_plan`: 변경을 하기 전, 무엇을 어떻게 바꿀지 계획한다. 관찰된 변경 diff는 아직 없다.
- `post_change_reconcile`: 이미 변경한 뒤, 실제 변경을 등록과 대조해 정합을 맞춘다.

## 요청에 필요한 것

- `customization_id`: 변경할 기존 ID.
- `change_path`: `pre_plan` 또는 `post_change_reconcile`.
- `post_change_reconcile`이면 `refs`에 `custom_baseline`과 `candidate`가 필요하다.

## 검사기가 수집하는 사실 (discovered-facts)

- `post_change_reconcile`:
  - `observed-change-paths`: `custom_baseline`→`candidate` 순변경 경로.
  - `observed-change-source-objects`: 위 경로의 실체 객체.
- 공통: 등록된 ID 목록, 등록된 테스트, 공유 경로 소유자.

## LLM이 만들 proposal 산출물

> 완성 예시: `example-change-proposal.md`. 문서 필수 필드·evidence_refs 형식은 `proposal-format.md`.

- 변경 계획(pre_plan) 또는 변경 정합(post_change_reconcile).
- 매니페스트 delta: 바뀐 `changed_paths`만.
- 재실행할 계약·테스트: 변경으로 영향받는 계약과 그 테스트.
- 공유 영향: 변경 경로가 공유 경로면 소유자 지분을 선언한다.
- 미해결 질문: 사람 결정 항목.

## 이 모드 특유 규칙

- [필수] 대상 `customization_id`가 등록돼 있지 않으면 검사기가 거부한다(block). 등록된 ID여야 한다.
- 필수 테스트를 실행하지 않고 `skip`으로 두면 거부된다.
- 관찰 주장에는 `evidence_refs`를 붙인다.

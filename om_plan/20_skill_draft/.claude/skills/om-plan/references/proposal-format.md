# proposal 문서 형식과 검사기 검증 규칙

LLM은 `proposal/` 폴더에 YAML 문서를 쓴다. 검사기(`plan check`)가 이 문서를 읽어 `discovered-facts.json`과 대조한다. 아래 규칙은 검사기 코드(`harness/acgh/plancore/validate.py`, `harness/acgh/integrations/om/collectors.py`)가 실제로 강제하는 것이다. 규칙을 어기면 block(1) 또는 analysis_error(3)로 반려된다. 완성 예시는 `example-initial-proposal.md`를 읽는다.

## 문서 로딩 규칙

- `proposal/` 아래 모든 `*.yaml`·`*.json`을 읽는다. 심볼릭 링크가 있으면 거부한다.
- proposal 문서는 "사실(machine evidence)"로 쓰이지 않는다. 사실은 검사기가 재계산한다. proposal은 사람이 검토할 제안·질문이다.

## 최소 요건 (floor)

- 문서 전체에 최소 하나의 `decisions` 항목, `findings` 항목, 또는 명시적 `no_change`가 있어야 한다. 빈 서술만 있으면 거부된다.
- `no_change: true`를 쓰면 `decisions`·`findings`와 함께 쓸 수 없다.

## decision 필수 필드 (검사기 강제)

`decisions`의 각 항목은 매핑이며 아래 **6개 필드를 모두** 가져야 한다. 하나라도 없으면 거부된다.

- `subject`: 이 결정이 다루는 대상(예: 커스터마이징 ID·제목).
- `decision`: 무엇을 하기로 했는지.
- `decision_source`: 반드시 `proposed`·`human_input`·`observed` 중 하나. (그 외 값은 거부)
- `evidence_refs`: 아래 "evidence_refs 형식"을 따르는 근거 리스트. `decision_source: observed`이면 비어 있으면 안 된다.
- `affected_customization_ids`: 영향받는 커스터마이징 ID 리스트.
- `required_follow_up`: 남은 후속 작업(없으면 빈 리스트나 명시적 "none").

## evidence_refs 형식 (검사기 강제)

각 evidence_ref 항목은 다음 둘 중 하나다.

- 문자열: `"<파일>#<JSON포인터>"`
- 매핑: `{ref: "<파일>#<JSON포인터>", expected: <기대값>}` — `expected`가 있으면 검사기가 그 포인터의 실제값과 `==` 비교한다.

제약:

- `<파일>`은 run 폴더 기준 상대경로이며 run 폴더 안이어야 한다. 사실상 `discovered-facts.json`을 쓴다.
- `proposal/` 아래 파일은 근거로 쓸 수 없다(자기참조 금지).
- `#` 뒤는 JSON-Pointer다. `/`로 구분하며, 리스트는 정수 인덱스, 매핑은 키로 내려간다. 예: `/canonical_payload/items/2/value/0/customization_ids`.
- 포인터의 인덱스는 운영자의 실제 `discovered-facts.json` 구조에 바인딩된다. 사실을 읽어 해당 값의 실제 좌표를 만들고, `expected`에는 그 값을 그대로 복사한다.

## no_change 요건 (검사기 강제)

`no_change`를 쓰면 다음이 모두 필요하다.

- `no_change: true`
- `rationale`: 비어 있지 않은 문자열.
- `affected_customization_ids`: 리스트.
- `evidence_refs`: 각 항목이 매핑 `{ref, expected}`이고 `ref`가 `discovered-facts.json#`로 시작해야 한다.

## 그 밖에 담을 수 있는 필드

- `findings`: 리스트. 관찰·판단 항목.
- `questions` 또는 `unresolved_questions`: 리스트. 사람만 답할 항목(담당자 등).
- `owner`: 담당자. 요청 `owner`가 비어 있으면 여기에 값을 채우지 않는다(지어내기 금지).
- `next_step_blocked`: `true`이면 STOP. 담당자 미정·모호한 커밋에서 필요하다.
- `shared_impact` 또는 `shared_impacts`: 리스트. 각 항목은 `{path, customization_ids}`. 공유 경로의 지분을 선언한다.
- `required_tests`(decision 안): 각 항목 `{id, status, required, result}`. `status: existing`은 `registered-tests`에 있어야 하고, `required: true`인데 `result`가 `skip`·`skipped`·`not_run`이면 거부.

## 검사기가 강제하는 그 밖의 규칙

- 담당자 미정: 요청 `owner`가 비어 있으면 proposal에 담당자 질문 + `next_step_blocked: true`가 있어야 한다.
- 담당자 지어내기 금지: 요청 `owner`가 비어 있는데 proposal `owner`에 실제 값을 넣으면 거부.
- 모호한 커밋: `customization-commits` 중 `customization_ids`가 정확히 1개가 아닌 커밋이 있으면, 그 커밋 sha(앞 12자) 또는 "ID"를 담은 미해결 질문과 STOP이 있어야 한다.
- 공유 경로 소유자: 변경 경로가 공유 경로면 `shared_impact`의 소유자 집합이 `shared-path-owners`와 일치해야 한다.
- 모드별: `feature`는 대상 ID가 이미 등록돼 있으면 거부. `change`는 대상 ID가 등록돼 있지 않으면 거부. 중복 등록 ID가 있으면 거부.

## 참고: 최종 등록 자료(answer-key) 목표 형식

proposal이 승인되면 최종적으로 아래 형식의 등록 자료가 된다(반영은 `apply` 단계, /om-plan 범위 밖).

- `manifests/<ID>.yaml`: `schema_version: 2`, `customization_id`, `status`, `kind`, `title`, `implementation.changed_paths[]`.
- `customization-registry.yaml`의 `entries[]`: `customization_id`, `title`, `owner`, `owner_status`, `status`, `criticality`, `manifest`, `contracts[]`, `provenance`.

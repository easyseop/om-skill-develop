# initial 모드 — 최초 등록 계획

## 목적

공식 버전 대비 현재 커스텀에 들어간 모든 커스터마이징을 처음으로 식별해 등록 계획을 만든다.

## 요청에 필요한 것

- `refs.official`: 공식 버전 커밋.
- `refs.current_custom`: 현재 커스텀 커밋.
- `owner`: 없으면 `null`. 담당자는 사람이 정한다.

## 검사기가 수집하는 사실 (discovered-facts)

- `customization-commits`: 커밋 인벤토리. 각 항목은 `sha`, `subject`, `customization_ids`(커밋 트레일러에서 추출), `changed_paths`.
- `net-custom-paths`: official→current_custom 순변경 경로.
- `net-custom-source-objects`: 위 경로의 실체 객체.
- 공통: 등록된 ID 목록, 등록된 테스트, 공유 경로 소유자, 중복 등록 ID.

## LLM이 만들 proposal 산출물

커밋 인벤토리의 `customization_ids`를 기준으로 커스터마이징별로 다음을 작성한다. 문서의 필수 필드·evidence_refs 형식은 `proposal-format.md`, 완성 형태는 `example-initial-proposal.md`를 읽는다.

- 매니페스트 초안: 각 커스터마이징 ID의 `changed_paths`를 커밋 사실에서 모은다(형식은 `proposal-format.md`의 매니페스트 목표 형식).
- 판단 항목(`decisions`): 각 커스터마이징 등록 결정을 `proposal-format.md`의 필수 6필드로 작성한다. 커밋에서 관찰했으므로 `decision_source: observed`이고 `evidence_refs`로 커밋 사실을 가리킨다.
- 공유 영향(`shared_impact`): 변경 경로가 공유 경로면 소유자 지분을 선언한다. 없으면 `[]`.
- 미해결 질문: 담당자 미정, 업무 정상 조건 등 사람 결정 항목. 담당자 미정이면 `next_step_blocked: true`.

## 필드 채우기 규칙 (사실에 없는 값 처리)

- `status`: 제안 단계이므로 `proposed`. 승인·반영 후 `active`가 된다.
- `kind`: 커밋이 제품 핵심 코드를 직접 수정하면 `core-patch`. 판단이 서지 않으면 미해결 질문으로 남긴다.
- `title`: 사실에 title이 없다. 커밋 subject를 기반으로 한 초안으로 쓰고, 담당자 검토 대상임을 미해결 질문에 남긴다.
- `criticality`: 사실 근거가 없다. 지어내지 않는다. 미상으로 두거나 미해결 질문으로 남긴다.
- `contracts`(업무 정상 조건): `registered-tests`·`registered-contracts` 사실이 비어 있으면 근거가 없다. 계약 내용을 지어내지 말고 "담당자가 정의 필요"를 미해결 질문으로 남긴다.

## 이 모드 특유 규칙

- 커밋 하나가 여러 ID에 걸치거나 ID가 없으면(모호), 그 커밋 sha(앞 12자)를 담은 질문 + STOP을 넣는다.
- 한 커스터마이징이 여러 커밋에 나뉘어 있으면(예: 2커밋 시리즈) 하나의 매니페스트로 합쳐 인식한다.
- 관찰 값은 모두 `discovered-facts`에서 온다. 지어내지 않는다.

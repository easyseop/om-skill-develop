# upgrade 모드 — 버전 업그레이드 영향 계획

## 목적

공식 버전을 인접한 다음 버전으로 올릴 때, 커스터마이징이 받는 영향을 정리하고 이동·재적용 계획을 만든다. 이 단계는 계획만 만든다(실제 반영·반영 후 대조는 범위 밖).

## 요청에 필요한 것 (스키마 강제)

- `hop_policy`: `adjacent_only` (고정값).
- `versions`: `base`·`target`(예: base "1.13.1", target "1.13.2"). 인접만 허용. 비인접은 `analysis_error`(Q2=A).
- `deployment_method`: 배포 방식(예: docker). 필수(Q5=B).
- `official_documents`: 최소 1건. 각 항목은 `{source, version_token, deployment_methods}`.
  - `source`: http/https/file URL 또는 로컬 파일 경로.
  - `version_token`: 문서 원문 bytes 안에 실제로 들어 있어야 한다(검사기가 포함 여부 검증).
  - `deployment_methods`: 요청의 `deployment_method`를 덮어야 한다(또는 `all`). 안 덮으면 `DEPLOYMENT_DOCUMENT_MISSING`.
- `refs`: `official_base`·`official_target`·**`custom_baseline`** 세 개 모두 필수. `custom_baseline`은 현재 커스텀 기준(커스터마이징의 현재 위치).

## 검사기가 수집하는 사실 (discovered-facts)

- `official-upgrade-paths`: `official_base`→`official_target` 순변경 경로(공식이 바꾼 것). 인접 릴리스라도 수백~수천 개일 수 있다.
- `official-upgrade-source-objects`: 위 경로의 실체 객체.
- `official-documents`: 공식 문서 snapshot(원문 bytes·버전 토큰·byte digest).
- 공통: 등록된 ID 목록, 등록된 계약·테스트, 공유 경로 소유자.

## LLM이 만들 proposal 산출물

문서의 필수 필드·evidence_refs 형식은 `proposal-format.md`를 따른다. upgrade는 아래 산출물이 추가된다. 각 산출물의 **정확한 필드·enum은 아래 "검사기가 강제하는 upgrade 전용 규칙"을 반드시 지킨다.**

- `decisions`: 공통 6필드. 최소 하나는 `official-upgrade-paths`에 근거(evidence_ref)한다.
- `pointer_movements`: 파일 이동·이름변경으로 커스터마이징 경로가 옮겨간 경우.
- `shared_code_definitions_delta`: 공유 코드가 바뀐 경우 값 이동을 delta로 표현.
- `crosschecks`: 공식 문서가 말하는 변경과 실제 코드 변경 대조.
- `operations`(또는 `checklist`): 마이그레이션·재적용 순서와 배포방식별 주의.
- `shared_impact`: 아래 규칙 1에 따라 필수.
- `findings`, `unresolved_questions`: 관찰·판단·사람 결정 항목.

## 검사기가 강제하는 upgrade 전용 규칙 (collectors.py 코드에서 확인)

어기면 block(1)이다. 이 규칙들은 문서 검토로는 안 보이고 실제 검증에서 발동한다.

1. **`shared_impact`는 옵션이 아니라 필수다.** 검사기가 `official-upgrade-paths ∩ shared-path-owners`를 자동 계산하고, 교집합의 **각 경로마다** `shared_impact`에 소유자 집합을 `shared-path-owners`와 정확히 일치하게 선언해야 한다. 공식 diff가 크면 교집합도 크다(예: 1.13.1→1.13.2에서 23경로). 하나라도 빠지면 그 경로마다 block. `[]`는 교집합이 실제 0일 때만 쓴다.

2. **운영 finding은 체크리스트에 연결돼야 한다.** `findings` 중 `category`가 `db_migration`·`reindex`·`configuration`인 항목은, 그 `id`가 `operations[].finding_id`(또는 `operations[].source_finding`, `checklist[].finding_id`)로 참조돼야 한다. 안 하면 "operational finding is missing from checklist".

3. **`crosschecks[].relation`은 정해진 5개만 허용된다.**
   - `documented_and_observed`
   - `documented_not_located`
   - `code_change_not_documented`
   - `documented_contradicts_observed`
   - `not_applicable`
   그 외 값은 "unsupported document/code relation"으로 block.

4. **`pointer_movements`를 선언하면 `shared_code_definitions_delta`도 반드시 함께 선언한다.** 파일이 이동만 하고 기능이 살아 있으면 "기능 손실"이 아니라 경로 이동 + 공유 코드 정의 delta로 표현한다(앵커 stale에 속지 않는다). delta 없이 pointer_movements만 있으면 block.

## 그 밖의 주의

- 1702개처럼 큰 diff를 전부 remap할 필요는 없다. 근거로 확정할 수 있는 것만 결정으로 쓰고, 분석이 필요한 부분(커스텀별 remap 완전성 등)은 `unresolved_questions`로 남긴다.
- 공식 문서 snapshot의 byte digest는 같은 run 안에서만 무결성을 보장한다(동적 byte로 run 간 문서가 다를 수 있다).
- request `owner`가 null이면 담당자 질문 + `next_step_blocked: true`.
- 완성 예시: `example-upgrade-proposal.md`(강제 4규칙이 모두 보이는 통과 구조).

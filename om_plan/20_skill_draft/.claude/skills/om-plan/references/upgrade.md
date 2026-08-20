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

## LLM이 만들 proposal 산출물 — 9항목 체크리스트 (01A §4)

upgrade는 아래 9개(CHK-1~9)를 **모두** 다뤄야 한다. 해당 없으면 그 항목에 `not_applicable`+사유를 명시한다(그냥 누락 금지). 각 항목의 필드·enum은 "검사기가 강제하는 규칙"을 지킨다.

| ID | 산출물 | 무엇 |
|---|---|---|
| CHK-1 | `official-doc-findings` | 공식 문서(원문 snapshot)에서 뽑은 요구사항·발견 |
| CHK-2 | `doc-code-crosscheck`(`crosschecks`) | 문서 설명 ↔ 실제 코드 변경 대조(relation 5종) |
| CHK-3 | `upgrade-plan` | 반영 계획(이동·대체·충돌) |
| CHK-4 | `path-remap` | 경로 이동표 |
| CHK-5 | `manifest-deltas` | 매니페스트 변경분 |
| CHK-6 | `shared_code_definitions_delta` | 공유 코드 정의 이동 delta |
| CHK-7 | `contracts-to-run`(`required_tests`) | 재실행할 계약/테스트 |
| CHK-8 | `operations` | 운영(migration·reindex·config) |
| CHK-9 | `unresolved_questions` | 사람 결정 항목 |

- `decisions`(공통 6필드): 최소 하나는 `official-upgrade-paths`에 evidence_ref로 근거한다.
- `shared_impact`: 아래 규칙 1에 따라 필수(공유 경로 교집합).
- `pointer_movements`: 파일 이동을 관찰하면 신고(→ CHK-6 delta 동반).
- **추적성**: 관리 파일 변경(manifest-deltas 등)에는 `checklist_ref: [CHK-n]`로 어느 항목 때문에 바뀌었는지 표기한다(구조화 필드, 주석 아님). 단 이건 자기신고라 참고용이며, 실제 강제는 아래 층1 커버리지다.

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

## 9항목이 어떻게 검증되나 — 3층 (Q11 결정, 반영 예정)

위 4규칙(shared_impact·operations 연결·relation·delta 결속)은 이미 강제된다. 나머지 9항목은 아래 3층으로 검증된다(Codex 조치3·6 반영 예정, 그전까지는 LLM 책임).

- **층1 [검사기가 사실로 강제 — 못 속임]**
  - CHK-4 `path-remap`: **존재만이 아니라 커버리지.** 영향 경로(`official-upgrade-paths ∩ 등록 커스텀 경로`)를 **전부** remap해야 한다. 빠진 경로 있으면 block. (shared_impact와 동형)
  - CHK-2 `crosscheck`: CHK-1 `official-doc-findings`의 각 항목이 crosscheck에 연결돼야 한다.
- **층2 [독립 2차 판독 — 검사기 아님]**
  - CHK-1 `official-doc-findings`·CHK-8 `operations`의 **문서 누락**은 뒤 계약 테스트가 못 잡는다(코드 아닌 배포 요구사항). 그래서 **분리된 LLM이 원문 snapshot을 다시 읽어** 요구사항을 독립 추출·대조한다. 누락(예: 릴리스노트의 "기동 전 DB 마이그레이션"을 빠뜨림) 발견 시 질문/차단. 같은 LLM 자기검토는 편향이라 무의미하니 반드시 독립 컨텍스트.
- **층3 [존재만]**: CHK-3·5·7·9 등 판단형은 비어 있지 않은 존재만.

> 즉, 코드 산출물(remap 등)은 뒤 테스트가 잡으니 사실-커버리지로, **문서 산출물은 뒤가 못 잡으니 2차 판독으로** 메운다. "9칸 채우기+N/A 남발"은 위장이니 피하고, **영향 경로를 실제로 다 덮었는지**로 판정한다.

## 별도 게이트 — watch/risk (이 계획과 분리)

sensitive-zones(frozen→block / protected→approval / watched→pass)의 검사는 `/om-plan`이 아니라 **별도 명령**이 한다:

- `watch`(`run_upgrade_watch.py`): 새 공식 버전이 watch 경로를 바꿨는지 **병합 전** 검사.
- `risk`(`run_upgrade_risk_gates.py`): 업그레이드 위험 검사.

주의: 이 두 스크립트는 현재 clean-export(GitLab 반영본)에 **동봉되지 않아 거기선 실행할 수 없다(갭 A1·A2, 22 감사)**. upgrade 계획은 이들을 실행하지 않지만, 공식 변경이 sensitive-zone(예: `bootstrap/sql/**`=watched)을 건드리면 **finding·질문으로 사람에게 표시**해 별도 게이트가 돌아야 함을 알린다.

## 그 밖의 주의

- 1702개처럼 큰 diff를 전부 remap할 필요는 없다. 근거로 확정할 수 있는 것만 결정으로 쓰고, 분석이 필요한 부분(커스텀별 remap 완전성 등)은 `unresolved_questions`로 남긴다.
- 공식 문서 snapshot의 byte digest는 같은 run 안에서만 무결성을 보장한다(동적 byte로 run 간 문서가 다를 수 있다).
- request `owner`가 null이면 담당자 질문 + `next_step_blocked: true`.
- 완성 예시: `example-upgrade-proposal.md`(강제 4규칙이 모두 보이는 통과 구조).

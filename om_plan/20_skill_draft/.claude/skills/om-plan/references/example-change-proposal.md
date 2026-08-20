# 완성 예시 — change 모드 proposal (BANK-OM-007, post_change_reconcile)

clean-room에서 `plan check` 통과(approval)한 실제 구조다. `custom_baseline`→`candidate`에서 검사기가 자동 탐지한 변경 2경로를 등록과 정합한다. 포인터 인덱스는 운영자의 실제 `discovered-facts.json`에 맞춰 재확인한다.

```yaml
# manifest delta(정보용) — 바뀐 changed_paths 만
manifest:
  schema_version: 2
  customization_id: BANK-OM-007
  status: proposed
  kind: core-patch
  title: Tibero 서비스 연결 커버리지 (초안 — 담당자 검토 필요)
  implementation:
    changed_paths:            # observed-change-paths(items/2) 그대로 복사한 delta
    - openmetadata-ui/src/main/resources/ui/src/generated/entity/services/connections/serviceConnection.ts
    - openmetadata-ui/src/main/resources/ui/src/utils/DatabaseServiceUtils.test.tsx

decisions:
- subject: BANK-OM-007 등록 변경 정합 (post_change_reconcile)
  decision: >-
    관찰된 변경 경로 2개를 기존 등록 BANK-OM-007의 manifest delta로 정합한다.
  decision_source: observed
  affected_customization_ids:
  - BANK-OM-007
  evidence_refs:
  - ref: discovered-facts.json#/canonical_payload/items/2/value/0
    expected: openmetadata-ui/src/main/resources/ui/src/generated/entity/services/connections/serviceConnection.ts
  - ref: discovered-facts.json#/canonical_payload/items/2/value/1
    expected: openmetadata-ui/src/main/resources/ui/src/utils/DatabaseServiceUtils.test.tsx
  - ref: discovered-facts.json#/canonical_payload/items/4/value/6   # 대상 ID가 등록됨
    expected: BANK-OM-007
  required_tests:
  - id: tests/bank/contracts/test_tibero.py::test_connection_schema_roundtrip
    status: existing          # registered-tests에 존재
    required: false           # 계획 단계라 실행 전. required:true+not_run은 block.
    result: not_run           # 실행 결과를 지어내지 않는다. apply 단계에서 실행·기록.
  required_follow_up:
  - apply 단계에서 위 테스트를 실제 실행하고 result를 기록한다(required:true 승격).
  - 담당자 확정 후 owner_status를 assigned로 변경한다.

shared_impact: []             # 두 변경 경로 모두 shared-path-owners의 키가 아니다(교집합 없음)

unresolved_questions:
- BANK-OM-007의 실제 운영 담당자(owner)는 누구인가? 요청 owner가 null이라 사람이 확정한다.
- 이 변경이 매핑되는 계약이 하나가 맞는지 담당자가 확인한다.
next_step_blocked: true
```

## 핵심

- 변경 경로는 검사기가 자동 탐지한 `observed-change-paths`를 그대로 근거화한다(지어내지 않음).
- 미실행 필수 테스트는 `required:false`+`not_run`+follow-up으로 표현한다(`required:true`+미실행은 block).
- 대상 ID가 등록돼 있음을 evidence_ref로 증명한다(change는 등록된 ID여야 함).

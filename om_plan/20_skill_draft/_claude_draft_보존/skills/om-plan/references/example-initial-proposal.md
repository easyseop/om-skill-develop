# 완성 예시 — initial 모드 proposal (BANK-OM-001)

아래는 `proposal/BANK-OM-001.yaml`의 완성 형태다. 검사기의 필수 필드·evidence_refs 형식을 모두 만족한다. **포인터의 인덱스(`items/2`, `value/0`)는 예시이며, 운영자의 실제 `discovered-facts.json`을 읽어 그 값의 실제 좌표로 바꾸고 `expected`에 실제값을 복사한다.**

```yaml
# 등록 대상 매니페스트(정보) — answer-key 목표 형식
manifest:
  schema_version: 2
  customization_id: BANK-OM-001
  status: proposed          # 제안 단계 값. 승인·반영 후 active가 된다
  kind: core-patch          # 커밋이 제품 핵심 코드를 직접 수정하면 core-patch
  title: 기준코드(InstanceCode)   # 사실에 title이 없으면 커밋 subject 기반 초안, 담당자 검토 필요
  implementation:
    changed_paths:
    - bootstrap/sql/migrations/native/1.13.1/mysql/schemaChanges.sql
    - openmetadata-service/src/main/java/org/openmetadata/service/resources/instancecode/InstanceCodeResource.java
    - openmetadata-spec/src/main/resources/json/schema/entity/data/instanceCode.json
    # ... 실제로는 커밋의 changed_paths 48개 전부. 사실에서 그대로 복사한다.

# 검사기가 검증하는 핵심 — decisions
decisions:
- subject: BANK-OM-001 기준코드(InstanceCode)
  decision: 커밋 23ec5bf의 변경 48개 경로를 BANK-OM-001로 최초 등록한다
  decision_source: observed
  affected_customization_ids:
  - BANK-OM-001
  evidence_refs:
  - ref: discovered-facts.json#/canonical_payload/items/2/value/0/customization_ids
    expected:
    - BANK-OM-001
  - ref: discovered-facts.json#/canonical_payload/items/2/value/0/sha
    expected: 23ec5bf00e67d22815ef314b389abe424097bba2
  required_follow_up:
  - 담당자 확정 후 owner_status를 assigned로 변경
  - 업무 정상 조건(contracts)을 담당자가 검토

# 담당자 미정 → 질문 + STOP (요청 owner=null)
unresolved_questions:
- BANK-OM-001의 실제 운영 담당자(owner)는 누구인가? 요청에 없어 사람이 확정해야 한다.
next_step_blocked: true

# 공유 경로 없음(shared-path-owners 비어 있고 교집합 없음)
shared_impact: []
```

## 왜 이 형태인가 (검사기 대응)

- `decisions[0]`에 필수 6필드(`subject`·`decision`·`decision_source`·`evidence_refs`·`affected_customization_ids`·`required_follow_up`)를 모두 넣었다.
- `decision_source: observed`이므로 `evidence_refs`가 비어 있으면 안 된다. 그래서 커밋 사실을 가리키는 ref 2개를 넣고 `expected`로 실제값을 못 박았다.
- ref는 `discovered-facts.json#<JSON포인터>` 형식이고 run 폴더 안 파일만 가리킨다. `proposal/` 자기참조는 쓰지 않았다.
- 요청 `owner`가 null이라 `owner`를 채우지 않고 `unresolved_questions` + `next_step_blocked: true`로 STOP했다.
- `manifest` 블록은 정보용이다. 검사기 floor는 `decisions`로 충족된다.

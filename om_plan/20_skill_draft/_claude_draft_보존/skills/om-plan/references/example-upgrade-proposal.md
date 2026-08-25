# 완성 예시 — upgrade 모드 proposal (1.13.1→1.13.2)

clean-room에서 `plan check` 통과(approval)한 실제 구조를 압축한 것이다. upgrade 전용 강제 4규칙(공유영향 필수·운영 finding 연결·crosscheck relation enum·pointer_movement/delta)이 모두 보이게 했다. 실제로는 큰 diff라 shared_impact가 23개였다 — 아래는 형식만 보이도록 일부만 싣는다.

```yaml
proposal_title: OpenMetadata 공식 업그레이드 영향 계획 — 1.13.1 → 1.13.2 (adjacent)
mode: upgrade
upgrade: {base: "1.13.1", target: "1.13.2", deployment_method: docker, hop: adjacent}
summary: >-
  상류 순변경 1702경로 중 23개가 등록 커스터마이징의 공유 경로와 겹친다.
  공유 소유권을 선언하고 세부 재매핑은 사람 검토로 남긴다. owner 미정으로 STOP.

decisions:
- subject: 공식 업그레이드 범위 1.13.1→1.13.2
  decision: official_base→official_target 순변경 경로 1702개를 영향 범위로 고정한다. 개별 remap은 후속 사람 검토.
  decision_source: observed
  affected_customization_ids: [BANK-OM-001, BANK-OM-002, BANK-OM-003, BANK-OM-004, BANK-OM-005, BANK-OM-006, BANK-OM-007]
  evidence_refs:
  - ref: discovered-facts.json#/canonical_payload/items/1/value/commit_sha   # official_base
    expected: afcb2d2cd7e7c28f1d0ce60538c60a96f4eb9dc9
  - ref: discovered-facts.json#/canonical_payload/items/2/value/commit_sha   # official_target
    expected: 2763bf97ce265662793a1a38d353147cc6d6c2e3
  required_follow_up:
  - 커스터마이징별 세부 경로 재매핑을 사람 검토로 확정한다.

# 규칙2: 운영 finding(category db_migration/reindex/configuration)의 id를 operations에서 참조해야 함
findings:
- id: F-DB-MIGRATION
  category: db_migration
  statement: 1.13.2 순변경에 native DB 마이그레이션 SQL이 포함된다. 기동 전 실행 필요.
  evidence_refs:
  - ref: discovered-facts.json#/canonical_payload/items/3/value/3
    expected: bootstrap/sql/migrations/native/1.13.2/mysql/schemaChanges.sql

operations:
- {step: 1, finding_id: F-DB-MIGRATION, action: 기동 전 1.13.2 native DB 마이그레이션 적용, deployment_method: docker}
- {step: 2, action: 23개 공유 경로의 커스텀 패치를 상류 1.13.2 위에 재적용/병합 후 재빌드}

# 규칙3: relation은 허용된 5개 enum 중 하나만
crosschecks:
- relation: documented_and_observed
  documented: 릴리스 노트가 "Run database migrations before starting the 1.13.2 services"를 명시
  observed: official-upgrade-paths에 1.13.2 mysql schemaChanges.sql 존재

# 규칙1: official-upgrade-paths ∩ shared-path-owners 교집합(여기선 23개) 전부 필수. 소유자는 사실 그대로.
shared_impact:
- {path: openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/CollectionDAO.java, customization_ids: [BANK-OM-001, BANK-OM-002]}
- {path: openmetadata-ui/src/main/resources/ui/src/locale/languages/ar-sa.json, customization_ids: [BANK-OM-001, BANK-OM-002, BANK-OM-003, BANK-OM-004]}
# … 나머지 21개 경로도 동일 형식으로 전부 선언(누락 시 경로별 block)

unresolved_questions:
- 이 업그레이드 계획의 운영 담당자(owner)는 누구인가? 요청 owner가 null이다.
- 커스터마이징별 세부 경로 재매핑은 official-upgrade-source-objects 대조가 필요하며 사람 검토로 남긴다.
next_step_blocked: true
```

## 강제 4규칙 대응 (없으면 block)

1. `shared_impact`: 교집합 경로(여기선 23개) **전부** 선언. 소유자는 `shared-path-owners` 사실 그대로. `[]`는 교집합 0일 때만.
2. 운영 finding(`db_migration`/`reindex`/`configuration`)의 `id`를 `operations[].finding_id`로 연결.
3. `crosschecks[].relation`은 5개 enum(`documented_and_observed` 등)만.
4. `pointer_movements`를 쓰면 `shared_code_definitions_delta`도 동반(이 예시는 pointer_movements 없어 생략).

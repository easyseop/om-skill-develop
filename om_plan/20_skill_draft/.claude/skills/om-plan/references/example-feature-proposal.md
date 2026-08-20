# 완성 예시 — feature 모드 proposal (BANK-OM-008, 신규)

clean-room에서 `plan check` 통과(approval)한 실제 구조다. feature는 requirement 기반이라 관찰된 변경 diff가 없다. 그래서 **신규 ID 가용성만 관찰(observed)로 근거화**하고, 구현 세부는 proposed 결정 + 질문으로 남긴다.

```yaml
manifest:
  schema_version: 2
  customization_id: BANK-OM-008
  status: proposed
  kind: unknown               # 구현 방식 미정. 근거 없어 지어내지 않는다.
  title: 감사 로그 보존기간 정책 설정   # requirement 기반 초안, 담당자 검토 필요
  implementation:
    changed_paths: []         # 구현 전이라 관찰된 변경 경로 없음(질문 참조)

decisions:
# 결정 1 — 신규 ID 가용성(관찰 근거). 등록·중복 목록에 BANK-OM-008 부재 확인.
- subject: BANK-OM-008 신규 ID 가용성
  decision: 등록·중복 목록 어디에도 BANK-OM-008이 없어 신규 ID 요건을 충족함을 확인한다.
  decision_source: observed
  affected_customization_ids:
  - BANK-OM-008
  evidence_refs:
  - ref: discovered-facts.json#/canonical_payload/items/1/value      # registered-customizations
    expected: [BANK-OM-001, BANK-OM-002, BANK-OM-003, BANK-OM-004, BANK-OM-005, BANK-OM-006, BANK-OM-007]
  - ref: discovered-facts.json#/canonical_payload/items/1/fact_id    # 인덱스 드리프트 방지용 sibling 핀
    expected: registered-customizations
  - ref: discovered-facts.json#/canonical_payload/items/2/value      # duplicate-registered-customizations
    expected: []
  required_follow_up:
  - none
# 결정 2 — 추가 계획(제안). 구현 미착수라 관찰 근거 없이 proposed.
- subject: BANK-OM-008 감사 로그 보존기간 정책 설정 커스터마이징
  decision: 감사 로그 보존기간 설정 신규 커스터마이징 추가를 제안한다. 경로·계약은 구현 착수 후 확정.
  decision_source: proposed
  affected_customization_ids:
  - BANK-OM-008
  evidence_refs: []           # proposed는 관찰 근거 불요(observed만 non-empty 강제)
  required_follow_up:
  - 실제 변경 파일(changed_paths)을 확정하고 매니페스트를 채운다.
  - 동작을 검증할 계약과 테스트를 정의한다.
  - 실제 운영 담당자(owner)를 확정한다.

findings:
- summary: 보존기간 동작을 확인할 계약·테스트가 아직 없다(registered-contracts/tests에 BANK-OM-008 항목 없음).

shared_impact: []             # 변경 경로 미확정이라 겹침 판단 불가(질문으로 남김)

unresolved_questions:
- BANK-OM-008의 실제 운영 담당자(owner)는 누구인가? 요청 owner가 null이다.
- 실제 변경 파일(changed_paths)은 무엇인가? 구현 전이라 미확정.
- 이 기능의 kind는 무엇인가(core-patch/plugin 등)?
- 변경 경로가 공유 경로와 겹치는가? 경로 확정 전 판단 불가.
- 신설할 계약·테스트의 범위·이름은 무엇인가?
next_step_blocked: true
```

## 핵심

- 관찰 가능한 유일한 사실(신규 ID 부재)만 `observed`로 근거화하고, `expected`에 실제 등록 목록을 복사한다.
- 구현 세부(경로·kind·계약)는 **지어내지 않고** `proposed` 결정 + 질문으로 남긴다.
- `proposed` 결정은 `evidence_refs: []`가 허용된다(비어 있으면 안 되는 건 `observed`뿐).

# Codex 지시 — om-apply 리허설 발견 P0 수정 + 재리허설

작성일: 2026-08-24. 작성자: Claude. 근거: 리허설 결과 66(Claude가 P0-1 코드 실측 확인).

## 범위

**이번에 고침**: P0-1(검사기 결함) · P0-3(등록자료 정정 — **사람 승인됨 2026-08-24**) · P1-1(빠른검사 하한) · 격리 위생(P2-1 재발 방지) · 재리허설.
**이번에 안 함(기록됨)**: P0-2 활성 기준 잠금(기존 R-1·R-2 기준선 잠금 재검토 때 함께) · P1-2~P1-5(후속) · upgrade 독립 문서검토(별도 단계 — STOP 상태 유지가 정상).

## 1. P0-1 — `validate_management` 경로 역할 분리 (핵심)

`harness/acgh/integrations/om/apply.py` (약 L118-147):

- **`all_owned_paths`(전 ID union)** 는 **"final diff 경로가 어느 매니페스트에도 없음"**(등록 밖 변경) 검출에만 쓴다. — 현행 유지.
- **"매니페스트 경로에 최종 효과 없음(no final candidate effect)"** 검사는 전 ID union이 아니라 **이번 계획 대상(planned units의 ID들)의 계획 경로에만** 적용한다. 다른 ID의 경로가 이번 diff에 없는 것은 **정상**이다.
- 반례 추가(기존 OA와 함께 전부 유지돼야 함):
  - (a) **정상 단일 ID 변경**(다른 ID 경로는 diff에 없음) → **PASS + `static_consistent_awaiting_verify`** ← 이번 리허설이 못 간 그 지점
  - (b) 등록 밖 경로 → 여전히 block (OA-03 유지)
  - (c) 이번 계획의 required 경로가 최종 효과 없음 → 여전히 block (OA-02/04 유지)
  - (d) 여러 ID를 대상으로 한 계획이면 그 대상 ID들의 계획 경로가 모두 검사됨

## 2. P0-3 — 활성 공유소유맵 정정 (사람 승인됨)

- 대상: `harness/registrations/om-temp-1.13.1/shared-path-owners.yaml` (활성 등록자료 — **이번에 한해 수정 승인**).
- 매니페스트 파생 공유인데 맵에 누락된 2경로 추가:
  - `openmetadata-ui/src/main/resources/ui/src/generated/entity/services/connections/serviceConnection.ts`
  - `openmetadata-ui/src/main/resources/ui/src/utils/DatabaseServiceUtils.test.tsx`
- **소유자 목록은 추정 금지 — 실제 매니페스트들을 대조해 실측 기입**(보고상 BANK-OM-006·007이나, 기입 전 확인). 결과서에 **신·구 등록자료 digest** 기록(digest 변동은 정상).
- 이 정정 후 "manifest 파생 공유(39) ⊇ shared-path-owners" 관계가 **등가**가 되는지, 아니면 여전히 차이가 남는지 결과서에 명시.

## 3. P1-1 — 빠른검사 하한

- 계획의 `fast_checks`가 **비어 있으면 승인 불가**: 최소 1개, 또는 **구조화된 `not_applicable + 사유`** 필수(자유 생략 금지).
- 반례: fast_checks 없음+사유 없음 → block / `not_applicable+사유` → 통과.

## 4. 격리 위생 (P2-1 재발 방지)

- 이후 격리 복사는 **`.git` 파일·디렉터리 모두 제외**하고, 복사본에서 `git rev-parse --git-dir`가 **원본을 가리키지 않음을 확인** 후 사용.
- 오염됐던 `checker-template`은 **재사용 금지**(폐기).

## 5. 재리허설 (새 run — blocked run 재사용 금지)

수정 후, 활성 기준(`8ac18ad0...`)으로:
1. **change 통짜 재실행**(새 run): plan approval → apply start → LLM 반영 → apply check → **PASS + `static_consistent_awaiting_verify` + verify payload `eligible: true`** 도달.
2. **scope probe 재실행**: 계획밖+매니페스트 확장 → **여전히 block** (P0-1 수정이 이 게이트를 약화시키지 않았음을 증명).
3. 전체 회귀(기존 235 + 신규 반례) green.

## 금지

- push·MR·원격 쓰기 0. 검사기 **커밋 금지**(미커밋 작업트리 위 수정만). 활성 등록자료는 **§2의 2경로 추가 외 수정 금지**.
- 확인 못 한 것 단정 금지. "테스트 green"만으로 완료 주장 금지.

## 수용 기준 (Claude 재검증)

| # | 확인 |
|---|---|
| 1 | 정상 단일 ID 변경이 실데이터에서 **awaiting_verify 도달**, payload eligible:true |
| 2 | 등록 밖 변경·touch원복·계획밖+매니페스트확장 **여전히 block** |
| 3 | 공유소유맵 정정: 실측 소유자, 신구 digest 기록, 39↔37 차이 해소 여부 명시 |
| 4 | fast_checks 하한 동작(비면 block, N/A+사유는 통과) |
| 5 | 전체 회귀 green, 검사기 커밋 0·push 0 |

## 결과

`skill_develop/om_plan/68_Codex_omapply_P0수정_결과_20260824.md`: 수정 위치(파일:줄)·신구 digest·재리허설 각 단계 SHA/digest·반례 결과·남은 위험.

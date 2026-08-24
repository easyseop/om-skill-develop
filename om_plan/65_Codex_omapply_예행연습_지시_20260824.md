# Codex 지시 — om-apply v1 실데이터 예행연습

작성일: 2026-08-24. 작성자: Claude. 실행: Codex.
전제: om-apply v1 구현 완료·Claude 검증 통과(64). 지금까지 검증은 **임시 git 반례** 기준 — 이번엔 **실제 OpenMetadata 자료**로 통짜 흐름을 돌린다.

## 목적

실제 1.13.1 change · 1.13.2 upgrade 자료로 `om-plan(승인 계획) → apply start → LLM 반영 → apply check → verify 인계`가 **실물에서** 성립하는지 확인한다. 문제가 나오면 그게 이 예행연습의 성과다 — **억지로 통과시키지 말고 정직하게 보고**한다.

## 안전 경계 (절대 준수)

- **push·MR·원격 조작 금지.** 모든 작업은 로컬.
- **활성 등록자료(`om-temp-1.13.1`) 원본 수정 금지** — 예행연습용 복사본/격리 경로 사용.
- **검사기 트리에 새 커밋 금지**(om-apply 미커밋 작업트리 그대로). 제품 repo(OM_TEMP)는 **예행연습 전용 브랜치에서만** 커밋 허용(LLM 반영 단계의 산출이므로) — 어떤 브랜치를 만들었는지 결과서에 명시.
- 확인 못 한 것 단정 금지.

## 시나리오 1 — 1.13.1 change (핵심)

작은 실제 변경 1건(예: BANK-OM-005 한글 IME 등 소규모 ID)으로:

1. **om-plan**: change 요청( pre_plan 또는 post_change_reconcile 중 실제 흐름에 맞는 쪽)으로 `plan start`→proposal→`plan check` **approval** 획득.
2. **apply start**: 승인 계획 binding·refs·등록자료 digest 잠금 확인.
3. **LLM 반영**: 계획대로 제품 코드를 예행연습 브랜치에 구현(커스텀 ID 트레일러 커밋).
4. **apply check**: 3방향 대조 → **PASS + `static_consistent_awaiting_verify`**.
5. **verify 인계 payload** 확인: candidate SHA/tree·그 ID의 계약 required_tests가 정확한지.

### 시나리오 1 반례 probe (실데이터에서 게이트 발동 확인)
- 1-a. 반영 중 **계획 밖 파일 1개를 일부러 추가 변경**(+매니페스트에도 추가) → `apply check`가 **scope variance 질문 + not pass**로 멈추는지.
- 1-b. (가능하면) required 경로를 touch 후 원복 → block 되는지.

## 시나리오 2 — 1.13.2 upgrade

1. **om-plan upgrade**(1.13.1→1.13.2, 기존 예행연습 요청 자료 재사용 가능)로 approval 계획 획득.
2. **apply start → LLM 반영**: path-remap action(keep/move/replace/…)대로 커스텀을 1.13.2 위에 재적용. 병합 문제 발생 시 **3단 복구(①LLM 수정 →②재계획 →③사람행 STOP)** 순서대로 시도하고 각 단계 기록.
3. **apply check**: 통과 or **정직한 STOP**(뭐가 왜 막혔는지). upgrade는 어려운 시나리오라 STOP도 유효한 결과다.

## 수용 기준 (Claude가 재검증)

| # | 확인 |
|---|---|
| 1 | 시나리오1: 통짜 흐름이 실데이터에서 PASS + awaiting_verify 도달 |
| 2 | verify 인계에 정확한 candidate SHA/tree + 그 ID 계약 테스트만 포함 |
| 3 | 반례 1-a: 실데이터에서도 scope variance STOP 발동 |
| 4 | 3방향 정합: 최종 diff의 모든 경로가 ID 귀속(등록 밖 0), 관리파일=실제 일치 |
| 5 | 시나리오2: 진행 결과가 정직히 기록(통과든 STOP이든, 3단 복구 시도 포함) |
| 6 | 활성 등록자료 원본·검사기 커밋 불변, push 0 |

## 결과

`skill_develop/om_plan/66_Codex_omapply_예행연습_결과_20260824.md`:
- 사용한 저장소·refs·예행연습 브랜치명, 각 단계 실행 명령과 산출물 위치
- 시나리오별 판정(캡처한 verdict/final_state), 반례 probe 결과
- **발견된 문제 전부**(사소해도) + 그 원인 분석, 남은 위험
- 예행연습 산출물은 임시라 다른 컴퓨터에 전달 안 됨 — 결과서에 핵심 값(digest·SHA)을 남겨 재현 가능하게.

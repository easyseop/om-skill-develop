# Claude 독립 검증 — om-apply v1 구현 (Codex 결과서 63)

검증일: 2026-08-24. 대상: clean-export 브랜치 `codex/om-plan-verified-gates-20260820` 위 미커밋 작업트리. 정본 지시서: 61.

## 판정: 통과 (P0/P1 없음) — 단, 실데이터 예행연습 전

"통과 보고"를 믿지 않고 직접 확인한 것:

### 1. 실재·상태
- 주장된 신규 파일 전부 실재: `applycore/workflow.py`(1118줄)·`boundary.py`·스키마 4종·`integrations/om/apply.py`·`.claude/skills/om-apply/SKILL.md`·반례 753줄. **미커밋**(지시 준수), 기존 HEAD `7c544efb2b` 불변.

### 2. 내가 직접 실행
- 집중 반례 24개 + **전체 235개**(기존 210+반례24+배선1, 수 정합) 전부 통과(exit 0, 내 실행).

### 3. 핵심 설계 실측 (코드 근거)
- **성공 상태 단일**: 실패 없을 때만 `verdict=PASS` + `final_state=static_consistent_awaiting_verify`(workflow.py:1066-1068). 완료 run은 읽기전용(:697-701). verify 인계에 required_tests + `contract_status=claim_only_not_executed`(:1092-1094) — **계약충족을 pass로 안 침** ✅
- **자기세탁 차단(OA-01)**: 테스트가 `verdict != "pass"` + "scope variance" 질문 존재를 단언 — 계획밖+매니페스트 동시확장이 통과 못 함 ✅
- **touch후 revert 차단(OA-02)**: `verdict=="block"` + "touch/revert" 사유 단언 ✅
- **신규 공유 = 자동승인 없음**: `new shared path ... confirm ownership` **질문** 생성(apply.py:172-176), 소유자 누락은 block(:178-183) ✅
- **경계**: applycore에 OM 의존 0 (boundary.py의 "integrations" 문자열은 금지어 선언 자체) ✅
- **settings.json**: apply 명령 차단 1줄만 제거, push/tag 금지 유지 ✅

### 4. Codex의 정직 고지(동의)
direct_tests 재유입 없음(OA-17)·예전 B형 orchestration 미재사용·저수준 primitive만 재활용 — 지시서 61 §6과 일치.

## 잔여 (위험 아님·다음 단계)
1. **실데이터 예행연습 미실행** — 임시 git 반례 기준. 실제 1.13.1 change / 1.13.2 upgrade 자료로 `start→LLM 반영→check→verify 인계` 예행연습이 다음 관문.
2. verify 부재(awaiting_verify의 소비자 없음 — 3단계 구현 시), STOP 후 재개 채널 없음(새 run 재시작), apply 전용 Hook 없음 — 전부 v1 범위 밖으로 기록됨.
3. commit·push는 예행연습 후 사람 결정.

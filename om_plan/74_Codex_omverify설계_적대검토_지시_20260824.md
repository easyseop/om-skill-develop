# Codex 적대검토 지시 — om-verify 설계 초안(73)

작성일: 2026-08-24. **성격: 설계 적대검토(분석).** 구현·커밋·push 금지. 코드·인프라를 실측해 허점을 찾고 선택지·근거를 낸다.

## 대상
- 검토: `skill_develop/om_plan/73_omverify_설계초안_20260824.md` (사람 결정 V-1~V-3 확정 반영본) + 참고 `72_...`(test-agent 파일럿·채택).
- 실측 근거 코드:
  - om-apply의 verify 인계 payload 생성: clean-export `harness/acgh/applycore/workflow.py`(binding 부분, 약 L1070-1095).
  - 기존 계약 테스트 인프라: 완전본 트리 `tests/bank/contracts/_runtime_contract.py`(서버 지정·`OPENMETADATA_AUTH_TOKEN`)·`_browser_contract.py`.
  - lab 인프라: `~/openmetadata-lab` (기동 방식·이미지 관리 — **접근 불가하면 결과 첫머리에 명시**하고 그 항목은 Claude 실측으로 위임).
- 호의적 보완 해석 금지. 파일:줄 근거. 사람 결정 재확정 금지(V-1~V-3은 임시 확정 상태 — 뒤집을 근거가 나오면 "재검토 권고"로만).

## 집중 검토 (깨보기)

1. **① 인계 재검증** — "required_tests를 계약에서 재계산"이 정말 위조 불가인가:
   - 계약 파일 자체가 apply 이후 바뀌었으면? (apply 시점 등록자료 digest ↔ verify 시점 재계산의 시간차)
   - required_tests가 **빈** 커스텀이면? (테스트 0개 → verified가 되는 빈틈?)
2. **② 환경↔candidate 결속** (최중요) — "떠 있는 서버 ≠ candidate" 바꿔치기:
   - `/api/v1/system/version`의 revision은 빌드 시 주입 값 — **조작 가능한가?** 이미지 digest는 누가 어떻게 검증하나?
   - **lab(A안) 실측**: 현재 openmetadata-lab이 어떻게 기동돼 있고(compose? 이미지 태그?), candidate 배포·결속이 현실적으로 어떤 절차가 되는지. C안(평시 A/릴리즈 B)이 이 인프라에서 구현 가능한지.
3. **③ pytest 실행 우회** — 스킵/부분실행/캐시/`-k` 필터 조작이 "전부 pass"로 위장할 길. 기존 `_runtime_contract.py`가 서버를 어떻게 지정·인증하는지 실측(verify가 그 방식을 재사용 가능한가).
4. **⑤ receipt** — 재사용·짜깁기: SHA 결속·읽기전용으로 충분한가? receipt 생성 주체가 조작하면?
5. **④ UI층 동어반복 방지** — 시나리오 정답원이 구현 복제가 되는 걸 기계적으로 막을 수 있나(없으면 정직하게 "사람 검토 항목"으로).
6. **판정 3상태의 빈틈** — verified/failed/infra_error 어디에도 안 걸리는 경우, 그리고 "infra_error를 반복해 통과처럼 보이게" 하는 운용 우회.

## 결과
`skill_develop/om_plan/75_Codex_omverify설계_적대검토_결과_20260824.md`: 항목별 [허점·근거(파일:줄/실측)·심각도·수정 제안], lab 실측 결과(또는 접근불가 명시). 확정 금지 — 선택지·권고로.

## 금지
구현·수정·커밋·push·MR 금지. lab 서버 상태 변경 금지(읽기 확인만). 확인 못 한 것 단정 금지.

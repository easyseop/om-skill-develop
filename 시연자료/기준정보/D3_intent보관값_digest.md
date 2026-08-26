> 수집: 2026-08-26. 사람 검토(intent_review) 보관값 — 서버 세션(LLM)이 수정 불가한 위치(별도 컴퓨터 git)에 보존. 검증자 대조 완료.

## 보관값 (6단계 plan check에 명시할 것)
input_lock_digest: sha256:1cfae81f527b4fdd15bead0af582a801c343e4276a6a7565ef15cf1236350acf

## intent 대조 (검증자 확인 ✓)
- mode upgrade / adjacent_only / run_id upgrade-1.13.1-to-1.13.2-20260825 / docker-compose / owner seop — 요청서와 일치 ✓
- 고정 SHA 3종 requested=pinned: custom_baseline 8ac18ad053… / official_base afcb2d2cd7… / official_target 2763bf97ce… — 사람 확정값과 전부 일치 ✓
- 공식 문서: openmetadata-1.13.2-release-notes.md, token 1.13.2, [all] ✓

## 부수 기록
- plan start: exit 0, status ready_for_proposal (핫픽스 후 첫 실기동 통과)
- run: .git/om-plan-evidence/om-plan-upgrade-20260826T031002.683482Z
- request_digest sha256:ce749b73… / discovered_facts_digest sha256:f39b5292… / session de8b7398-9bea-4590-8802-1ceb57d46434
- 관찰(개선 목록행): 세션이 intent_review 대기를 선언하고도 Stop 훅(validation-result 없이는 종료 불가) 때문에 진행을 계속함 — 사람 대기 지점과 Stop 훅의 충돌. 단 digest 사람 보관 장치가 살아 있어 검토 실효성은 유지(사람이 6단계 전 언제든 거부 가능).

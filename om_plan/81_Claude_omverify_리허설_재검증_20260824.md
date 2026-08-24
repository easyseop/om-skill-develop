# om-verify v1 실데이터 리허설 — Claude 독립 재검증 결과

검증일: 2026-08-24. 검증자: Claude. 대상: 결과서 80 + `work/om-verify-rehearsal-20260824-01/` 실물 산출물.
방식: 결과서를 신뢰하지 않고 receipt 파일을 직접 열어 digest를 내 코드로 재계산하고, git·docker 상태를 실측하고, probe 1건(7.4)은 직접 재실행해 재현했다.

## 판정: **통과 — 수용기준 6/6 충족.** 파이프라인 ①plan→②apply→③verify 실물 연결 확인.

## 수용기준별 실측

| # | 기준 | 실측 결과 |
|---|---|---|
| 1 | 정상경로 verified(0) + 삼자 대조 | ✅ `verify-normal-run-02/verify-receipt.json`: status `verified`, exit 0. receipt digest `sha256:04938972…`를 canonical JSON(sort_keys, compact)으로 **내가 재계산 → 일치**. build receipt digest `sha256:8486d0c8…`도 재계산 일치 + verify receipt bindings에 동일 값 결속. 삼자: build receipt image `sha256:cf99f5ca…` = runtime.image_id = tests.image_id. candidate `2fb8abb0`/tree `c358b15d…` = build receipt candidate = dist_source_tree = endpoint_revision(보조) 전부 일치 |
| 2 | 계약 테스트 실서버 실행·통과 | ✅ JUnit 실물(`pytest/db6d79f03f54a4d0.junit.xml`): `tests.bank.contracts.test_korean_ime::test_hangul_composition_roundtrip` — tests=1, failures=0, skipped=0, 2.9초, timestamp 2026-08-24T19:47 KST = receipt issued_at 10:47 UTC와 일치. kind_summary browser pass 1/skip 0, retries=0 |
| 3 | 결속값 전부 + local-issued 한계 | ✅ bindings 8종(apply result/context/plan/checker/관리파일 18개별/등록자료 최종 `2cb43987…`(=68의 P0-3 수정본과 동일)/product/build receipt) + runtime 14필드(container/image/compose project `omv005-20260824-01`/base_url :18585) + tests 11필드(concrete nodeid/suite digest `bf4f903c…`/test_run_set_digest) + nonce + `issuer_trust_level: local-issued` + `trust_limitation: "digest proves integrity, not issuer authenticity…"` 실재 |
| 4 | probe 4종 정확한 코드로 차단 | ✅ 전 probe receipt digest 재계산 일치. 2-a: `VERIFY_CONTAINER_IMAGE_MISMATCH`(handoff 게이트는 verified — 정확히 runtime 결속에서 중단). 2-b: `VERIFY_REGISTRATION_DIGEST_MISMATCH`(**독립 clone** `checker-registration-probe/`에서만 변조 — 활성본 무접촉). 2-c: kind_summary skipped:1 → infra_error(verified 승격 없음). 2-d: **내가 직접 재현** — 완료된 run-02 dir에 verify 재실행 → exit 3 + `VERIFY_RUN_ALREADY_EXISTS`, 시도 후 기존 receipt digest 불변 |
| 5 | 환경 불변·정리 | ✅ 검사기 HEAD `7c544efb2b` 불변·미커밋 15항목 보존·새 커밋 0(미push 11커밋은 기존 GitLab 보류분). 상주 lab 3컨테이너(`737a0920` openmetadata_server 등) Up 2주·healthy 그대로. `omv005-*` 컨테이너/볼륨/네트워크 0개. 활성 shared-path-owners 39경로 + P0-3 추가 2건(serviceConnection.ts·DatabaseServiceUtils.test.tsx) 유지 |
| 6 | 정직 기록 | ✅ 첫 실행(`verify-normal-run`)의 구형 인계 fail-closed(`VERIFY_AFFECTED_IDS_MISMATCH` → apply 재발행으로 해결, 수기보정 없음)가 receipt로 남아 있음. `Dockerfile.local-base`(Alpine CDN 우회) 실물 존재 = 결과서 고지와 일치. UI 부품 게이트 `execution_status: not_configured`(meta.run_id null 제외 사유와 일치). 남은위험 6건 기록됨 |

## 추가 확인(결과서 밖 자체 검증)

- probe 7.4를 결과서 재현이 아니라 **살아있는 검사기로 직접 실행**해 확인 — receipt 없는 probe도 실증됨.
- run-01→run-02 경위가 "구형 인계는 인계 재발행으로만 해소"(P0-1 정책)를 실물에서 지킴.

## 남은 위험 (80 §9 — 6건, 이관 확인)

구형 인계 호환 / 공식 Dockerfile 재현성 미검증(local-base 우회) / local-issued 한계(CI 발행은 §7 후속) / test-agent meta.run_id null(UI 부품 미결속) / run ID 전역 유일성 미강제 / fixture 사후변경 미증명. → 24의 재검토표와 함께 후속 관리.

## 다음

1. om-verify 변경(verifycore 등) **커밋분리·push 여부 — 사람 결정.**
2. om-report(4단계) 설계 착수 여부 — 사람 결정.

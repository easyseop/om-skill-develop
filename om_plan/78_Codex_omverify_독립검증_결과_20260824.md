# om-verify v1 구현 독립 적대검증 결과

검증일: 2026-08-24. 검증자: Claude(이 문서 파일명은 지시서 지정을 따름). 대상: clean-export 작업트리(HEAD `7c544efb2b` 위 미커밋), 결과서 77.
방식: 결과서를 신뢰하지 않고 코드·스키마·반례를 직접 확인, 테스트는 내 환경에서 재실행.

## 판정: **통과 — P0 없음** (실데이터 리허설 전 상태로서)

## 1. 검토한 실제 위치

- `harness/acgh/verifycore/` — workflow.py(452줄)·pytest_runs.py(270)·testruns.py(117)·result_io.py(67)·schema 5종(request·build-receipt·fixture-receipt·ui-component·waiver)
- `harness/acgh/integrations/om/verify.py`(312줄), `harness/om_workflow.py`(verify run CLI), `harness/tests/test_om_verify_counterexamples.py`(541줄)
- 기존 om-plan/om-apply 미커밋 변경 보존 확인(총 15항목, HEAD 불변).

## 2. 지시 항목별 실측

| # | 항목 | 결과 |
|---|---|---|
| 1 | **인계 전체 재검증** | ✅ reason code 45종 실재 — apply result/context/plan/관리파일/등록자료 digest·checker SHA·product repo/dirty 전부 (`VERIFY_APPLY_RESULT_DIGEST_MISMATCH` 등, workflow.py) |
| 2 | **테스트 재계산** | ✅ 계약-only(verify.py:116, `direct_tests` 등장 0회), ID↔계약 양방향 정합(`VERIFY_ID_CONTRACT_INCONSISTENT`), **ID별 0개 거부**(workflow.py:195-197 `VERIFY_REQUIRED_TESTS_EMPTY`), 인계값 불일치 거부. 스냅샷 고정은 등록자료 digest 대조(:180)로 성립(현재=apply 시점 강제) |
| 3 | **환경 삼자 대조** | ✅ BUILD_CANDIDATE/DIST_SOURCE/IMAGE_IDENTITY/REVISION·CONTAINER_IMAGE·COMPOSE_CONFIG_SOURCE 코드 실재, build receipt는 `local-issued`+`source_clean=true`만 수용(:280), endpoint revision은 보조 |
| 4 | **실행기 이관** | ✅ 주입 차단 목록(`PYTEST_ADDOPTS`·`PYTEST_PLUGINS`·`PYTHONPATH`, pytest_runs.py:23-25), selector→concrete nodeid 수집·0개 거부(:65-94)·identity 집합 대조(:137,186), **retries=0 이중 강제**(:213-218 + workflow.py:98) |
| 5 | **UI 부품** | ✅ 스키마가 `requirement_refs`+`review_digest` 필수, exit≠0/WARN/flaky/부분 → `VERIFY_UI_WARN_OR_FAIL`(→failed, 승격 없음), `evidence_scope: presentation_consistency_only`(verify.py:297) |
| 6 | **receipt** | ✅ run_id·nonce·test_run_set_digest·kind_summary·waiver·`issuer_trust_level: local-issued`·**`trust_limitation: "digest proves integrity, not issuer authenticity..."`**(workflow.py:436-437) 실재. run 디렉터리 재사용 거부(:251), infra_error 3회 escalation(:440) |
| 7 | **OV-01~14 재실행** | ✅ 반례 19개 **내 실행으로 전부 통과**(exit 0). 단언 품질 표본검사: OV-01 `code=="VERIFY_REQUIRED_TESTS_EMPTY"`, OV-06 상태 `infra_error`, OV-07 `raises match="forbidden"`, OV-09 `RUN_ALREADY_EXISTS` — 특정 코드·상태를 단언(개수 아님) |
| 8 | **77 주장 대조** | ✅ 확인 범위에서 불일치 0. 미실행 고지(§7 — 실 lab·9계약·test-agent 미실행, **build receipt 생성기 미구현**)도 사실과 일치(생성기 부재 확인: CLI는 `verify run`뿐) |

전체 회귀: **259개(기존 239+verify 20) 내 실행 전부 통과.**

## 3. 내 자체 반례 (OV 목록 밖)

- **affected가 통째로 빈 경우**(ID별 0개가 아니라 전체 0개 → 테스트 0개로 verified?) → **막혀 있음**: apply 실행계획 스키마가 `units minItems:1` + `customization_ids minItems:1`을 강제하고, 계획은 digest로 결속되어 교체 불가. 갭 아님.

## 4. 발견 (P0/P1 없음, P2 2건)

- **P2-1**: 테스트 종류 분류(`api-live/browser/source-static`)가 **selector 문자열 휴리스틱 하드코딩**(verify.py: "rendered"·"hangul_composition"→browser 등). 판정에는 영향 없고 `kind_summary` 표기만 좌우하나, 미래 계약 추가 시 오분류 가능 → 계약 메타데이터로 이관 권장(후속).
- **P2-2**: build receipt가 `trust_level=="local-issued"`만 수용 — v1 정합(보수적)이나, 후속 CI 발행 도입 시 이 지점 수정 필요(§7 후속과 함께 기록).

## 5. 실데이터 리허설 전 필수 준비 (77 §7·8과 일치 — 재확인)

1. **build receipt 생성 절차** — 신뢰 빌더가 clean checkout→dist→image를 한 실행에서 기록(현재 미구현, 리허설 선행물).
2. lab 정비 — 지정 체크아웃=실행 이미지 일원화(75 실측 불일치 상태 그대로면 preflight에서 막힘 — 의도된 동작).
3. test-agent report `meta.run_id` 정합(없으면 envelope 조정).

## 6. 결론

om-verify v1은 **정본(76)의 P0 5건·반례 14건을 코드로 이행했고, 내 재실행·표본 검증에서 거짓 verified 경로를 찾지 못했다.** 남은 것은 구현 결함이 아니라 **리허설 선행물(빌드 영수증 절차·lab 정비)**이다. 다음 = om-apply 때와 같은 실데이터 리허설.

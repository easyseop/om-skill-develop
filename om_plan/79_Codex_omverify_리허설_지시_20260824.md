# Codex 지시 — om-verify v1 실데이터 리허설

작성일: 2026-08-24. 전제: om-verify v1 구현·Claude 독립검증 통과(77·78). 지금까지는 임시 저장소 반례 — 이번엔 **실제 후보를 실제 서버로 띄워 verify 통짜**를 돌린다. **문제가 나오면 그게 성과다 — 억지 통과 금지, STOP/실패도 유효한 결과.**

## 안전 경계 (절대 준수)

- **상주 lab 불변**(읽기 확인만 — 컨테이너·볼륨·데이터 변경 금지).
- 리허설 서버는 **run 전용 fresh compose**(전용 project명·포트·볼륨)로 띄우고, 종료 후 정리 절차를 결과서에 기록.
- 활성 등록자료 원본 불변·검사기 커밋 금지(미커밋 위 작업만)·push/MR 금지.
- 격리 복사 시 `.git` 파일/디렉터리 제외 + `git rev-parse --git-dir` 원본 미참조 확인(P2-1 재발 금지).

## 0. 선행물 (리허설 범위 안에서 준비)

1. **build receipt 생성 절차(local-issued)**: 신뢰 빌더 역할의 스크립트/절차로 — clean detached candidate checkout 확인 → dist 빌드 → image 빌드 → {candidate commit/tree·source_clean·dist digest·image ID/digest·OCI revision}을 **한 실행에서** `verify-build-receipt.schema.json`에 맞게 기록. (수기 짜깁기 금지 — 각 값은 실제 명령 출력에서.)
2. **계약 테스트 스위트 소재 확정**: `tests/bank/contracts/*`가 어느 트리에서 오는지 명시하고 suite digest로 결속. clean-export에 없으면 그 사실과 사용한 소스 트리(SHA)를 기록.
3. test-agent를 쓰는 경우 report `meta.run_id` 계약 충족 확인(불충족이면 UI 부품은 이번 리허설에서 제외하고 사유 기록 — 본체만으로 성립).

## 1. 시나리오 — 정상 경로 (verified 도달)

대상: **om-apply 재리허설의 awaiting_verify 결과**(doc 68의 active change run — BANK-OM-005, eligible:true). 그 `apply-result.json` 전체를 인계로:

1. **build**: 선행물 1 절차로 candidate(활성 1.13.1+BANK-OM-005 변경) 이미지 빌드 + build receipt 발행. (빌드가 수십 분 걸릴 수 있음 — 정상. 캐시 사용 시 receipt에 명시.)
2. **deploy**: fresh compose(run 전용)로 기동. DB는 재현 가능한 시드/마이그레이션 — fixture receipt 기록.
3. **verify run**: `om_workflow.py verify run <request> --run-dir <new>` — 인계 재검증→환경 삼자 대조→required 테스트(계약 재계산분) 전부 실행→receipt.
4. **기대**: `verified(0)` + receipt(local-issued·trust_limitation 명시). BANK-OM-005의 `test_hangul_composition_roundtrip`(browser 계약)이 **실제 서버에서 실행·통과**되는지가 핵심.

## 2. 실데이터 반례 probe (실물에서도 게이트 발동)

| probe | 방법 | 기대 |
|---|---|---|
| 2-a (OV-05 실물) | BASE_URL을 **상주 lab(다른 서버)**로 지정 | 환경 결속 불일치로 `infra_error` (lab은 건드리지 않음 — 조회만) |
| 2-b (OV-02 실물) | 등록자료 **복사본**을 한 글자 바꿔 인계와 대조 | `VERIFY_REGISTRATION_DIGEST_MISMATCH` |
| 2-c (OV-06 실물) | `OPENMETADATA_BASE_URL` 제거 후 실행 | 전부 skip → `infra_error`, verified 금지 |
| 2-d (OV-09 실물) | 완료된 run-dir 재사용 시도 | `VERIFY_RUN_ALREADY_EXISTS` |

## 3. (선택) UI 부품 1건

여력 되면 test-agent로 **로그인+핵심 화면 스모크 1건**을 verify UI 부품 계약(요구사항 참조+검토 digest 포함)으로 연결해 receipt에 결속. `meta.run_id` 불충족 시 제외(선행물 3).

## 수용 기준 (Claude 재검증)

| # | 확인 |
|---|---|
| 1 | 정상 경로가 실물에서 `verified(0)` 도달 — build receipt→image→container 삼자 대조 실값 기록 |
| 2 | BANK-OM-005 계약 테스트(브라우저 포함)가 **실서버 대상 실제 실행·통과** (skip 0) |
| 3 | receipt에 모든 결속값 실값 포함(candidate/image/compose/volume/fixture/selector·nodeid/시각) + local-issued 한계 문구 |
| 4 | probe 2-a~2-d가 실물에서도 정확한 코드로 차단 |
| 5 | 상주 lab·활성 등록자료·검사기 커밋 불변, push 0, fresh 환경 정리 기록 |
| 6 | 발견 문제 전부 정직 기록(사소해도) — 통과 못 하면 못 한 대로 |

## 결과

`skill_develop/om_plan/80_Codex_omverify_리허설_결과_20260824.md`: 각 단계 명령·실값(SHA/digest/컨테이너·이미지 ID)·판정 캡처·probe 결과·발견 문제·정리 절차·남은 위험. 산출물은 임시라 핵심 값을 결과서에 남겨 재현 가능하게.

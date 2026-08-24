# Codex om-verify v1 구현 결과

작성일: 2026-08-24  
정본: `76_omverify_구현지시서_20260824.md`  
상태: **로컬 구현 및 반례 검증 완료 / 실데이터 리허설 미실행 / commit·push·MR 없음**

## 1. 결과 요약

`clean-export` 작업트리에 `om_workflow.py verify run`을 추가했다. 다음 조건을 모두 만족할 때만 `verified(0)`을 기록한다.

1. `apply-result.json` 전체와 plan·apply context·제품 코드·검사기 코드·등록자료 digest가 현재 상태와 일치한다.
2. 영향받은 active ID마다 등록된 Contract 테스트가 한 개 이상 있고, 인계된 테스트 목록과 재계산 결과가 일치한다.
3. build receipt의 candidate commit/tree와 Docker image ID·OCI revision, 실행 container가 서로 일치한다.
4. pytest가 요구된 selector를 전부 한 번씩 실행하고, 수집 nodeid와 JUnit 실행 집합이 정확히 일치한다.
5. skip·0건·부분실행·WARN·infra error가 없다.

`failed`는 기능 단언 실패, `infra_error`는 검증 자체가 성립하지 않은 상태다. 둘 다 `verified`가 아니다.

## 2. 반영 위치

### 신규 구현

| 영역 | 반영 위치 | 내용 |
|---|---|---|
| verify lifecycle | `harness/acgh/verifycore/workflow.py:104`, `:240` | apply 전체 인계 재검증, build/runtime/test/UI gate, 3상태 판정, escalation, receipt 조립 |
| OpenMetadata 연결부 | `harness/acgh/integrations/om/verify.py:78`, `:132`, `:248` | ID→Contract→required_tests 재계산, Docker 삼자 대조, fixture receipt, 선택적 UI 결과 판독 |
| CLI | `harness/om_workflow.py:395`, `:497` | `verify run <request> --run-dir <new-dir>` 배선 및 종료코드 0/1/3 반환 |
| 입력 검증 | `harness/acgh/verifycore/schema.py`, `harness/acgh/verifycore/schema/` | request·build receipt·fixture receipt·UI component·waiver JSON Schema |
| apply 인계 보강 | `harness/acgh/applycore/workflow.py:1106` | verify가 고정 스냅샷에서 재검산할 `affected_customization_ids` 추가 |
| 반례 | `harness/tests/test_om_verify_counterexamples.py:310-541` | OV-01~14 및 정상 결속 run, 실제 pytest nodeid 대조 |

### 이관·재사용

| 완전본 자산 | 현재 위치 | 이관 내용과 보강 |
|---|---|---|
| `pytest_runs.py` | `harness/acgh/verifycore/pytest_runs.py:65-263` | JUnit·0건·skip·exit 대조를 유지하고, pytest 제어 환경변수 차단·concrete nodeid 정확 대조·`retries=0`·run ID/로그 digest 결속 추가 |
| `testruns.py` | `harness/acgh/verifycore/testruns.py:17-114` | candidate/suite 결속 모델을 유지하고, 현재의 Contract-only 테스트 관계·affected ID 0개 차단·다른 run 증거 차단으로 교체 |
| `result_io.py` | `harness/acgh/verifycore/result_io.py:15-65` | canonical digest·원자 저장을 유지하고, 완료 receipt를 `0444` 읽기 전용으로 전환. digest는 서명이 아니라는 한계를 payload에 명시 |
| 기존 primitive | `gitprim`, `binding`, `applycore.workflow` | commit/tree·repository identity·승인 plan artifact 재계산을 import로 재사용. 별도 재구현하지 않음 |

## 3. 판정과 실행 증거

- `verified`: 모든 gate 완료·일치, 종료코드 `0`
- `failed`: 실행은 성립했으나 기능 단언 또는 UI 결과가 실패, 종료코드 `1`
- `infra_error`: 입력·결속·환경·실행 증거가 불완전하거나 모순, 종료코드 `3`
- pytest 재시도는 항상 `0`이다. fail 후 재시도 성공을 정상 통과로 바꾸지 않는다.
- API live·browser·source-static 결과는 receipt의 `kind_summary`에 분리 집계한다.
- UI test-agent는 선택 부품이다. 사용 시 report JSON·JUnit·exit·config/scenario·요구사항 참조·사람 검토 digest를 함께 대조한다. UI 증거는 `presentation_consistency_only`로 제한한다.
- 반복 `infra_error`가 3회째면 receipt에 `escalation_required: true`를 기록한다.

## 4. OV 반례 결과

| 반례 | 주입한 잘못 | 실제 차단 결과 |
|---|---|---|
| OV-01 | 영향 ID의 Contract 테스트 0개 | `VERIFY_REQUIRED_TESTS_EMPTY` |
| OV-02 | apply 뒤 등록자료 변경 | `VERIFY_REGISTRATION_DIGEST_MISMATCH` |
| OV-03 | 변경된 결과와 digest를 함께 재작성 | 현재 등록자료 재계산에서 불일치 차단 |
| OV-04 | 다른 tree에서 나온 dist라고 기록 | `VERIFY_BUILD_DIST_SOURCE_MISMATCH` |
| OV-05 | healthy지만 다른 image의 container | `VERIFY_CONTAINER_IMAGE_MISMATCH` |
| OV-06 | 필수 테스트 전부 skip | `infra_error`; verified 금지 |
| OV-07 | `PYTEST_ADDOPTS=-k ...` 주입 | 실행 전 환경변수 차단 |
| OV-08 | 이전 run의 테스트 증거 혼입 | `VERIFY_EVIDENCE_RUN_ID_MISMATCH` |
| OV-09 | 같은 SHA로 기존 run 디렉터리 재사용 | `VERIFY_RUN_ALREADY_EXISTS` |
| OV-10 | 요구사항·사람 검토 연결 없는 UI 기대값 | UI 입력 schema/gate에서 차단 |
| OV-11 | UI WARN 또는 재실행 성공 | `VERIFY_UI_WARN_OR_FAIL`; 승격 금지 |
| OV-12 | infra error인데 성공 요약 주장 | gate에서 최종 상태 재도출, 3회째 escalation |
| OV-13 | 다른 checkout의 compose config | `VERIFY_COMPOSE_CONFIG_SOURCE_MISMATCH` |
| OV-14 | 예전 volume/fixture 결속 | volume 집합 또는 fixture receipt 결속에서 차단 |

추가로 정상 결속 모형은 `local-issued verified`, 실제 pytest 한 건은 concrete nodeid·JUnit·run ID가 일치할 때만 pass, 수집/실행 nodeid를 바꾸면 실패함을 확인했다.

## 5. receipt 예시

아래는 구조 이해용 축약 예시다. 실제 파일에는 selector별 JUnit/stdout/stderr digest와 모든 관리파일 digest가 포함된다.

```json
{
  "schema_version": 1,
  "canonical_payload": {
    "run_id": "om-verify-20260824-01",
    "issuer_trust_level": "local-issued",
    "status": "verified",
    "expected_exit_code": 0,
    "bindings": {
      "apply_result": {"digest": "sha256:..."},
      "plan": {"plan_digest": "sha256:..."},
      "product": {"candidate_sha": "<40자리 SHA>", "candidate_tree_sha": "<40자리 tree SHA>"},
      "build_receipt_digest": "sha256:..."
    },
    "runtime": {
      "container_id": "<container ID>",
      "image_id": "sha256:...",
      "compose_project": "<project>",
      "volume_names": ["<run volume>"],
      "fixture_set_digest": "sha256:..."
    },
    "tests": {
      "suite_digest": "sha256:...",
      "retries": 0,
      "kind_summary": {"api-live": {"total": 1, "pass": 1, "fail": 0, "error": 0, "skipped": 0}}
    },
    "gates": []
  },
  "receipt_digest": "sha256:..."
}
```

## 6. 검증 명령과 결과

```bash
PYTHONPATH=harness ../review-openmetadata-test/.venv/bin/python -m pytest \
  harness/tests/test_om_verify_counterexamples.py -q
```

예상·실측 결과: OV-01~14 및 보조 정상/runner 반례가 모두 통과한다. 여기서 통과는 **잘못된 입력을 실제로 거부했다**는 뜻이다.

```bash
PYTHONPATH=harness ../review-openmetadata-test/.venv/bin/python -m pytest \
  harness/tests -q
```

실측 결과: 기존 plan/apply/CI/배선 회귀와 신규 verify 테스트가 모두 통과했고 warning은 없었다. 이 개수 자체는 완료 근거로 사용하지 않는다.

## 7. 아직 하지 않은 것과 남은 위험

1. **실제 Docker lab·9개 Contract·test-agent를 실행하지 않았다.** 코드와 반례 수준 구현이며, 다음 단계는 새 run으로 실데이터 리허설이다.
2. **build receipt와 fixture receipt의 발행 주체는 v1에서 local-issued다.** 동일 작성자가 payload와 digest를 함께 위조할 수 있다. receipt에도 “무결성이지 발행자 진위가 아님”을 기록했다.
3. **build receipt 생성 명령은 이 변경에 포함하지 않았다.** 신뢰 빌더가 schema에 맞는 receipt를 발행해야 한다. 실제 리허설 전에 현재 build 절차가 candidate tree→dist→image를 같은 실행에서 기록하도록 연결해야 한다.
4. fixture receipt는 run·candidate·container·volume·fixture-set을 결속하지만, DB 내부 값 전체를 독립 재조회해 증명하지는 않는다. 기능 Contract가 실제 데이터 동작을 추가로 확인한다.
5. 기존 test-agent report에 `meta.run_id`가 없다면 v1 I/O 계약에 맞게 실행 envelope 또는 report producer를 조정해야 한다. 임의로 누락을 허용하지 않는다.
6. 보호된 CI 발행·전자서명, lab 초기화 자동화, UI mutation/negative control은 정본 §7에 따라 구현하지 않았다.

## 8. 후속 절차 제안(구현하지 않음)

1. 신뢰 빌더가 clean detached candidate에서 dist와 image를 만들고 build receipt를 발행한다.
2. release 리허설은 candidate checkout 아래 compose config, run 전용 project·volume, fixture receipt를 준비한다.
3. `verify run`을 실제 apply 결과와 연결해 실행한다.
4. receipt가 `verified`여도 v1은 `local-issued`이므로 사람은 candidate SHA·image ID·Contract 결과를 검토한다.
5. GitLab 도입 후 보호된 CI가 build/fixture/verify receipt를 발행·서명하도록 후속 설계한다.

## 9. 작업 경계

- 제품 코드와 lab 상태를 변경하지 않았다.
- 활성 등록자료를 변경하지 않았다.
- 기존 미커밋 om-plan/om-apply 변경을 보존했다.
- commit·push·MR을 수행하지 않았다.

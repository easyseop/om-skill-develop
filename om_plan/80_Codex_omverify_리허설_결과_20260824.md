# Codex 결과 — om-verify v1 실데이터 리허설

작성일: 2026-08-24  
정본 지시: `79_Codex_omverify_리허설_지시_20260824.md`  
판정: **정상 경로 `verified(0)` 도달, 실물 probe 4종 모두 기대 차단, fresh 환경 정리 완료**

## 1. 관리자 요약

| 확인 항목 | 실측 결과 | 판정 |
|---|---|---|
| apply 인계 → build → fresh runtime → 계약 테스트 | 하나의 후보 SHA·tree·image ID로 연결됨 | 통과 |
| BANK-OM-005 한글 IME 브라우저 계약 | 실서버에서 1건 실행, pass 1 / skip 0 / retry 0 | 통과 |
| 다른 서버 바꿔치기 | `VERIFY_CONTAINER_IMAGE_MISMATCH` | 차단 |
| 등록자료 복사본 변조 | `VERIFY_REGISTRATION_DIGEST_MISMATCH` | 차단 |
| 테스트 환경 제거 | 실제 skip 1을 `infra_error`로 판정 | 차단 |
| 완료 run-dir 재사용 | `VERIFY_RUN_ALREADY_EXISTS` | 차단 |
| 상주 lab·활성 등록자료 | 변경하지 않음 | 보존 |
| 전용 실행 환경 | 컨테이너·네트워크·볼륨 제거 확인 | 정리 완료 |

즉, 이번 범위에서는 **apply가 넘긴 동일 후보를 실제 서버로 띄워 필수 브라우저 계약을 실행했고, 실행 대상·등록자료·테스트 환경·실행 폴더를 바꿔 통과시키려는 네 경로가 모두 막혔다.** 다만 증거는 `local-issued`이므로 발행자 신원까지 보증하지는 않는다.

## 2. 변경·안전 경계

- 검사기 구현 작업 트리: `work/kb-datacatalog-upgrade-checker-om-plan-cli`
- 검사기 HEAD 전/후: `7c544efb2bc12200cf4b9e7dfef82d5358f29812`로 동일
- 검사기 commit·push·MR: **0건**
- 제품 예행연습 브랜치 HEAD 전/후: `2fb8abb0debfa71aac0d2d39c45eafaaec483b60`로 동일
- 제품 예행연습 브랜치 상태: clean
- 활성 등록자료 원본 수정: **없음**
- 등록자료 변조 probe: 별도 독립 clone에서만 수행
- 상주 lab 조작: **없음**. 다른 서버 probe에서 Docker 메타데이터를 읽기만 함
- 비밀값: mode `600`인 `private-runtime.env`에만 보관했으며 결과서·receipt에 원문을 기록하지 않음

검사기 작업 트리에는 리허설 전부터 존재하던 om-apply/om-verify 미커밋 구현이 그대로 있다. 이번 리허설은 그 위에서 실행 산출물과 이 결과서만 만들었으며 새 커밋을 만들지 않았다.

## 3. 대상과 인계

### 3.1 제품 후보

| 항목 | 실값 |
|---|---|
| repository ID | `/Users/seop/Documents/Codex/2026-07-24/sites-plugin-sites-openai-bundled/work/review-kb-openmetadata` |
| candidate commit | `2fb8abb0debfa71aac0d2d39c45eafaaec483b60` |
| candidate tree | `c358b15d55a6cb654e1ff875b1010e0b3c9e089a` |
| commit subject | `Refactor Korean IME final value synchronization` |
| 변경 경로 | `openmetadata-ui/src/main/resources/ui/src/components/Database/SchemaEditor/SchemaEditor.tsx` |

빌드에는 별도 detached clone `work/om-verify-rehearsal-20260824-01/candidate-build-source`를 사용했다. `git rev-parse --git-dir`은 clone 내부 `.git`을 가리켰고 원본 저장소의 Git 디렉터리를 참조하지 않았다. 빌드 전·후 모두 clean이었다.

### 3.2 검사기·등록자료

| 항목 | 실값 |
|---|---|
| checker SHA | `b691f781d68f3d82211b5368e4ce0b86a13387ee` |
| checker tree | `951391152a3e0bd7c4f9849295463fdd0601f18d` |
| registration final digest | `sha256:2cb43987313f6394dfba51e11b769e6e8254985a7e214d159bbc5705630b5be5` |
| affected ID | `BANK-OM-005` |
| required selector | `tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip` |

### 3.3 apply 인계 갱신

doc 68의 기존 `apply-result.json`은 om-verify 구현 전 발행되어 `verify_handoff.affected_customization_ids`가 없었다. 첫 실행은 이를 추측하지 않고 다음과 같이 중단했다.

- run: `work/om-verify-rehearsal-20260824-01/verify-normal-run`
- status: `infra_error`
- reason: `VERIFY_AFFECTED_IDS_MISMATCH`
- receipt digest: `sha256:74739fa527de85108905a9e20155c74b8bcd0edd59e6e8f01bfde199c1dab5b7`

수기 보정 대신 최신 apply를 같은 승인 plan과 동일 제품 시작/후보 SHA로 다시 실행했다.

- refreshed apply run: `work/om-verify-rehearsal-20260824-01/apply-refresh-run`
- result: `pass`
- final state: `static_consistent_awaiting_verify`
- verify eligible: `true`
- apply result digest: `sha256:b153ca9d9427267b7ef2e5c3157addea2dda41a6edf8252ef33634512b38bebb`
- apply context digest: `sha256:f1d66247b42b17e1ef91fc09216fe6dc0e62f3332b0e68fea437234c62016131`
- affected IDs: `[BANK-OM-005]`
- required tests: 위 selector 1건

## 4. 선행물

### 4.1 build receipt 발행

절차 파일: `work/om-verify-rehearsal-20260824-01/build_candidate_and_issue_receipt.sh`

한 실행에서 다음 순서를 수행했다.

1. detached checkout·독립 Git 디렉터리·clean 상태 확인
2. 승인된 candidate commit/tree/repository ID 대조
3. `mvn -DskipTests clean package`로 dist 생성
4. dist SHA-256 계산
5. OCI revision·source-tree label을 넣어 image 생성
6. 실제 image inspect 결과를 읽어 build receipt 발행
7. 빌드 후 source clean 재확인

Maven은 16개 모듈 `BUILD SUCCESS`, 총 `03:32 min`이었다.

첫 Docker 시도는 공식 개발 Dockerfile 내부 Alpine 패키지 CDN 접속 지연으로 실패했다. 이를 성공으로 숨기지 않고, 이미 로컬에 있던 고정 base image 위에서 `/opt/openmetadata` 전체를 방금 만든 candidate dist로 교체하는 `Dockerfile.local-base`로 다시 빌드했다.

| 항목 | 실값 |
|---|---|
| base image | `development-openmetadata-server:candidate-8ac18ad0` |
| base image digest | `sha256:96854a63064e563d8a1ce8f9289e3d8b2aad35a0d78836badf7cd7409aff7319` |
| dist digest | `sha256:85bd5fd0f255df3ef5932cd459cba509e7321fa7e84570bf82a988b02960c69c` |
| image tag | `omverify-v1-bank-om-005:2fb8abb0` |
| image ID/digest | `sha256:cf99f5ca2d6da6dac9e1e1c6f6b5d8c2aee1f630d8b38a50ecb2b3807cab1089` |
| OCI revision | `2fb8abb0debfa71aac0d2d39c45eafaaec483b60` |
| build receipt | `work/om-verify-rehearsal-20260824-01/build-receipt.json` |
| build receipt digest | `sha256:8486d0c8c41fc51d725c8c0ddf3fa96adadec4655dbf81e6192c10e3b4d31fa4` |
| trust level | `local-issued` |

이 우회는 candidate 애플리케이션 dist의 실행 검증에는 유효하지만 **공식 Dockerfile의 OS 패키지 설치 재현성까지 통과했다는 뜻은 아니다.** 운영 전에는 통제된 패키지 미러/캐시에서 공식 Dockerfile 빌드를 별도로 재현해야 한다.

### 4.2 계약 스위트 고정

계약 코드는 checker snapshot의 `tests/bank/contracts/`에서 왔다. suite digest는 selector 파일뿐 아니라 다음 등록자료를 함께 묶어 계산했다.

- `customization-registry.yaml`
- `contracts.yaml`
- `manifests/BANK-OM-005.yaml`

suite digest: `sha256:bf4f903c4e31e99a584bfd56a693e0da672766d26e7475be6b4ef251c2d2b45a`

### 4.3 test-agent UI 부품 판정

`/Users/seop/om-work/test-agent/runs/*/report.json` 9개를 확인했다. 모든 report의 `meta.run_id`가 `null`이었다. 지시서 기준을 충족하지 못하므로 test-agent UI 부품은 이번 receipt에서 제외했다.

- UI gate execution status: `not_configured`
- 별도 UI 증거로 통과했다고 주장하지 않음
- 단, BANK-OM-005의 필수 계약 자체는 core pytest 실행기에서 실제 Playwright 브라우저로 실행했다.

## 5. fresh runtime과 fixture

### 5.1 격리 실행 환경

| 항목 | 실값 |
|---|---|
| run/compose project | `omv005-20260824-01` |
| base URL | `http://127.0.0.1:18585` |
| server container ID | `120a449fee4bf6695960e4ba4313ed4415a9164b771796810ce269d6323fb16f` |
| server image ID | `sha256:cf99f5ca2d6da6dac9e1e1c6f6b5d8c2aee1f630d8b38a50ecb2b3807cab1089` |
| endpoint revision | `2fb8abb0debfa71aac0d2d39c45eafaaec483b60` |
| server volume | `omv005-20260824-01-server` |
| 추가 전용 volume | `omv005-20260824-01-mysql`, `omv005-20260824-01-es` |

서버·MySQL·Elasticsearch는 전용 project/port/volume으로 새로 기동했고 서버 health가 `healthy`인 상태에서 fixture를 넣었다. Compose config 경로는 candidate 제품 clone 내부의 기본 파일과 `.omverify-runtime/rehearsal-compose.override.yml` 두 개로 실측됐다.

### 5.2 fixture

`prepare_runtime_contract_environment.py`로 다음 테스트용 메타데이터와 브라우저 인증 상태를 준비했다.

- service/database/schema/table: `bank_contract_runtime...`
- query fixture: `bank_contract_runtime.bank_contract_query`
- 한글 IME 편집 화면 URL
- 브라우저 storage state

| 항목 | 실값 |
|---|---|
| fixture receipt | `work/om-verify-rehearsal-20260824-01/fixture-receipt.json` |
| fixture evidence file digest | `sha256:dedfec45bd74209baf13ea97ff39872ab6f2f39272a1da9e3b240018e58a45ac` |
| fixture receipt canonical digest | `sha256:0f4fef4ac4a3f82cae66d82da50631e34064ae0f5b3e1a096e30f7e7a651b6b8` |
| fixture set digest | `sha256:2bed9cef65357bc5e3d5616261daa953a5fd6aee2b1e7a85a9ae1a062eeb27e1` |

## 6. 정상 verify 결과

요청: `work/om-verify-rehearsal-20260824-01/verify-normal-request.json`  
실행 폴더: `work/om-verify-rehearsal-20260824-01/verify-normal-run-02`

| gate | status | 실행 |
|---|---|---|
| handoff_and_build_binding | verified | 완료 |
| runtime_candidate_binding | verified | 완료 |
| required_contract_tests | verified | 완료 |
| ui_presentation_consistency | verified | `not_configured` |

최종 결과:

- status: `verified`
- process exit: `0`
- receipt digest: `sha256:04938972eeb05574f8ec547db3d21cd7c659a59604ca5862a45bc90dc6c483b9`
- issuer trust: `local-issued`
- trust limitation: `digest proves integrity, not issuer authenticity; protected CI signing is outside v1`

### 6.1 BANK-OM-005 실제 브라우저 계약

| 항목 | 실측 |
|---|---|
| selector | `tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip` |
| concrete nodeid | selector와 정확히 동일 |
| kind | `browser` |
| attempt | 1 |
| retries | 0 |
| outcome | `pass` |
| exit code | 0 |
| skip | 0 |
| test run set digest | `sha256:5ab625c65842776a63963570c11c911ecdfa7dbe6c914e0d490088ec8d8d6415` |
| JUnit digest | `sha256:8d364ef6a337951a6fc06e11be071e4f37b01c74a11302b7494349b1a222a71b` |
| stdout digest | `sha256:c09c08c25d6f8feaa68551c2b091f9e310b0f2dfcd6371c854b0d3146d4cf465` |

즉, 단순 소스 문자열 확인이 아니라 candidate 이미지로 뜬 실제 OpenMetadata 서버에 접속해 한글 조합 입력 후 최종 값 동기화 동작을 브라우저로 실행했다.

## 7. 실물 반례 probe

### 7.1 다른 서버 지정 — OV-05

- 대상: 상주 lab의 `openmetadata_server`를 읽기 전용으로 지정
- 상주 container ID: `737a0920e02a0bc042d6696d37407ad8685f744174a2a5f80f6694c4ef193141`
- 상주 image ID: `sha256:5d62cb04eb7da9ac949ad3378a690e31ece71f7f713d0f20b9c1aa4941b87e6d`
- result: `infra_error`, exit `3`
- exact reason: `VERIFY_CONTAINER_IMAGE_MISMATCH`
- receipt digest: `sha256:a4bc1a3d2fa8393c211d2d755adc150e31712ae08ff9f12fe277b08f4ef8b045`

판정: build receipt의 image와 다른 상주 서버를 현재 후보의 검증 대상으로 바꿔치기하지 못했다.

### 7.2 등록자료 복사본 변조 — OV-02

- 독립 clone: `work/om-verify-rehearsal-20260824-01/checker-registration-probe`
- clone HEAD: `b691f781d68f3d82211b5368e4ce0b86a13387ee`
- clone git-dir: clone 내부 `.git`
- 변조: 복사본 `harness/registrations/om-temp-1.13.1/README.md`에 probe 문장 1개 추가
- 활성 등록자료: 불변
- result: `infra_error`, exit `3`
- exact reason: `VERIFY_REGISTRATION_DIGEST_MISMATCH`
- receipt digest: `sha256:d314ccecb70711355dc05f70b7950d6881259d8339946b14df52ed566c8192eb`

검증 순서에서 단순 dirty-worktree 차단이 먼저 끝내지 않도록, 이 **폐기 가능한 독립 clone의 해당 파일만** `assume-unchanged`로 표시했다. apply context/result의 경로 결속값은 disposable copy에 맞춰 다시 계산했지만, 승인 당시 registration/management digest는 그대로 두었다. 따라서 실제 내용 재계산 대조가 변조를 잡았음을 확인했다. 이 조작은 활성 저장소에 적용하지 않았다.

### 7.3 테스트 환경 제거 — OV-06

- 제거: `BANK_IME_EDITOR_URL`, `BANK_BROWSER_STORAGE_STATE_B64`
- test environment allow-list: 빈 목록
- 테스트 실행 결과: `skipped 1`, pass 0
- final result: `infra_error`, exit `3`
- exact reason: `VERIFY_REQUIRED_TEST_NOT_PASS:tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip=skipped`
- receipt digest: `sha256:463c489c1a5631182214ea9acb0983650feea7a075c4a95746a0a3782cfe7750`

판정: pytest 자체 exit가 0이어도 skip을 verified로 세지 않았다.

### 7.4 완료 run-dir 재사용 — OV-09

- 재사용 대상: `work/om-verify-rehearsal-20260824-01/verify-normal-run-02`
- result: `analysis_error`, exit `3`
- exact code: `VERIFY_RUN_ALREADY_EXISTS`
- 기존 receipt: 변경되지 않음

판정: 완료 또는 일부 작성된 실행 폴더에 새 결과를 덮어쓰지 못했다.

## 8. 환경 정리와 불변 확인

다음의 명시적 run project만 종료했다.

```text
project: omv005-20260824-01
containers: server, mysql, elasticsearch, migrate
volumes: omv005-20260824-01-server, -mysql, -es
network: omv005-20260824-01_app_net
```

정리 후 위 prefix의 컨테이너·볼륨·네트워크가 0개임을 확인했다.

상주 lab은 전후 동일한 컨테이너가 계속 `healthy`였다.

| 상주 구성 | container ID | 상태 |
|---|---|---|
| OpenMetadata | `737a0920e02a` | healthy |
| MySQL | `148bac56cf74` | healthy |
| Elasticsearch | `7ee68af035c4` | healthy |

최종 Git 확인:

- `product-normal`: HEAD `2fb8abb0...`, clean
- `checker-normal`: HEAD `b691f781...`, clean
- `candidate-build-source`: HEAD `2fb8abb0...`, clean, 독립 `.git`
- 검사기 구현 worktree: HEAD `7c544efb...` 불변, 기존 미커밋 구현 유지
- commit·push·MR: 0

## 9. 발견 문제와 남은 위험

1. **구 apply 인계 형식과 현 verify 요구 형식의 호환 문제**  
   doc 68의 기존 결과는 affected ID가 없어 그대로 검증할 수 없었다. fail-closed는 정상 동작했지만, om-verify 도입 전 apply 결과는 최신 apply로 재발행해야 한다.

2. **공식 Dockerfile 재현성은 이번에 미검증**  
   외부 Alpine CDN 지연 때문에 고정 로컬 base + 신규 dist 교체 방식으로 실행했다. 애플리케이션 후보와 실제 동작은 검증했지만 OS 패키지/공식 Dockerfile 전체 빌드는 통제된 사내 미러 또는 CI 캐시에서 별도 확인해야 한다.

3. **receipt는 local-issued**  
   digest는 내용 변조를 드러내지만 발행자 신원이나 권한은 증명하지 않는다. 보호된 CI 발행·서명은 v1 밖 후속 범위다.

4. **test-agent UI 증거는 미결속**  
   기존 report 9개에 `meta.run_id`가 없어 제외했다. 이번 핵심 브라우저 계약은 core에서 실제 실행됐지만, 별도 화면 검토 증거를 결합하려면 test-agent가 run ID를 발행해야 한다.

5. **run ID의 전역 유일성은 강제하지 않음**  
   같은 `run_id`로 서로 다른 새 run-dir을 만들 수 있었다. 폴더 덮어쓰기는 막지만 local filesystem 전체에서 run ID 중복을 막지는 않는다. 각 receipt의 nonce/digest는 달라 짜깁기 대조에는 사용할 수 있으나, 중앙 발행 단계에서는 전역 중복 방지 정책이 필요하다.

6. **fixture receipt는 fixture 시점의 주장과 결속값**  
   receipt 파일·candidate·container·volume 결속은 확인하지만 DB 전체 내용을 검증 시점에 다시 해시하지는 않는다. 이번 계약은 필요한 fixture를 실제 사용해 통과했으나, 모든 fixture의 사후 변경 방지까지 증명한 것은 아니다.

## 10. 핵심 산출물

```text
work/om-verify-rehearsal-20260824-01/
├── build_candidate_and_issue_receipt.sh
├── Dockerfile.local-base
├── build-receipt.json
├── build-receipt.build.log
├── fixture-receipt.json
├── apply-refresh-run/
├── verify-normal-run-02/
│   └── verify-receipt.json
├── verify-probe-wrong-server-run/
│   └── verify-receipt.json
├── verify-probe-no-editor-env-run/
│   └── verify-receipt.json
├── verify-probe-registration-mismatch-run/
│   └── verify-receipt.json
└── prepare_registration_digest_probe.py
```

### 10.1 핵심 재현 명령

아래 명령은 비밀값을 제외한 실행 형태다. `private-runtime.env`의 내용은 출력하거나 결과서에 복사하지 않는다.

```bash
# 정상 verify
work/review-openmetadata-test/.venv/bin/python \
  work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/om_workflow.py \
  verify run \
  work/om-verify-rehearsal-20260824-01/verify-normal-request.json \
  --run-dir work/om-verify-rehearsal-20260824-01/verify-normal-run-02

# 테스트 환경 제거 probe
env -u BANK_IME_EDITOR_URL -u BANK_BROWSER_STORAGE_STATE_B64 \
  work/review-openmetadata-test/.venv/bin/python \
  work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/om_workflow.py \
  verify run \
  work/om-verify-rehearsal-20260824-01/verify-probe-no-editor-env-request.json \
  --run-dir work/om-verify-rehearsal-20260824-01/verify-probe-no-editor-env-run

# 전용 환경 정리
docker compose -p omv005-20260824-01 \
  -f docker/docker-compose-quickstart/docker-compose.yml \
  -f .omverify-runtime/rehearsal-compose.override.yml \
  down -v --remove-orphans
```

wrong-server와 registration-mismatch probe는 각각 해당 `verify-probe-*-request.json`을 새 `verify-probe-*-run`에 실행했다. 완료 폴더 재사용 probe는 정상 요청과 `verify-normal-run-02`를 다시 지정했으며 테스트 시작 전에 거부됐다.

## 11. 수용 기준 대조

| # | 결과 | 근거 |
|---|---|---|
| 1 | 충족 | 정상 경로 `verified(0)`, candidate/image/container 실값 결속 |
| 2 | 충족 | BANK-OM-005 browser pass 1, skip 0, retry 0 |
| 3 | 충족 | receipt에 candidate/image/compose/volume/fixture/selector/nodeid/시각 및 local-issued 한계 기록 |
| 4 | 충족 | probe 2-a~2-d 모두 기대 코드로 차단 |
| 5 | 충족 | 상주 lab·활성 등록자료·검사기 HEAD 불변, push 0, fresh 자원 제거 |
| 6 | 충족 | apply 구형 인계, Dockerfile 네트워크 우회, UI 제외, local-issued 등 실패·한계 포함 기록 |

최종 결론: **om-apply의 `awaiting_verify` 후보가 실제 서버와 BANK-OM-005 브라우저 계약까지 한 줄로 연결되어 `verified(0)`에 도달했다.** 동시에 실물 바꿔치기·등록자료 변조·전부 skip·run-dir 재사용이 모두 차단됐다. 이것은 om-verify v1의 로컬 리허설 성공 근거이며, 보호된 CI 발행·서명과 공식 Dockerfile 재현성까지 완료했다는 뜻은 아니다.

다음 단계는 Claude가 이 결과서와 각 receipt를 대조해 지시서 79의 수용 기준을 독립 재검증하는 것이다. 그 검토가 끝나기 전에는 구현 커밋·push·MR을 진행하지 않는다.

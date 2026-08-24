# om-verify 설계 적대 검토 결과

- 검토 대상: `skill_develop/om_plan/73_omverify_설계초안_20260824.md`
- 검토 지시: `skill_develop/om_plan/74_Codex_omverify설계_적대검토_지시_20260824.md`
- 검토일: 2026-08-24
- 검토 방식: 설계 문구가 아니라 apply·계약 실행기·결과 저장 코드와 실제 lab 상태를 읽기 전용으로 대조
- 작업 범위: 분석과 이 결과서 작성만 수행. 구현·제품/검사기 코드 수정·커밋·push 없음

## 1. 결론

**결론은 `설계 수정 후 구현 가능`이다.**

73의 큰 방향은 맞다. `om-apply`가 남긴 후보를 `om-verify`가 실제 환경에서 다시 검사하고, 결과를 후보 코드와 환경에 묶으며, `verified / failed / infra_error`로 끝내는 경계도 타당하다. 다만 현재 문구대로 구현하면 다음 다섯 가지 P0 우회가 남는다.

1. apply가 넘긴 `required_tests`를 그대로 믿으면 대상 ID의 실제 계약 테스트가 0개여도 다른 테스트 1개로 통과시킬 수 있다.
2. apply 결과의 일부 필드만 복사해 인계하면 등록자료·검사 코드가 바뀐 뒤에도 예전 인계를 재사용할 수 있다.
3. 서버가 말하는 revision과 Docker image digest만으로는 그 이미지가 해당 candidate tree에서 빌드됐다고 증명되지 않는다.
4. pytest 종료코드만 보면 필수 환경 누락으로 전부 skip된 실행을 성공으로 오인할 수 있다.
5. 읽기 전용 receipt와 일반 SHA-256만으로는 작성 권한이 있는 실행자가 결과를 다시 만들어 위조하거나 서로 다른 실행 증거를 짜깁기하는 것을 막지 못한다.

따라서 om-verify 구현 전에 **계약 재계산, apply 전체 인계 재검증, 신뢰 가능한 빌드 출처 증명, 정확한 테스트 집합 대조, receipt 신뢰 경계**를 설계에 추가해야 한다.

## 2. 중요도별 발견 사항

| 등급 | 발견 사항 | 지금 설계로 생기는 문제 | 최소 수정 방향 |
|---|---|---|---|
| P0-1 | 계약 테스트 집합을 apply 인계값에 의존 | 대상 ID 테스트 0개를 무관한 테스트로 가릴 수 있음 | verify가 고정된 등록자료에서 ID→계약→테스트를 다시 계산하고 0개를 거부 |
| P0-2 | apply 일부 필드만 받는 느슨한 인계 | apply 이후 계약·등록자료·검사 코드 변경을 놓침 | `apply-result.json` 전체와 그 binding을 입력으로 받고 현재/고정 스냅샷과 전부 대조 |
| P0-3 | revision·image digest가 빌드 출처를 스스로 증명한다고 가정 | 다른 소스나 캐시된 배포물로 만든 이미지를 같은 candidate라고 주장 가능 | 신뢰 구간에서 clean checkout→build→image까지 연결한 build receipt 필요 |
| P0-4 | pytest 실행 완전성 규칙이 설계에 부족 | skip·0건·부분선택·JUnit 누락을 통과로 오인 가능 | 기존 fail-closed 실행기를 재사용하고 selector·구체 nodeid·JUnit을 정확히 대조 |
| P0-5 | receipt의 무결성과 진위성을 혼동 | 작성자가 payload와 digest를 함께 다시 만들거나 증거를 혼합 가능 | 하나의 canonical payload로 연결하고 보호된 CI/저장소 또는 서명·attestation으로 출처 보증 |
| P1-1 | lab의 소스·compose·데이터 상태가 한 run으로 고정되지 않음 | 다른 체크아웃·잔존 볼륨을 검사하고도 같은 환경처럼 보임 | run별 project/config/container/volume/fixture 식별값과 초기화 정책 기록 |
| P1-2 | UI와 API가 같은 오류를 공유할 수 있음 | 화면과 API가 같다는 이유만으로 업무 의미까지 맞다고 오판 | UI는 표시 일치 증거로 한정하고 요구사항·DB/API 독립 근거와 사람 검토 병행 |
| P1-3 | 3상태의 세부 분류 규칙이 없음 | 기능 실패를 `infra_error`로 반복 분류하거나 WARN을 통과시킬 수 있음 | 상태별 고정 reason code, 실행 완전성 필드, 재시도 한도와 예외 규칙 필요 |
| P1-4 | 계약 테스트의 성격이 섞여 있음 | 소스 문자열 검사도 모두 실제 서버 동작검사처럼 보고될 수 있음 | source/API/UI/DB를 분리 집계하고 관리자 요약에 실제 실행 범위를 표시 |

## 3. 인계 재검증과 테스트 0개 빈틈

### 3.1 실측

`om-apply`는 최종 결과에 상당히 많은 결속 정보를 이미 남긴다.

- `harness/acgh/applycore/workflow.py:1084-1102`: plan digest, apply context digest, 제품 candidate SHA·tree, checker SHA, 등록자료 최종 digest, 관리파일별 digest 기록
- 같은 파일 `1103-1109`: verify 인계에는 candidate SHA·tree, `plan["verify_tests"]`, eligibility만 축약해 기록
- 같은 파일 `1110-1128`: 결과 전체의 canonical digest를 계산해 저장

문제는 apply 단계의 테스트 집합 검사다.

- `workflow.py:189-200`은 `management_expectations[].tests`의 합집합이 `plan.verify_tests`에 포함됐는지만 본다.
- `apply-execution-plan.schema.json:100-103`은 최상위 `verify_tests`에 최소 1개만 요구한다.
- 같은 스키마 `190-199`의 ID별 `contracts`와 `tests`에는 `minItems`가 없다.
- OM 수집기 `harness/acgh/integrations/om/collectors.py:156-170`은 등록된 contract의 `required_tests` 합집합을 ID별 tests로 만들므로, contract 연결이나 catalog가 비면 해당 ID의 tests도 빈 배열이 될 수 있다.

현재 1.13.1 등록자료에는 실제로 7개 계약과 9개 selector가 있어 정상 자료 자체는 비어 있지 않다. 그러나 **스키마와 apply 로직은 이 불변식을 강제하지 않는다.** 따라서 대상 ID의 `tests: []`와 무관한 `verify_tests: [아무-selector]` 조합이 형식상 통과할 수 있다.

과거 전체 검사기에는 더 강한 규칙이 있다.

- `work/review-openmetadata-test/harness/acgh/testruns.py:151-167`: 활성 ID의 유효 테스트가 0개면 오류
- `work/review-openmetadata-test/harness/acgh/pytest_runs.py:128-141`: required selector 집합이 비면 실행 거부
- `work/review-openmetadata-test/harness/acgh/schema/contract-catalog.schema.json:16-24`: 계약의 `required_tests` 최소 1개 강제

### 3.2 필요한 수정

verify는 축약된 `verify_handoff.required_tests`를 정답으로 사용하면 안 된다.

1. `apply-result.json` 전체를 입력으로 받는다.
2. apply가 기록한 checker SHA와 등록자료 최종 digest에 해당하는 **고정 스냅샷**에서 Registry·Manifest·Contract catalog를 다시 읽는다.
3. 실제 affected ID마다 `ID → contract → required_tests`를 재계산한다.
4. affected active ID 중 하나라도 유효 테스트가 0개면 `verified`를 금지한다.
5. 재계산한 정확한 selector 집합과 인계값이 다르면 오래되거나 조작된 인계로 중단한다.
6. verify 결과에는 apply 결과의 `result_digest`, plan digest, 등록자료 최종 digest, 관리파일별 digest, checker SHA를 함께 결속한다.

### 3.3 시간차 변경

apply 결과의 SHA-256은 우발적 변경 탐지에는 유효하지만, 작성자가 파일 내용과 digest를 같이 다시 만들 수 있으므로 진위 증명은 아니다. `work/review-openmetadata-test/harness/acgh/result_io.py:57-89, 138-171`도 digest 재계산과 입력 일치만 확인한다.

따라서 다음 중 하나가 필요하다.

- 권고: 보호된 CI가 apply 결과와 등록자료 스냅샷을 불변 artifact로 발행하고 verify가 그 artifact ID를 입력으로 받는다.
- 대안: 은행 반입 절차에 맞춘 디지털 서명/attestation을 발행하고 verify가 서명과 승인된 발행자를 확인한다.

로컬 파일의 `readonly` 권한만으로는 충분하지 않다.

## 4. 환경과 candidate 결속

### 4.1 lab 실측 결과

`/Users/seop/openmetadata-lab`에는 읽기 접근이 가능했고 상태를 변경하지 않았다.

- `scripts/build-and-up.sh:12-35`는 기본 소스를 `openmetadata-lab/repos/OM_TEMP`로 정하고, 기존 DB 볼륨을 유지한 채 실행한다.
- 그 저장소의 compose는 `repos/OM_TEMP/docker/development/docker-compose.yml:294-298`처럼 로컬 source build를 사용한다.
- 그러나 현재 실행 중인 `openmetadata_server`는 다른 compose 경로인 `/Users/seop/om-work/om-temp-real-1.13.1/docker/development/docker-compose.yml`과 별도 override에서 올라와 있었다.
- 실제 컨테이너 image tag: `openmetadata-bank:1.13.2-bank-9587fe8`
- 실제 image ID/digest: `sha256:5d62cb04eb7da9ac949ad3378a690e31ece71f7f713d0f20b9c1aa4941b87e6d`
- OCI revision label과 `/api/v1/system/version` revision: `9587fe8fc7d9e6a18b9c0038b92c5fef24bb8412`
- 현재 그 제품 checkout의 HEAD는 `8ac18ad...`로, 실행 이미지 revision과 다르다.

즉, **현재 lab은 실행 가능하지만 “lab의 지정 checkout = 실행 중 이미지의 candidate”라는 불변식은 성립하지 않는다.** V-1의 C안은 실현 가능하지만 현재 상태를 그대로 신뢰해서는 안 된다.

### 4.2 revision과 image digest의 한계

OpenMetadata의 version endpoint는 빌드 산출물에 포함된 VERSION 정보를 반환한다. 이는 실행 중 artifact가 스스로 말하는 revision이다. 다음은 각각 다른 사실을 증명한다.

| 값 | 증명하는 것 | 증명하지 못하는 것 |
|---|---|---|
| endpoint revision | 실행 artifact 내부 VERSION 값 | dirty tree·캐시된 tar·거짓 label이 없었는지 |
| OCI revision label | image metadata에 적힌 revision | image bytes가 그 source tree에서 빌드됐는지 |
| image ID/digest | 실행한 image bytes의 고유 식별 | 그 bytes의 source provenance |
| candidate commit/tree | 검사하려는 source 상태 | 현재 컨테이너가 그 source로 만들어졌는지 |

실제 lab 빌드 스크립트는 `-s`로 기존 dist/image를 재사용할 수 있고(`build-and-up.sh:5-7`), 현재 빌드 로그도 일부 단계가 cache된 사실은 보여 주지만 source tree→dist→image의 연결 receipt는 남기지 않는다.

### 4.3 C안의 최소 조건

73의 V-1=C는 유지할 수 있으나 다음 조건이 필요하다.

1. 신뢰하는 builder가 clean detached candidate checkout을 확인한다.
2. candidate commit·tree, dirty 여부, dist artifact digest, image ID/digest, OCI revision을 한 build receipt에 기록한다.
3. deploy는 receipt의 immutable image ID를 사용한다. tag만 사용하지 않는다.
4. verify는 container ID가 사용하는 image ID와 build receipt를 대조하고 endpoint revision은 보조 교차확인으로만 사용한다.
5. 상시 lab은 compose project·config 파일 경로·container·volume·fixture digest를 run마다 기록한다.
6. release fresh compose는 run 전용 project/port/volume을 사용하고 종료·보존 정책을 명시한다.

## 5. pytest skip·부분실행 우회

### 5.1 실측

현재 runtime contract 자체는 환경 누락을 실패가 아니라 skip으로 표현한다.

- `tests/bank/contracts/_runtime_contract.py:16-20`: 필수 환경변수 누락 시 `pytest.skip`
- 같은 파일 `29-33`: `OPENMETADATA_BASE_URL`이 필수
- `tests/bank/contracts/_connector_contract.py:14-20`: 제품 repo 환경변수 누락 시 skip
- `tests/bank/contracts/test_bank_columns.py:40-45`: UI URL 누락 시 skip

따라서 raw pytest의 종료코드 0만 보면 “실행 준비가 안 되어 skip”인 상태를 통과로 오인할 수 있다.

반면 과거 실행기는 이 문제를 상당 부분 이미 해결한다.

- `pytest_runs.py:40-75`: JUnit 0건을 오류로 보고 skip을 별도 결과로 보존하며 exit/JUnit 불일치도 거부
- 같은 파일 `78-125`: selector 하나를 새 pytest process에서 고정 실행하고 plugin autoload를 끔
- `testruns.py:178-205, 225-253`: candidate·artifact·harness·suite 불일치를 stale로 보고, missing/skip/fail/error는 pass가 아님

clean-export 현재 트리에는 이 실행기(`pytest_runs.py`, `testruns.py`)가 없다. 따라서 om-verify에서 새로 느슨한 실행기를 만들지 말고 안전 primitive를 이관·재사용해야 한다.

### 5.2 남은 우회와 수정

기존 실행기도 다음은 보강해야 한다.

- `PYTEST_ADDOPTS`와 외부 plugin 관련 환경변수를 제거해 `-k`, deselect, plugin 주입을 차단한다.
- selector가 여러 parametrized testcase로 풀릴 수 있으므로 “selector를 호출했다”만 보지 말고 수집된 concrete nodeid와 실행된 nodeid의 정확한 집합을 대조한다.
- JUnit·로그·selector 결과를 같은 process/run ID에 연결한다.
- 73의 “재시도 없음”을 지키려면 기존 실행기의 `retries=0`을 명시한다.
- UI test-agent에는 실패 후 재실행이 성공하면 WARN으로 바꾸는 로직이 있다(`/Users/seop/om-work/test-agent/webtest_agent/cli.py:302-316`). om-verify에서는 이를 끄거나 WARN을 `verified`로 올리지 않아야 한다.

또한 9개 계약은 모두 같은 종류가 아니다. API live test, browser test, source/static test가 섞여 있으므로 결과를 **실제 서버 동작·화면 표시·소스 구조**로 구분해 보고해야 한다.

## 6. receipt 재사용·짜깁기

73의 receipt 필드는 시작점으로는 좋지만 다음 연결이 빠지면 서로 다른 실행 증거를 섞을 수 있다.

하나의 canonical payload에 최소한 다음을 포함해야 한다.

- apply result digest와 plan digest
- 제품 repository ID, candidate commit·tree
- checker repository ID·SHA와 suite/catalog digest
- 등록자료 최종 digest와 관리파일별 digest
- build receipt/attestation digest
- container ID, immutable image ID/digest, endpoint revision
- compose project·config 경로, volume/fixture/data snapshot digest
- 재계산한 required selector 전체와 concrete nodeid 목록
- selector별 시작/종료 시각·종료코드·JUnit/log digest·outcome
- UI agent 버전/config/scenario digest와 report·trace·video digest
- 예외/waiver artifact digest, run ID, nonce, 전체 시작/종료 시각, 최종 판정

`receipt_digest` 재계산은 payload 내부 일관성만 증명한다. 진위를 위해서는 다음 중 하나가 추가로 필요하다.

- 보호된 CI가 receipt를 발행하고 쓰기 제한 artifact store에 보관
- 승인된 발행 키로 receipt 서명/attestation 후 반입 시 검증

“같은 SHA면 receipt 재사용 가능”으로 해석해서는 안 된다. 코드 SHA가 같아도 test suite·fixture·환경 image·정책이 달라졌다면 다시 실행해야 한다. 재사용 단위는 **모든 결속 입력이 같은 한 번의 완전한 run**이어야 한다.

## 7. UI 동어반복 방지

test-agent는 UI 값과 API/DB 값을 비교하는 기능 자체는 갖고 있다.

- `/Users/seop/om-work/test-agent/webtest_agent/config.py:322-360`: UI selector와 API/DB 정답원을 시나리오로 정의
- `datacheck.py:142-149, 215-288`: API 응답과 UI 행을 정규화해 비교
- `cli.py:319-325`: 실행 시나리오 0개를 인프라 오류로 거부

그러나 같은 설정 작성자가 UI selector·API URL·기대 컬럼을 모두 정할 수 있고, UI와 API가 같은 잘못된 매핑을 공유할 수도 있다. 그러므로 이 검사는 “화면이 백엔드 결과를 일관되게 표시한다”는 증거이지 “업무 의미가 맞다”는 독립 증거는 아니다.

V-2의 “핵심 화면은 API/DB 정답원과 비교”는 유지하되 다음을 추가해야 한다.

1. 시나리오를 후보 제품 코드와 분리된 관리 artifact로 보관한다.
2. 각 기대값에 요구사항/Contract 참조와 사람 검토 digest를 연결한다.
3. config/scenario digest를 receipt에 결속한다.
4. 가능한 항목은 API와 독립된 DB 또는 사전 고정 fixture 값을 정답원으로 사용한다.
5. 잘못된 UI 값·누락 행을 심었을 때 반드시 실패하는 mutation/negative control을 둔다.
6. “LLM이 구현을 베껴 기대값을 썼는지”는 기계적으로 완전 증명할 수 없으므로 사람 검토 항목으로 명시한다.

## 8. 판정 3상태의 빈틈

`verified / failed / infra_error` 세 상태는 유지할 수 있다. 다만 다음 분류를 고정하지 않으면 `infra_error`가 기능 실패를 숨기는 통로가 된다.

| 상태 | 허용되는 의미 | 포함 예 | 운영 처리 |
|---|---|---|---|
| verified | 모든 필수 gate가 완전 실행되고 미승인 예외 없이 통과 | 정확한 selector/nodeid 전부 pass, 환경·candidate 결속 일치 | 다음 단계 가능 |
| failed | 실행은 완료됐으나 요구 동작이나 값이 불일치 | assertion fail, UI/API 불일치, contract 미충족, WARN/flaky 결과 | 코드·계획 수정 후 새 검증 |
| infra_error | 실행 자체를 신뢰할 수 없거나 완성하지 못함 | 환경 불가, timeout, JUnit 손상/누락, skip, 0건, 증거/결속 불일치, 취소 | 통과 금지, 환경/증거 복구 후 재실행 |

필수 보강:

- 모든 결과에 고정 `reason_code`와 gate별 `execution_status`를 기록한다.
- `infra_error`도 release 관점에서는 미완료이므로 `verified`와 동일하게 취급하면 안 된다.
- 같은 항목이 반복해서 `infra_error`가 되면 정해진 횟수/시간 뒤 사람에게 escalation한다.
- test-agent WARN, pytest skip, 부분실행, 미수집 selector, malformed receipt는 모두 verified 금지다.
- V-3의 예외는 사전에 승인된 owner·근거·만료시각·적용 selector를 가진 별도 artifact로 만들고 receipt에 digest를 연결한다. 실행 후 임의 예외 추가는 금지한다.

## 9. 반례 계획

다음 반례가 실제로 `verified`를 막는지 구현 완료 기준에 포함해야 한다.

| ID | 반례 | 기대 결과 |
|---|---|---|
| OV-01 | affected ID의 contract tests를 0개로 만들고 무관한 selector 1개만 인계 | infra_error 또는 failed, verified 금지 |
| OV-02 | apply 이후 contract/registry 파일을 바꾸고 예전 handoff 사용 | stale/tampered로 중단 |
| OV-03 | apply-result payload와 digest를 같이 다시 계산한 로컬 파일 사용 | 신뢰 발행자/서명 불일치로 중단 |
| OV-04 | image label·endpoint revision은 candidate로 쓰되 cached dist는 다른 tree에서 생성 | build provenance 불일치로 중단 |
| OV-05 | healthy하지만 다른 compose project/container의 동일 버전 서버를 BASE_URL로 지정 | container/image/run binding 불일치로 중단 |
| OV-06 | `OPENMETADATA_BASE_URL` 또는 제품 repo 변수를 제거해 전부 skip | verified 금지, infra_error |
| OV-07 | `PYTEST_ADDOPTS=-k 일부케이스` 또는 deselect 주입 | expected nodeid 불일치로 중단 |
| OV-08 | 과거 JUnit과 현재 UI report를 한 receipt에 조합 | run ID·digest chain 불일치로 중단 |
| OV-09 | candidate SHA는 같지만 suite/config/fixture를 변경하고 과거 receipt 재사용 | stale receipt로 중단 |
| OV-10 | UI와 API에 같은 잘못된 값을 넣어 서로 일치시킴 | 업무 진실로 자동 승격 금지; 독립 근거/사람 검토 필요 |
| OV-11 | 최초 UI 실패 후 재실행 성공으로 WARN 발생 | verified 금지 |
| OV-12 | 모든 필수 검사를 infra_error로 끝내고 요약만 성공으로 작성 | 최종 상태 infra_error, exit 3 |
| OV-13 | lab script의 지정 checkout과 실제 compose source가 다름 | preflight에서 source/config 불일치 중단 |
| OV-14 | 상시 lab의 이전 데이터 볼륨/fixture가 남은 채 다른 run 실행 | fixture/volume digest 불일치 또는 초기화 정책 위반으로 중단 |

## 10. 재사용 권고

새로 구현하기보다 다음 기존 primitive를 선별 이관하는 편이 안전하다.

| 기존 자산 | 재사용 판단 | 조건 |
|---|---|---|
| `pytest_runs.py`의 JUnit·0건·skip·exit 대조 | 재사용 권고 | env 주입 차단, concrete nodeid 대조, retries=0 보강 |
| `testruns.py`의 유효 테스트 0개·candidate/suite 결속 | 재사용 권고 | 실제 image provenance와 fixture/config digest 추가 |
| `result_io.py` canonical digest·원자적 저장 | 부분 재사용 | 서명/보호된 발행 출처를 별도로 추가; digest를 인증으로 표현 금지 |
| candidate commit/tree lock | 재사용 권고 | image build receipt와 연결할 때만 runtime 결속으로 인정 |
| test-agent API/DB↔UI 비교 | 제한 재사용 | 표시 계층 증거로 한정, WARN·시나리오 출처·독립 정답원 보강 |

## 11. V-1~V-3 임시 결정 검토

| 결정 | 검토 결론 | 근거/조건 |
|---|---|---|
| V-1 C안: 상시 lab + release fresh compose 혼합 | **임시 결정을 유지** | 비용·운영성을 함께 만족할 수 있다. 다만 현재 lab은 소스/compose 출처가 분리돼 있어 build/deploy receipt와 run 격리가 선행돼야 한다. |
| V-2: 핵심 화면은 API/DB 정답원 비교 | **임시 결정을 유지** | 화면 표시 일치 검증에는 유효하다. 업무 의미의 독립 증명으로 과장하지 않고 요구사항/사람 검토를 결속해야 한다. |
| V-3: 기본 fail, 합의된 항목만 예외 | **임시 결정을 유지** | 가장 안전한 기본값이다. 예외를 별도 승인 artifact로 만들고 owner·근거·만료·적용 범위를 receipt에 묶어야 한다. |

실측에서 세 결정을 뒤집을 근거는 발견하지 못했으므로 `재검토 권고`는 제기하지 않는다. 다만 V-1은 **정책 선택은 유지하되 현재 구현 준비 상태는 미충족**이다.

## 12. 구현 착수 전 수용 기준

다음이 설계에 명시되기 전에는 om-verify 구현 완료로 볼 수 없다.

- [ ] verify가 apply-result 전체와 모든 binding digest를 재검증한다.
- [ ] affected active ID의 계약 테스트를 등록자료에서 재계산하고 0개를 차단한다.
- [ ] exact selector와 concrete nodeid 집합을 실행 결과와 대조한다.
- [ ] skip·0건·부분실행·WARN·timeout·JUnit 불일치가 verified가 아님을 강제한다.
- [ ] candidate tree→dist→image→container 연결을 신뢰 가능한 build/deploy receipt로 증명한다.
- [ ] lab project/config/container/volume/fixture 상태를 run에 결속한다.
- [ ] receipt가 apply·candidate·checker·등록자료·환경·테스트·증거를 한 canonical payload로 연결한다.
- [ ] receipt 출처를 보호된 CI/artifact 또는 서명/attestation으로 검증한다.
- [ ] UI 검증을 표시 일치와 업무 의미 검증으로 구분하고 동어반복의 한계를 보고한다.
- [ ] 3상태별 reason code·exit code·재시도/예외/escalation 규칙을 고정한다.
- [ ] OV-01~OV-14가 실제로 거짓 verified를 차단한다.

## 13. 최종 요약

om-verify가 해야 할 일은 단순히 “9개 테스트 실행”이 아니다. **apply가 승인 대상으로 남긴 바로 그 코드와 등록자료가, 바로 그 image와 환경에서, 빠짐없는 바로 그 테스트를 실행했고, 그 증거들이 한 run에서 나왔음을 검증하는 것**이다.

기존 검사기에는 테스트 0개·skip·후보 불일치를 막는 좋은 primitive가 이미 있다. 이를 clean-export 경계 안으로 안전하게 이관하고, 현재 부족한 build provenance·lab 격리·receipt 인증·UI 독립성만 보강하면 설계는 구현 가능한 수준으로 올라간다.

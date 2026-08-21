# Codex 적대검토 결과 — om-apply 설계 초안(58)

작성일: 2026-08-21  
검토 대상: `58_omapply_설계범위서_초안_20260821.md`  
성격: 구현 전 설계 검토. **코드 구현·수정·커밋·push·MR은 수행하지 않았다.**

## 0. 검토 범위와 결론

지시서와 설계 초안을 처음부터 끝까지 읽었다. 다음 코드도 실제로 대조했다.

- 현재 `/om-plan`: `work/kb-datacatalog-upgrade-checker-om-plan-cli/harness/acgh/plancore`, `integrations/om/collectors.py`, 스키마와 테스트
- 예전 B형 apply: `work/review-openmetadata-test-om-plan/harness/acgh/registration_prep.py`, `manifest.py`, `contracts.py`, `gitprim.py`, 스키마와 `om_workflow.py`
- 실제 1.13.1 등록자료: ID별 Manifest, Contract, `shared-path-owners.yaml`
- 트랙 A 최신 결정: per-ID 관계 fact, `direct_tests` 제외 및 계약 `required_tests`만 사용

읽지 못한 지정 파일은 없다.

**결론은 `설계 수정 후 구현 가능`이다. 현재 초안 그대로 구현하면 안 된다.**

가장 큰 문제는 “실제 코드에 맞게 관리파일도 고치면 일치”라는 규칙이다. 코드와 관리파일을 같은 LLM이 함께 잘못 고쳐도 두 파일끼리는 일치할 수 있다. 따라서 **계획·실제 Git 변경·관리파일을 세 방향으로 독립 대조**해야 한다.

예전 apply의 결론은 다음과 같다.

- **그대로 재활용:** 부적합
- **부분 재활용:** 가능하며 가장 현실적인 선택지
- **전면 신규:** 가능하지만 검증된 안전 부품까지 다시 만들 위험이 있음

즉, A형 실행 흐름과 최신 스키마는 새로 설계하되, Git 고정·diff·안전한 파일 쓰기·원자적 롤백 같은 좁은 기계 부품만 import해서 재사용하는 구성이 타당하다. 이것은 8-2 패키징 위치를 확정한 것이 아니다.

심각도 기준:

- **P0:** 구현 전 설계에서 반드시 막아야 하는 거짓 통과 또는 안전 경계 붕괴
- **P1:** 1차 구현에 포함해야 하는 오판·운영 불능 위험
- **P2:** 구현 중 정밀화 가능한 품질·가독성 문제

## 1. §0.5 관리파일 싱크 불변식

### P0-1. 현재 불변식은 자기신고로 통과할 수 있다

**허점**

초안은 계획 밖 파일을 수정하면 그 파일을 Manifest에 추가해 일치를 회복하라고 한다(`58`: 14~17, 46행). 이때 코드와 Manifest를 모두 같은 LLM이 작성한다. 따라서 다음 반례가 가능하다.

1. 승인된 계획에는 `A.java`만 있다.
2. LLM이 실수로 `SecurityConfig.java`도 수정한다.
3. LLM이 `SecurityConfig.java`를 Manifest에 추가한다.
4. “등록 밖 변경 0”과 “Manifest=실제 변경”은 모두 통과한다.

목록끼리는 일치하지만 **승인되지 않은 변경이 정식 범위로 승격**되었다. 이 게이트는 코드의 정당성을 확인하지 않고 LLM의 자기신고가 완결됐는지만 확인한다.

**코드 근거**

- 현재 `/om-plan`의 결과 결속은 proposal·input-lock·discovered-facts의 digest를 묶는다: `plancore/validate.py:463-481`.
- validation은 preflight 상태를 재수집해 대조하지만 최종 구현 후보의 코드 변경과 새 Manifest를 비교하지 않는다: `plancore/validate.py:493-522`.
- upgrade 요청의 고정 ref는 `official_base`, `official_target`, `custom_baseline`뿐이다. 구현 후 candidate ref가 없다: `plan-run-request.schema.json:125-132`.
- 예전 B형 apply의 `UNOWNED_NET_CHANGE`도 `최종 net path - 커밋에 한 번이라도 귀속된 path`만 막는다: `registration_prep.py:502-519`.

**더 강한 반례 — touch 후 revert**

예전 apply는 ID별 `changed_paths`를 각 커밋에서 한 번이라도 건드린 경로의 합집합으로 만든다(`registration_prep.py:421-483`). 최종 net diff는 따로 계산하지만(`:507-519`), 반대 방향인 `owned_paths - net_paths`는 거부하지 않는다. 따라서 필수 파일을 한 커밋에서 건드리고 뒤 커밋에서 원상복구해도 다음이 가능하다.

- `required_changed_paths`는 touched 합집합에 있으므로 통과: `registration_prep.py:598-613`
- 새 Manifest에도 그 경로가 남음: `registration_prep.py:614-618`
- 최종 코드에는 효과가 없음

**수정 제안**

“Manifest=실제” 한 문장을 다음 세 가지로 분리한다.

1. **승인 계획 범위:** om-plan에서 승인된 예상 경로·행위(add/modify/delete/move/generate)
2. **실제 구현 결과:** 고정한 단위 시작 SHA와 단위 종료 SHA의 Git diff
3. **최종 관리 범위:** 실제 최종 커스텀 구현을 설명하는 ID별 Manifest

판정 규칙:

- 계획 범위와 실제 변경이 같으면 자동 진행
- 실제 변경이 계획보다 넓거나 좁으면 Manifest를 자동 확장해 통과시키지 말고 `scope_variance`로 기록 후 STOP
- 사람이 variance를 승인하거나 om-plan을 개정한 뒤에만 Manifest 갱신
- 최종 단계에서 `전체 최종 net diff == ID별 최종 귀속 범위의 합집합`을 양방향 비교
- `required_changed_paths`는 touched 여부가 아니라 최종 tree에서 요구한 내용·심볼이 남는지도 별도 검사

## 2. §3 파일 해석 규칙

### P0-2. `path-remap`은 출발지만 쓰면 통과한다

**허점**

현재 validator는 영향받은 등록 경로가 remap의 출발지에 등장하는지만 본다. 목적지 키·값·존재·안전성·유일성은 검사하지 않는다.

**코드 근거**

- `collectors.py:801-821`은 `path/from/source_path/old_path` 중 하나만 읽어 coverage 집합에 넣는다.
- `to`, `target_path`, 삭제 사유, 대상 존재 여부는 읽지 않는다.
- discovered fact 스키마의 `value`에는 구조 제약이 전혀 없다: `plan-discovered-facts.schema.json:17-27`.
- 정상 테스트는 `{from, to}`를 쓰지만(`test_plan_workflow.py:480-483`), 반례 테스트는 출발지 누락만 검사한다(`:2185-2194`). 목적지 누락·경로이탈·중복 목적지 반례가 없다.

따라서 아래 제안도 출발지 coverage만 충족하면 현재 구조상 목적지 오류를 잡지 못한다.

```yaml
path-remap:
- from: old/Custom.java
  # to 없음
```

또는 `to: ../../outside`, 존재하지 않는 경로, 전혀 다른 기능의 파일도 별도 검사 대상이 아니다.

**수정 제안**

mode별 proposal 스키마에 다음을 강제한다.

- `action: keep|move|replace|delete|generate`
- `from`과 `to`의 필수 조건을 action별로 구분
- literal repository-relative path만 허용하고 `..`, 절대경로, symlink 차단
- `from`은 official base 또는 custom baseline에 실제 존재
- `to`는 official target 또는 구현 후보에 실제 존재. `delete`만 목적지 생략 허용
- 영향 경로마다 정확히 하나의 disposition이 존재
- 여러 source가 같은 target을 가리키거나 한 source가 여러 target으로 갈 때는 명시적 관계와 검토 사유 필요
- 최종 candidate에서 target 파일·심볼·공유 코드 정의가 실제로 검출되는지 대조

### P1-1. `changed_paths`의 뜻이 기존 관리 모델과 충돌한다

초안은 `changed_paths`를 “생성/수정할 파일의 초기 계획 목록”이라고 정의한다(`58`: 46행). 예전 Manifest 구현은 이 값을 “현재 버전의 구현 범위”라고 정의한다(`manifest.py:11-16`, `:78-86`). 둘은 다른 개념이다.

계획 목록과 현재 구현 범위를 같은 필드로 쓰면 다음이 구분되지 않는다.

- 계획했지만 실제로 수정하지 않은 파일
- 계획 밖에서 추가로 수정한 파일
- 과거에는 필요했지만 새 버전에서 삭제·대체된 파일
- 생성기의 입력 파일과 생성 결과 파일

**수정 제안:** `planned_actions`와 최종 `implementation.changed_paths`를 분리한다. 전자는 run 산출물, 후자는 최종 등록 상태다. 계획 변경은 `scope_variance` 승인 이력으로 연결한다.

### P1-2. `required_changed_paths`의 뜻도 모드별로 갈린다

초안은 “이번 실행에서 반드시 변경돼야 하는 파일”로 읽힌다(`58`: 47행). 기존 Manifest validator는 required가 현재 구현 범위의 부분집합인지 검사한다(`manifest.py:11-17`, `:107-125`). 예전 apply는 이번 커밋 범위에서 한 번이라도 touched 되었는지를 본다(`registration_prep.py:598-613`).

세 의미가 다르다.

- 최종 기능에 반드시 존재해야 함
- 이번 변경에서 반드시 건드려야 함
- 특정 코드·값이 최종 결과에 반드시 남아야 함

**수정 제안:** 최소한 `required_final_paths/assertions`와 `required_transaction_actions`를 분리한다. 단순 파일명만으로 기능 생존을 판정하지 않는다.

### P1-3. 삭제·이동은 예전 apply 재사용 시 정상 처리할 수 없다

예전 `_check_path_modes`는 커밋에서 건드린 모든 경로가 최종 HEAD에 존재해야 한다. 없으면 `CHANGED_PATH_ABSENT_AT_HEAD`로 차단한다: `registration_prep.py:300-320`.

따라서 upgrade에서 합법적으로 파일을 삭제하거나 새 위치로 옮겨도 source 경로가 최종 HEAD에서 사라지면 거짓 block이 난다. `path-remap`과 호환되지 않는다.

**수정 제안:** 변경 단위를 단순 path 목록이 아닌 action 모델로 바꾸고, delete/move/replace는 이전 위치 부재가 정상인 경우를 명시한다.

### P1-4. 공유 파일 처리는 기존 목록 밖 신규 공유를 놓친다

현재 `/om-plan`은 관측 또는 공식 변경 경로와 **이미 등록된** `shared-path-owners`의 교집합만 확인한다: `collectors.py:647-667`. 새로 두 ID가 함께 수정하게 된 경로가 shared map에 없으면 이 검사로 발견되지 않는다.

예전 apply도 공유 목록의 기대값을 과거 `source-snapshot-path-owners.yaml`에서 파생한다: `registration_prep.py:710-748`. 새 버전에서 새로 공유된 경로를 자연스럽게 추가하는 모델이 아니다. 더구나 apply 쓰기 화이트리스트에는 `shared-path-owners.yaml`, `contracts.yaml`, `shared-code-definitions.yaml`이 없다: `registration_prep.py:964-978`, `:1054-1069`.

**수정 제안:** 최종 per-ID 관계에서 동일 경로가 여러 ID에 속하는 경우를 계산하고, 기존 shared map과는 포함·차이를 보고한다. 신규 공유는 사람 검토 대상으로 만들며 shared owner와 코드 정의 변경도 A형 proposal의 1급 산출물로 다룬다.

### P1-5. 자동 병합의 의미가 문서 안에서 충돌한다

초안 §3은 충돌 항목에 사람 판단이 필요할 수 있다고 한다(`58`: 51행). §8은 자동 병합을 확정했다고 한다(`58`: 94~96행). Git이 충돌 없이 합쳐졌다는 사실과 업무 의미가 보존됐다는 사실은 다르다.

**수정 제안:** 확정된 “자동 병합”을 다음 두 단계로 정밀화한다.

1. **자동 병합 시도:** Git 기계 병합
2. **병합 결과 수용:** 계획 범위·공유 영향·정적 assertion·후속 테스트 목록 대조

기계 충돌 0은 2단계 통과를 뜻하지 않는다. 수용 실패 시 자동 병합 결과를 후보로 보존하되 다음 단위로 진행하지 않는다.

## 3. §4 최소 개발 단위

### P1-6. `스키마 → 서비스 → UI`만으로 실제 OpenMetadata 의존성을 표현할 수 없다

실제 BANK-OM-001은 한 ID 안에 다음이 함께 있다.

- MySQL·PostgreSQL migration
- Java DAO·repository·resource·search index
- JSON schema·search mapping
- 생성된 TypeScript
- router·UI·REST API
- 19개 locale 파일

근거: `manifests/BANK-OM-001.yaml:7-55`.

BANK-OM-006은 JSON schema 하나가 여러 생성 TypeScript 파일과 UI utility로 전파된다: `manifests/BANK-OM-006.yaml:7-25`. 또한 006·007은 service schema와 generated 파일을 공유한다: `shared-path-owners.yaml:19-21`, `:42-47`.

따라서 폴더 이름 기반 레이어 그룹만 쓰면 다음 의존을 잘못 정렬할 수 있다.

- schema source → code generation → generated outputs
- DB migration의 DB별 동등성
- backend registration → API → UI route
- locale key source → 다국어 파일 전파
- shared file owner 간 조정
- 운영 작업(reindex/migration/configuration)

**수정 제안**

- ID는 **귀속·검토 묶음**으로 유지한다.
- 실행 단위는 `role + action + dependency`로 만든 DAG로 분리한다.
- 최소 role 예: `migration`, `schema-source`, `generation`, `generated-output`, `backend`, `api`, `ui`, `i18n`, `shared-integration`, `operation`.
- 파일 경로 추정만으로 role을 정하지 말고 repository layout·generator 설정·공식 업그레이드 문서·기존 관계 fact를 근거로 남긴다.
- 공유 파일 단위는 한 ID의 독점 단위가 아니라 관련 ID를 함께 표시한다.

## 4. §6 게이트

### P0-3. 형식 일치와 기능 구현을 혼동할 위험이 있다

초안은 테스트 실행이 om-verify 몫임을 적었지만(`58`: 26~27, 81~83행), “계약 불변식을 구현이 충족하는지 LLM 판단”을 전체 개발 후 게이트로 둔다. 작성자 LLM이 자기 코드를 읽고 “충족”이라고 쓰는 것은 독립 검증이 아니다.

현재 A′도 등록된 계약 테스트가 제안서 목록에 모두 있는지만 확인한다: `collectors.py:601-632`. 테스트를 실제 실행하거나 등록 자체가 충분한지는 검사하지 않는다. 계약의 `invariant`는 문장이고 실제 required test selector와 함께 저장된다(`contracts.yaml:3-53`).

**수정 제안**

- om-apply 결과의 상태를 `implemented` 또는 `ready`가 아니라 **`static_consistent_awaiting_verify`**처럼 명시한다.
- om-apply가 할 수 있는 판정은 다음으로 제한한다.
  - 계획·실제 diff·관리파일의 3방향 정합
  - 경로와 ID 귀속
  - 스키마와 증거 참조
  - 필수 테스트 목록 누락 여부
- “계약 충족”은 om-apply에서 pass로 만들지 않는다. LLM 평가는 `claim` 또는 `review_note`로만 저장한다.
- om-verify의 실제 결과와 동일 candidate SHA가 결속되기 전에는 최종 승인·배포 상태로 승격하지 않는다.

### P1-7. 현재 proposal·fact 구조는 비어 있지 않으면 통과하는 구간이 있다

**코드 근거**

- proposal loader는 모든 YAML/JSON mapping을 읽지만 mode별 문서 스키마를 강제하지 않는다: `plancore/validate.py:70-90`.
- discovered fact의 `value`는 구조 제한이 없다: `plan-discovered-facts.schema.json:17-27`.
- upgrade 산출물의 `meaningful()`은 문자열·목록·dict가 비어 있지 않은지만 본다: `collectors.py:769-799`.
- path-remap은 출발지 coverage만 본다: `collectors.py:801-821`.
- `id-contract-consistency`는 fact로 수집되지만(`collectors.py:503-505`), 현재 소비 지점 검색 결과 change의 `customization-relations` 외에는 이 consistency fact를 차단 게이트로 쓰지 않는다.

따라서 `upgrade-plan: [{note: done}]`, 출발지만 있는 remap, 내용 없는 manifest delta가 형식상 존재할 수 있다.

**수정 제안**

- om-apply 입력용 mode별 typed schema를 만든다.
- 각 단위에 `unit_id`, ID, action, source/target, dependency, expected result, evidence refs, required tests를 강제한다.
- evidence ref는 현재처럼 실제 fact pointer와 expected 값을 역참조한다.
- `id-contract-consistency.consistent != true`면 om-apply 시작 전 fail-closed 한다.
- 단순 digest 일치는 내용 변조 탐지이지 내용의 타당성 증명이 아니라는 문구를 설계에 명시한다.

### P1-8. 검사기 저장소와 제품 저장소의 최종 결속이 빠져 있다

예전 apply는 제품의 patch/custom SHA와 등록자료 digest를 재확인한다: `registration_prep.py:1041-1052`. 현재 om-plan도 input-lock과 fact를 재수집한다: `plancore/validate.py:493-522`.

그러나 A형에서는 최소한 다음이 함께 묶여야 한다.

- om-plan `plan_digest`
- 검사기/관리파일 저장소의 기준 commit 또는 등록자료 digest
- 제품 저장소의 단위 시작 SHA·단위 종료 SHA·최종 candidate SHA/tree
- 생성·수정된 관리파일 digest
- 후속 om-verify가 사용할 candidate SHA와 테스트 목록

하나라도 달라지면 이전 단위 결과나 verify 결과를 재사용하지 않도록 해야 한다.

## 5. A형 실행 경계의 정직한 정의

om-apply가 정적으로 확인 가능한 것과 불가능한 것을 다음처럼 구분해야 한다.

| om-apply에서 가능 | om-apply에서 불가능 — om-verify/사람으로 넘김 |
|---|---|
| 승인된 계획과 실제 Git 변경 비교 | 기능이 실제 서버에서 동작하는지 |
| 경로·ID·공유 owner 귀속 | UI가 원하는 방식으로 보이는지 |
| 관리파일 구조·상호 참조 | Contract의 업무 의미가 충분한지 |
| 필수 테스트가 실행 목록에 들어갔는지 | 테스트가 실제 통과하는지 |
| 코드 후보·관리자료 digest 결속 | clean merge가 의미 보존까지 했는지 |

이 경계를 지키면 om-apply는 “코드를 다 검증했다”가 아니라 **“승인된 계획에 따라 구현했고, 실행 검증에 넘길 후보와 관리자료가 서로 일치한다”**까지만 증명한다.

## 6. 예전 apply 재활용 타당성

### 6.1 (a) B형과 A형의 흐름 불일치

예전 모듈은 스스로 B형임을 명시한다. plan은 기존 Git 사실을 계산하고 apply는 승인된 생성 파일만 쓴다: `registration_prep.py:1-12`.

실제 흐름은 다음과 같다.

1. 이미 완성된 `patch_ref..custom_ref`의 커밋을 읽음: `registration_prep.py:378-425`
2. 각 커밋의 트레일러·경로로 ID 귀속: `:421-483`
3. 최종 net diff의 미귀속 경로를 차단: `:502-519`
4. proposal 승인 후 등록자료만 원자적으로 기록: `:1000-1137`

반면 A형은 계획을 읽고 아직 없는 제품 코드를 작성하며 단위마다 상태가 변한다. 예전 `build_plan`은 dirty worktree를 차단하고(`:385-391`), merge commit도 차단한다(`:425-429`). 따라서 A형의 단위 작성 엔진이나 upgrade 자동 병합 엔진으로 그대로 호출할 수 없다.

**그대로 쓸 수 있는 후보**

- Git ref를 SHA로 고정하고 다시 확인하는 패턴
- 커밋 트레일러 cardinality 검사
- per-commit changed paths와 최종 net diff 계산
- 경로 literal/symlink/submodule/LFS 방어 개념
- proposal·등록자료 digest 대조
- `_safe_target`, apply lock, atomic write, rollback 패턴

**흐름상 새로 필요한 것**

- om-plan의 단위 DAG를 실행하는 A형 orchestration
- 단위 시작/종료 SHA와 scope variance
- add/modify/delete/move/generate 행위 모델
- 자동 병합 시도와 수용 판정 분리
- 단위별 중단·재개 및 최종 candidate 생성
- om-verify 인계 binding

### 6.2 (b) 최신 트랙 A 정책·스키마와의 불일치

#### 불일치 1 — 테스트 관계

현재 트랙 A는 ID 관계의 `tests`를 **그 ID가 참조한 Contract의 `required_tests`만**으로 계산한다: 현재 `collectors.py:156-170`. 테스트도 `direct_tests`가 포함되지 않음을 확인한다: `test_om_registration_facts.py:129-180`.

예전 apply는 다르다.

- 신규 Manifest에 `assurance.direct_tests`를 씀: `registration_prep.py:188-219`
- 신규 ID 입력에서 `direct_tests`를 받음: `:236-279`
- 예전 Contract 해석은 `direct ∪ contract-derived`를 effective tests로 사용: `contracts.py:98-121`
- 예전 Manifest 스키마도 두 목록의 합집합 정책을 명시: `manifest.schema.json:54-60`

따라서 예전 apply를 그대로 재사용하면 폐기한 direct-test 우회 통로를 다시 활성화하거나, 현재 per-ID 관계 fact와 다른 테스트 집합을 생성한다.

#### 불일치 2 — per-ID 관계와 consistency

현재 수집기는 다음을 fact로 보존한다.

- `customization-relations`
- `contract-tests`
- `id-contract-consistency`

근거: 현재 `collectors.py:465-505`.

예전 `registration-proposal.schema.json`은 patch/custom refs, commit inventory, current diff, Manifest changes를 저장하지만 위 관계 fact나 om-plan의 3-digest `plan_binding`을 모른다: `registration-proposal.schema.json:7-82`.

예전 apply는 registry↔Manifest↔Contract 참조를 검사하지만(`registration_prep.py:135-158`), 현재 om-plan이 고정한 discovered facts와 동일한 per-ID 관계를 읽고 plan digest에 결속하지 않는다.

#### 불일치 3 — 쓰기 범위

예전 apply가 쓸 수 있는 것은 다음뿐이다.

- `commit-inventory.yaml`
- `current-diff-paths.txt`
- `customization-registry.yaml`
- `manifests/BANK-OM-*.yaml`

근거: `registration_prep.py:964-978`, `:1054-1069`.

따라서 Contract, shared owner, shared code definition까지 코드와 함께 갱신해야 하는 A형 §0.5를 충족할 수 없다.

#### 불일치 4 — 배선·패키징

예전 `om_workflow.py`의 apply는 `prepare_registration.py apply`를 호출하는 얇은 B형 래퍼다: `om_workflow.py:417-443`. 현재 clean-export의 `harness/acgh`에는 `registration_prep.py`, `manifest.py`, `contracts.py`, `shared_code.py`가 없다. 파일 하나를 복사해 재활용할 수 있는 구조가 아니다.

### 6.3 선택지와 트레이드오프

| 선택지 | 가능한 구성 | 장점 | 치명적 비용·위험 | 실측 판단 |
|---|---|---|---|---|
| **재활용** | 예전 `build_plan/apply_plan`을 A형 om-apply 본체로 사용 | 이미 있는 등록·atomic write 코드 활용 | B형 사후 등록 흐름, dirty/merge/delete 제약, direct_tests 구정책, plan binding·per-ID fact 부재, Contract/shared 파일 쓰기 불가 | **현재 상태 그대로는 부적합** |
| **부분 재활용** | A형 orchestration·스키마·게이트는 신규. Git/digest/safe-write/rollback 같은 좁은 부품만 import | 검증된 안전 부품을 살리면서 최신 트랙 A 정책 유지 | 부품 경계와 새 반례 테스트가 필요 | **기술적으로 가장 타당한 선택지** |
| **신규** | A형 전체를 새 모듈로 작성 | 흐름과 스키마가 단순하고 명확 | Git 파싱·원자적 쓰기·stale 방어까지 재구현하면 과거 결함 재발 가능 | 가능하나 공통 안전 부품까지 새로 만들 이유는 약함 |

**검토 결론:** “예전 apply 전체 재활용”은 (a) 흐름과 (b) 최신 정책 양쪽에서 맞지 않는다. **A형 수명주기와 데이터 모델은 신규**, 검증된 저수준 안전 primitive는 **부분 재활용**하는 안이 가장 일관된다. 패키징 위치 8-2는 이 결론만으로 확정하지 않는다.

## 7. 58 설계서에 필요한 수정 목록

1. §0.5를 “관리파일=코드”에서 **계획↔실제 diff↔최종 관리자료 3방향 대조**로 개정
2. 계획 밖 변경은 Manifest 자동 추가가 아니라 `scope_variance + STOP + 승인/재계획`으로 변경
3. §2 입력에 plan digest, 제품 baseline, 검사기/등록자료 기준, action별 typed unit 추가
4. §3의 `changed_paths`와 계획 경로를 분리하고 required 의미를 세분화
5. path-remap에 목적지·삭제·생성·안전 경로·실재 대조 규칙 추가
6. §4를 폴더 레이어가 아닌 role/action/dependency DAG로 변경
7. §6에서 LLM의 계약 충족 판단을 pass 게이트에서 제거하고 claim으로 격하
8. 최종 상태를 `static_consistent_awaiting_verify`로 정의하고 verify 전 승인·배포 차단
9. §7 재활용 지도를 “A형 신규 orchestration + 안전 primitive 부분 재활용”으로 수정
10. direct_tests 계약-only 정책, per-ID 관계와 consistency, shared 관리자료 갱신을 명시

## 8. 구현 전 반례로 추가할 항목

| ID | 반례 | 기대 판정 |
|---|---|---|
| OA-01 | 계획 밖 보안 파일을 수정하고 Manifest에도 같이 추가 | scope variance STOP |
| OA-02 | required 파일을 수정했다가 뒤 커밋에서 원상복구 | block |
| OA-03 | final net diff에 있지만 어느 ID에도 귀속되지 않은 경로 | block |
| OA-04 | Manifest에 있으나 final candidate에는 효과가 없는 경로 | block 또는 명시적 unchanged disposition |
| OA-05 | path-remap에 `from`만 있고 `to` 없음 | block |
| OA-06 | path-remap target이 `../`·절대경로·symlink | analysis_error/block |
| OA-07 | path-remap target이 official target/final candidate에 없음 | block |
| OA-08 | 합법적인 move/delete인데 source가 final HEAD에 없음 | 정상 처리 |
| OA-09 | 새 파일을 두 ID가 공유하지만 기존 shared map에는 없음 | 신규 공유 검토 STOP |
| OA-10 | shared owner 한 ID 누락 | block |
| OA-11 | 생성 schema는 바뀌었으나 generated outputs가 stale | block |
| OA-12 | MySQL migration만 바뀌고 PostgreSQL 대응 누락 | dependency/completion block |
| OA-13 | `upgrade-plan: [{note: done}]`처럼 비어있지 않기만 한 문서 | schema block |
| OA-14 | `id-contract-consistency=false`인데 실행 시도 | preflight block |
| OA-15 | 코드·Manifest 정합은 맞지만 required test 미실행 | apply는 awaiting_verify, verify는 block/incomplete |
| OA-16 | apply 후 candidate SHA가 바뀜 | 이전 apply/verify 결과 재사용 금지 |
| OA-17 | 예전 direct_tests가 Track A tests에 다시 합쳐짐 | 회귀 실패 |
| OA-18 | 예전 B형 apply가 Contract/shared definition 수정을 쓰려 함 | forbidden 또는 A형 writer로 명시 처리 |
| OA-19 | Git clean merge지만 커스텀 assertion이 사라짐 | merge 수용 block |
| OA-20 | 여러 ID 공유 파일을 한 ID 단위가 독점 변경 | shared coordination block |

## 9. 사람에게 올릴 미확정 질문

다음은 이번 검토에서 임의로 확정하지 않았다.

1. **8-2 패키징 위치:** 완전본 확장 / clean-export 편입 / 공통 core와 OM integration 분리
2. **계획 범위 이탈 승인 단위:** 파일별 / 개발 단위별 / ID별
3. **신규 공유 경로 승인 주체:** 관련 ID 담당자 전원 / 한 명의 통합 담당자
4. **자동 병합 수용 실패 처리:** 후보 보존 후 사람 수정 / 해당 단위 롤백 후 재계획
5. **om-apply 종료 전 최소 실행 검사:** 완전 정적 유지 / 컴파일·생성기 같은 빠른 검사는 apply에 포함

기술 권고는 문서에 제시할 수 있지만, 위 항목은 사람 결정 뒤 구현 지시서에 고정해야 한다.

## 10. 최종 판정

- **설계 상태:** 수정 필요
- **구현 착수:** P0-1~P0-3과 재활용 경계를 58에 반영한 뒤 가능
- **예전 apply:** as-is 재활용 부적합, 안전 primitive의 부분 재활용 가능
- **현재 코드 변경:** 없음
- **커밋·push·MR:** 없음


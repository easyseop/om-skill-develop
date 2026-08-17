# Codex 전달 — GitLab CI 제품 clone의 `--filter=blob:none` 수정 (P: fail-closed로 파이프라인이 아예 안 돎)

> 작성: 2026-08-17. 이 문서를 코덱스에게 그대로 전달한다.
> 근거: `~/Desktop/om-plan_GitLab배선_적대검토_결과_20260815.md`,
> `~/Desktop/om-plan_GitLab이전전_확인_코덱스_지시문_20260817.md`의 확인 결과(코덱스 §7).

## 1. 문제 (한 줄)

`.gitlab-ci.yml`이 제품(OpenMetadata) 저장소를 **`--filter=blob:none`(파일 내용 제외)**로
clone하는데, `/om-plan` 검사기는 필요한 source blob을 **`GIT_NO_LAZY_FETCH=1`으로 확인하고
없으면 `SOURCE_BLOBS_UNAVAILABLE`로 fail-closed**한다. 두 정책이 충돌해, 실제 GitLab
파이프라인에서 preflight가 fact 수집 단계에서 **항상 `analysis_error`로 멈춘다**(위험한
false pass가 아니라, 아예 동작하지 않음).

## 2. 근거 파일·행 (기준 commit `ee9f033`)

- 제품 clone (수정 대상, 2군데):
  - `.gitlab-ci.yml` **128행** (`om_plan_preflight` job): `git clone --filter=blob:none "$product_url" "$OM_PLAN_RUNTIME_ROOT/product"`
  - `.gitlab-ci.yml` **257행** (`om_plan_validate` job): 동일
- 충돌하는 검사기 로직:
  - `harness/acgh/integrations/om/collectors.py` **117행** `environment["GIT_NO_LAZY_FETCH"] = "1"`
  - 같은 파일 **149행** `SOURCE_BLOBS_UNAVAILABLE`

주의: 같은 파일 118·203행의 `git clone --filter=blob:none --no-checkout ...`은 **request/proposal
source**를 받는 것으로, 여기서는 blob이 필요 없다(내용 미사용). **이 두 줄은 건드리지 마라.**
수정 대상은 **제품(product) clone 2군데(128·257)뿐**이다.

## 3. 수정 목표

preflight·validate job의 제품 clone 이후, 검사기가 읽을 **pinned ref의 source blob이 job
로컬에 물리적으로 존재**하게 만든다. 방법은 코덱스 판단(아래 중 택1 또는 동등안).

- **(a) 가장 단순** — 제품 clone 2군데에서 `--filter=blob:none`을 제거해 full clone.
  분명하고 확실하나, 대형 저장소면 job마다 무겁다.
- **(b) 필요한 blob만 materialize** — blobless clone은 유지하되, preflight 전에 검사가 읽을
  **pinned ref(공식·커스텀 SHA)에 도달 가능한 blob을 명시적으로 fetch**한다(예:
  `git -C product fetch --refetch` 또는 대상 ref 범위의 blob 채우기). CI job이 `GIT_NO_LAZY_FETCH=1`
  하에서도 없는 게 없도록.
- **(c) partial 유지 + lazy-fetch 허용은 금지** — 검사기의 `GIT_NO_LAZY_FETCH=1` fail-closed는
  "무엇을 검사했는지" 무결성의 일부다. 이를 끄는 방향(lazy-fetch 허용)으로 우회하지 마라.

(a)를 기본으로 하되, 저장소 크기·job 시간이 문제면 (b)로 최소화하라.

## 4. 반드시 성립해야 할 것 (수용 기준)

1. **`GIT_NO_LAZY_FETCH=1` 상태에서** preflight의 fact 수집(특히 `_materialized_objects`)이
   `SOURCE_BLOBS_UNAVAILABLE` 없이 통과한다 — 즉 pinned ref의 필요한 blob이 job 로컬에 있다.
2. preflight→validate 전 과정이 실제 제품 clone 방식으로 정상 진행돼 정상 계획에서
   `review_ready`/exit 2에 도달한다(로컬 재현).
3. **보안·무결성 불변:** 검사기의 `GIT_NO_LAZY_FETCH=1`·`SOURCE_BLOBS_UNAVAILABLE` fail-closed를
   약화하지 않는다. `.pyc` cleanup·지문·격리·fresh 검증 등 기존 경계 무변경.
4. request/proposal source의 clone(118·203행)은 변경하지 않는다.
5. 회귀 test 추가: blobless 제품 상황에서 fact 수집이 실패하고, 수정 방식 적용 후 통과함을
   증명(또는 CI가 blob을 확보함을 검증하는 test).

## 5. 참고 — 제품 콘텐츠는 로컬에 준비됨

사용자가 GitHub이 살아 있을 때 제품 1.13.1을 완전 materialize했다
(`~/om-work/OpenMetadata-1.13.1-full`, full clone, 전체 이력 누락 객체 0,
공식 `afcb2d2c`·커스텀 `8ac18ad0` 모두 blob 포함). 즉 **GitLab에 올릴 원본에는 blob이 다 있다.**
남은 문제는 **CI job이 그 저장소에서 blob까지 가져오게** 하는 것뿐이다(이 수정).

## 6. 손대지 말 것 / 보류

- `verdict.py` enum·exit code, 제품 코드, 활성 등록자료, 이미 채택된 E-hardening·`.pyc`·격리 로직.
- request/proposal source clone(118·203행).
- Q3·Q6·Q7·Q8·Q9는 임의 확정 금지. exit 2의 pipeline 성공 매핑(Q9)도 이 수정 범위 아님.

## 7. 종료 조건 / 다음

1. 수정한 `.gitlab-ci.yml`(및 필요 시 helper)과 §4를 만족하는 test·로컬 실행 결과를 보고한다.
2. 완료 보고가 오면 Claude가 읽기 전용으로 재검토한다. 재검토 핵심:
   `GIT_NO_LAZY_FETCH=1`에서 fact 수집이 통과하는가, fail-closed 무결성이 유지되는가,
   request/proposal source clone은 그대로인가.
3. 이 수정 뒤에 GitLab 이전(저장소 2개 push)과 외부 설정으로 넘어간다. 실제 GitLab pipeline은
   사용자 승인 후 실행한다.

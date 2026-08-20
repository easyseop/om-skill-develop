# 요청 파일 형식 (plan-run-request)

`plan start`에 넘기는 요청 YAML의 필드다. 스키마 정본은 검사기의 `harness/acgh/plancore/schema/plan-run-request.schema.json`이다. 이 문서와 스키마가 다르면 스키마를 따른다.

## 모든 모드 공통 필수 필드

- `schema_version`: 정수. 현재 `1`.
- `run_id`: 문자열. 이 실행을 식별하는 값.
- `mode`: `initial` · `feature` · `change` · `upgrade` 중 하나.
- `repositories`:
  - `product`: 제품(OpenMetadata) 저장소의 절대 경로.
  - `checker`: 검사기 저장소의 절대 경로.
- `refs`:
  - `official`: 공식 버전 커밋 SHA.
  - `current_custom`: 현재 커스텀 커밋 SHA.
  - (값은 모두 문자열이다.)

## 선택·모드별 필드

- `repository_ids`: `product`·`checker` 각각의 저장소 식별자(예: `easyseop/OpenMetadata`). 증거에 기록용.
- `product_version`: 문자열(예: `"1.13.1"`).
- `owner`: 담당자. 없으면 `null`. 비어 있으면 proposal은 담당자 질문 + `next_step_blocked: true`를 넣어야 한다.
- `customization_id`: `feature`·`change` 모드에서 대상 ID를 지정할 때.
- `upgrade` 모드 추가 필수: `deployment_method`(배포 방식)와 인접 버전 정보. `validate_request`가 upgrade에서 이 항목을 요구한다. 비인접 버전은 1차 범위에서 `analysis_error`로 거부된다(Q2=A).

## 예시 (initial)

```yaml
schema_version: 1
run_id: initial-1.13.1-example-20260819
mode: initial
repositories:
  product: /절대/경로/OpenMetadata-1.13.1-full
  checker: /절대/경로/kb-datacatalog-upgrade-checker
repository_ids:
  product: easyseop/OpenMetadata
  checker: datacatalog/kb-datacatalog-upgrade-checker
refs:
  official: afcb2d2cd7e7c28f1d0ce60538c60a96f4eb9dc9
  current_custom: 8ac18ad053d9274774e274ba17b35911ac0b9dcb
product_version: "1.13.1"
owner: null
```

## 검증

- 필수 필드가 하나라도 없으면 `plan start`가 입력 오류(종료코드 2)로 멈춘다.
- 무엇이 빠졌는지 사용자에게 보고하고 요청 파일을 고친 뒤 다시 실행한다.

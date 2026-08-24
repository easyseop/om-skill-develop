# Codex 결과 — om-apply P0 수정 및 재리허설

작성일: 2026-08-24  
정본 지시: `67_Codex_omapply_P0수정_지시_20260824.md`

## 1. 결론

요청한 P0-1·P0-3·P1-1과 격리 위생 조치를 반영했다. 새 run으로 실행한 실제 BANK-OM-005 change는 정상적으로 verify 인계 직전까지 도달했고, 계획 밖 파일과 Manifest 확장 probe는 계속 차단됐다.

| 항목 | 결과 |
|---|---|
| 정상 change | `pass / static_consistent_awaiting_verify` |
| verify 인계 | `eligible: true`, BANK-OM-005 계약 테스트 1개 |
| scope probe | `block / blocked`, `eligible: false` |
| 공유 소유맵 | Manifest 파생 39개 = 소유맵 39개, 차이 0 |
| 전체 회귀 | 239개 수집, 전체 실행 exit 0 |
| 원본 검사기 commit·push | 0건 |

`pass`는 정적 정합 검사가 끝났다는 뜻이다. 실제 Contract test는 실행하지 않았으며 결과에도 `claim_only_not_executed`로 남아 있다.

## 2. 반영 내용

### 2.1 P0-1 — 경로 역할 분리

파일: `harness/acgh/integrations/om/apply.py:118-161`

- `all_owned_paths`: 최종 제품 diff에 등록 밖 경로가 있는지 검사할 때만 사용한다.
- `planned_owned_paths`: 이번 계획의 unit에 포함된 ID 경로가 최종 후보에 실제 효과를 남겼는지 검사한다.
- 다른 ID의 Manifest 경로가 이번 단일 ID diff에 없다는 이유로 차단하지 않는다.
- 여러 ID가 계획 대상이면 그 ID들의 경로는 모두 검사한다.

### 2.2 P0-3 — 활성 공유 소유맵 2경로 정정

파일: `harness/registrations/om-temp-1.13.1/shared-path-owners.yaml:45-47,146-148`

두 경로 모두 BANK-OM-006·007 Manifest의 `implementation.changed_paths`에 실제로 존재함을 확인한 뒤 소유자를 기입했다.

- `openmetadata-ui/src/main/resources/ui/src/generated/entity/services/connections/serviceConnection.ts`
- `openmetadata-ui/src/main/resources/ui/src/utils/DatabaseServiceUtils.test.tsx`
- 소유자: `BANK-OM-006`, `BANK-OM-007`

활성 등록자료 변경은 이 파일의 두 항목 추가뿐이다.

| 구분 | 이전 | 이후 |
|---|---|---|
| 등록자료 전체 digest | `sha256:9cc42f0f4d4368df46d509f24c43942d6c9cf20f47013eac76818752d8a23c3d` | `sha256:2cb43987313f6394dfba51e11b769e6e8254985a7e214d159bbc5705630b5be5` |
| shared-path-owners 파일 digest | `sha256:c87ddc405651d4b348cceaadb127f28ba25ddcbec55775eba2ae20a6c0f672f6` | `sha256:8794bf00f829d1cc0d3d708bbd360d23bd3ecd0a827ada9b09e253fcd3b9b97e` |
| Manifest 파생 공유 경로 | 39 | 39 |
| 소유맵 경로 | 37 | 39 |
| 경로·소유자 차이 | 누락 2 | 0 |

### 2.3 P1-1 — fast_checks 하한

파일:

- `harness/acgh/applycore/schema/apply-execution-plan.schema.json:70-99`
- `harness/acgh/applycore/workflow.py:201-208,1040-1055`

허용 형식은 다음 둘뿐이다.

1. 항목이 한 개 이상인 `fast_checks` 배열
2. `not_applicable: true`와 비어 있지 않은 `reason` 객체

빈 배열은 apply 실행계획 스키마 검증에서 fail-closed로 거부된다. 구조화된 N/A는 결과 gate에 `execution_status: not_applicable`과 사유를 보존한다.

## 3. 반례와 회귀 결과

추가·고정 위치: `harness/tests/test_om_apply_counterexamples.py:363-451`

| 반례 | 기대 | 결과 |
|---|---|---|
| 정상 단일 ID, 다른 ID 경로는 diff에 없음 | pass·awaiting_verify | 통과 |
| 등록 밖 최종 diff 경로(OA-03) | block 유지 | 통과 |
| 계획 ID의 Manifest 경로에 최종 효과 없음(OA-04) | block 유지 | 통과 |
| 여러 ID 계획에서 두 번째 ID 경로 누락 | block | 통과 |
| `fast_checks: []` | 실행계획 승인 불가 | 통과 |
| 구조화된 N/A+사유 | 허용 | 통과 |

명시 실행 결과: 6개 모두 `PASSED`. 기존 touch→원복 OA-02를 포함한 `harness/tests` 전체는 239개가 수집됐고 전체 실행 exit code는 0이었다.

테스트 green만으로 완료를 주장하지 않는다. 아래 §5의 실제 OpenMetadata 자료 재리허설 결과를 함께 근거로 사용한다.

## 4. 격리 위생

오염된 과거 복사본:

- 폐기: `work/om-apply-rehearsal-20260824-01/checker-template`
- 폐기 전 `.git`은 원본 worktree의 Git dir를 가리켰다.
- 폐기 후 경로가 존재하지 않음을 확인했다.

새 격리 루트:

`work/om-apply-rehearsal-20260824-02`

1. 원본을 복사할 때 `.git` 파일·디렉터리를 모두 제외했다.
2. 복사 직후 `.git` 부재를 확인했다.
3. 상위 workspace Git도 실행 정본으로 쓰지 않도록 복사본 내부에 독립 `.git`을 생성했다.
4. 실제 사용 전 Git dir가 각 복사본 내부를 가리키는지 확인했다.

| 복사본 | 실제 Git dir |
|---|---|
| checker-snapshot | `.../checker-snapshot/.git` |
| checker-normal | `.../checker-normal/.git` |
| checker-probe | `.../checker-probe/.git` |

원본 checker의 Git dir나 원본 worktree 경로를 가리키는 복사본은 사용하지 않았다.

원본 checker 브랜치 HEAD는 작업 전후 모두 `7c544efb2bc12200cf4b9e7dfef82d5358f29812`이며 새 commit은 없다. 실행 결속을 위해 폐기 가능한 독립 복사본에만 root snapshot commit `b691f781d68f3d82211b5368e4ce0b86a13387ee`를 만들었다. 이는 원본 branch·remote에 연결되지 않는다.

원본 미커밋 코드와 격리 snapshot은 `.git`·pytest cache 제외 `diff -qr` 결과가 빈 출력이다.

## 5. 새 실데이터 재리허설

기준 후보:

- start commit: `8ac18ad053d9274774e274ba17b35911ac0b9dcb`
- start tree: `e86980f6d71295465fe6e75e5169fab57cebd4c4`
- 등록자료 digest: `sha256:2cb43987313f6394dfba51e11b769e6e8254985a7e214d159bbc5705630b5be5`

### 5.1 정상 BANK-OM-005 change

산출물:

- plan: `work/om-apply-rehearsal-20260824-02/normal-plan-run`
- apply: `work/om-apply-rehearsal-20260824-02/normal-apply-run`
- 제품 branch: `codex/rehearsal-om-apply-p0-normal-20260824`

| 값 | 결과 |
|---|---|
| plan verdict/state | `approval / review_ready` |
| input-lock digest | `sha256:0baa6dd59fe575050dd70a99ebf65ef7012f944728ae275bf074bd56de9358b0` |
| discovered-facts digest | `sha256:e5a02bf60de0dec1745726e771eb7467a106b50a557500ae10430c951b9e06db` |
| plan digest | `sha256:90312a9e965e328555b83edad7d05c4c36ff5ea07a53b11b08513c33c8226f54` |
| apply context digest | `sha256:3234595904648596c590ad4c3286f92abd81d824b339d166d54e38a77b77278e` |
| candidate commit | `2fb8abb0debfa71aac0d2d39c45eafaaec483b60` |
| candidate tree | `c358b15d55a6cb654e1ff875b1010e0b3c9e089a` |
| apply verdict/state | `pass / static_consistent_awaiting_verify` |
| result digest | `sha256:5abb16084d9763ed9a829d16c12747c64307342976a83a572d6b8af1e8e99be3` |
| verify eligible | `true` |

verify 인계 테스트:

`tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip`

등록자료 시작·종료 digest는 모두 `sha256:2cb439...b5be5`로 같고, 모든 apply gate가 pass했다. fast check는 구조화된 N/A 사유를 결과에 보존했다.

### 5.2 계획 밖 경로+Manifest 확장 probe

산출물:

- plan: `work/om-apply-rehearsal-20260824-02/probe-plan-run`
- apply: `work/om-apply-rehearsal-20260824-02/probe-apply-run`
- 제품 branch: `codex/rehearsal-om-apply-p0-scope-probe-20260824`

| 값 | 결과 |
|---|---|
| plan verdict/state | `approval / review_ready` |
| plan digest | `sha256:afb0a899599b223673993c5e6414db7ff14e07d8b9d0bc4036d43f1da5f6a516` |
| apply context digest | `sha256:6deb3d0a82175ba0ce129d454ba7f08fa41d17e23cf65469c10312f81f4c71fc` |
| candidate commit | `24ab579b26ea929bdb404323e5760d9fe1efa962` |
| candidate tree | `77137ed340ae673fd5f1079c36ae09928eeb021e` |
| apply verdict/state | `block / blocked` |
| result digest | `sha256:bc2b9b7fe9fb0061cfa1fb3ba84f37b85ea28c502899e5deda5af77379c3da15` |
| verify eligible | `false` |

probe는 다음을 각각 검출했다.

- 계획 밖 파일이 unit에 배정되지 않음
- BANK-OM-005의 계획된 `changed_paths`와 최종 Manifest 불일치
- 구조화된 scope variance에 대한 사람 승인 질문

P0-1 수정으로 전역 오탐은 제거됐지만 범위 이탈 차단은 약해지지 않았다.

## 6. 금지사항·안전 상태

- push: 0
- MR: 0
- remote 쓰기: 0
- 원본 검사기 commit: 0
- 활성 등록자료 수정: 승인된 `shared-path-owners.yaml` 두 경로 추가만
- 과거 blocked run 재사용: 0
- 제품 commit: 새 격리 예행연습 branch의 normal·probe 각 1건만

## 7. 남은 위험과 다음 단계

이번 지시 범위 밖 항목은 임의로 고치지 않았다.

1. P0-2 활성 candidate lock 대조는 기존 R-1·R-2 기준선 잠금 재검토와 함께 처리해야 한다.
2. 실제 Contract test는 아직 실행하지 않았다. 정상 결과는 `/om-verify` 인계 자격만 뜻한다.
3. 구조화된 fast-check N/A의 사유가 사실인지 기계적으로 판별하지는 못한다. 운영 정책에서 허용 주체와 허용 조건을 정해야 한다.
4. scope variance가 최종적으로 block되지만 개별 `scope` gate 행은 pass로 보이는 표시 문제(P1-2)는 남아 있다.
5. upgrade의 독립 공식문서 판독은 별도 단계이며, 증거가 없으면 계속 STOP하는 것이 정상이다.

다음 작업은 이 결과를 Claude 반례 검증에 넘기는 것이다. 검증 전 commit·push·MR은 하지 않는다.

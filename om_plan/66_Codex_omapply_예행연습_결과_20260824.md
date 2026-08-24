# Codex 결과 — om-apply v1 실데이터 예행연습

작성일: 2026-08-24  
정본 지시: `65_Codex_omapply_예행연습_지시_20260824.md`

## 1. 결론

실제 OpenMetadata 1.13.1 등록자료가 가리키는 활성 후보 `8ac18ad0...`를 기준으로 change·반례·upgrade 흐름을 실행했다.

| 항목 | 실제 결과 | 의미 |
|---|---|---|
| BANK-OM-005 계획 | `approval / review_ready` | 대상 ID·경로·계약 테스트 관계와 입력 고정은 정상 |
| BANK-OM-005 적용 | `block / blocked` | 정상 단일 ID 변경인데도 전체 112개 Manifest 경로를 이번 diff에 요구하는 결함 발견 |
| verify 인계 자료 | 내용은 정확, `eligible: false` | 후보 SHA/tree와 BANK-OM-005 계약 테스트만 계산됐지만 apply가 막혀 인계 불가 |
| 계획 밖 경로+Manifest 확장 | `block / blocked` | 실제 데이터에서도 범위 이탈과 관리자료 불일치를 검출 |
| touch→원복 | `block / blocked` | 최종 tree 효과가 없는 변경을 검출 |
| 1.13.2 upgrade 계획 | `block / not_ready` | 별도 LLM 컨텍스트의 공식 문서 판독 증거가 없어 정확히 STOP |

따라서 **반례 차단은 작동하지만 정상 change를 통과시키지 못하므로 om-apply v1은 아직 verify 인계 또는 운영 도입 단계가 아니다.** 필수 목표였던 `static_consistent_awaiting_verify`에는 도달하지 못했으며, 이를 억지로 PASS 처리하지 않았다.

## 2. 유효한 기준과 예비 실행 제외

### 2.1 이번 결과의 정본 후보

활성 등록자료 `commit-inventory.yaml`은 다음 후보를 가리킨다.

- commit: `8ac18ad053d9274774e274ba17b35911ac0b9dcb`
- tree: `e86980f6d71295465fe6e75e5169fab57cebd4c4`
- 등록자료 위치: `harness/registrations/om-temp-1.13.1`
- 등록자료 digest: `sha256:9cc42f0f4d4368df46d509f24c43942d6c9cf20f47013eac76818752d8a23c3d`

change·범위 이탈·touch→원복·upgrade의 최종 기록은 모두 이 기준으로 다시 실행한 결과다.

### 2.2 제외한 예비 실행

초기에는 로컬 `custom/om-1.13.1`의 `59dae915...`를 현재 기준으로 잘못 선택했다. 이 commit의 tree `1df344b6...`는 활성 후보의 tree와 다르다.

이 예비 실행은 다음 두 사실을 발견하는 용도로만 남긴다.

1. plan은 요청자가 적은 `custom_baseline`을 고정하지만, 그것이 활성 `commit-inventory.yaml`의 `custom_head_sha`와 같은지 대조하지 않는다.
2. 잘못 선택한 기준에서도 정상 change가 전역 112개 경로 검사에 막혀 같은 결함이 재현됐다.

예비 실행의 `change-*`, `probe-*`, `touch-*`, 기존 `upgrade-*` run과 digest는 **통과 근거·활성 후보 근거로 사용하지 않는다.** 유효 산출물은 이름에 `active`가 들어간 run이다.

## 3. 저장소·격리·안전 상태

### 3.1 검사기

- 원본 작업장: `work/kb-datacatalog-upgrade-checker-om-plan-cli`
- 브랜치: `codex/om-plan-verified-gates-20260820`
- 최종 HEAD: `7c544efb2bc12200cf4b9e7dfef82d5358f29812`
- 최종 tree: `48370424965a4067db98967b0f002f53d7413d01`
- 원본의 기존 미커밋 om-apply 구현: 그대로 보존
- 실행용 독립 복사본 검사기 snapshot: `6f19716b5213bc903ed448af7043bac14235fb13`
- snapshot tree: `7c6cdf495da5a50afa08caba714f2bd51a4288f7`

실행용 검사기 복사본:

- `work/om-apply-rehearsal-20260824-01/checker-s1`
- `work/om-apply-rehearsal-20260824-01/checker-probe-active`
- `work/om-apply-rehearsal-20260824-01/checker-upgrade`

### 3.2 제품 예행연습 브랜치

| 용도 | 브랜치 | 최종 commit | 최종 tree |
|---|---|---|---|
| 정상 change | `codex/rehearsal-om-apply-change-active-20260824` | `2c1088a4f9352a525a3830b4ec074c20d5d69dc3` | `c358b15d55a6cb654e1ff875b1010e0b3c9e089a` |
| 범위 이탈 | `codex/rehearsal-om-apply-scope-probe-active-run-20260824` | `17b5718772a617cb8d0140700d3490de120a9edb` | `3985315247bfed0816d4d99ef5e32e5560567e38` |
| touch→원복 | `codex/rehearsal-om-apply-touch-revert-active-20260824` | `5a9222983c12001ccc858aee8c21f6c61a7276ba` | `e86980f6d71295465fe6e75e5169fab57cebd4c4` |
| upgrade 조사 | `codex/rehearsal-om-apply-upgrade-20260824` | `2763bf97ce265662793a1a38d353147cc6d6c2e3` | `c00c04455fcdc41b8e9959990ef82750c54d4d1e` |

change와 반례 제품 commit에는 `Customization-ID: BANK-OM-005` 트레일러가 있다. 모두 로컬 예행연습 브랜치이며 push하지 않았다.

### 3.3 등록자료 격리

활성 원본 18개 파일을 독립 검사기의 다음 경로로 복사해 사용했다.

`harness/registrations/rehearsal-om-temp-1.13.1`

- 활성 원본과 정상 시나리오 복사본의 `diff -qr`: 빈 출력
- 정상 시나리오 시작·종료 digest: 모두 `sha256:9cc42f...23c3d`
- 범위 이탈 복사본 종료 digest: `sha256:1b0121e9f1a9605a60c76ca521f4dce4132079e8a952cd868c7a53a5125031d7`
- 활성 등록자료 원본 변경: 없음

## 4. 주요 실행 산출물

| 용도 | 경로 |
|---|---|
| 활성 change 요청 | `work/om-apply-rehearsal-20260824-01/change-active-request.yaml` |
| 활성 change plan | `work/om-apply-rehearsal-20260824-01/change-active-plan-run` |
| 활성 change apply | `work/om-apply-rehearsal-20260824-01/change-active-apply-run` |
| 활성 범위 이탈 apply | `work/om-apply-rehearsal-20260824-01/probe-active-run-apply-run` |
| 활성 touch→원복 apply | `work/om-apply-rehearsal-20260824-01/touch-active-apply-run` |
| 활성 upgrade 요청 | `work/om-apply-rehearsal-20260824-01/upgrade-active-request.yaml` |
| 활성 upgrade plan | `work/om-apply-rehearsal-20260824-01/upgrade-active-plan-run` |

Python은 기존 `work/review-openmetadata-test/.venv/bin/python`을 사용했고, `PYTHONPATH`는 각 독립 검사기의 `harness`로 지정했다.

## 5. 시나리오 1 — 실제 BANK-OM-005 change

### 5.1 om-plan 결과

수집된 실제 관계:

- 변경 경로 2개: `SchemaEditor.tsx`, `SchemaEditor.test.tsx`
- 필수 경로: `SchemaEditor.tsx`
- 계약: `CONTRACT-KOREAN-IME`
- 필수 계약 테스트: `tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip`
- Registry·Manifest·Contract의 ID 관계: 일치

판정과 결속값:

- `verdict: approval`
- `review_state: review_ready`
- request digest: `sha256:bc192a5a81057bf10259bb636179f414783ef892f8d846710666730161ede685`
- input-lock digest: `sha256:540e9921b48eef088c35ab3ad7771a932b689e67eea9a173a877fb1c342d4d23`
- discovered-facts digest: `sha256:11187565b1225d849793237032205d8723c7b3f3d3f25e23210bfa373a39a5d7`
- proposal digest: `sha256:7b29f191676e3d64738ccefbf0dd644ff088ccb63294b3067a1e715a3188cf68`
- plan digest: `sha256:23fd925782b5e4a19cc3561f6a7a8ed00be71c4d0fcdbec6788de286d6eda432`

### 5.2 LLM 반영과 제품 commit

- 구현: `handleCompositionEnd`의 최종값 동기화 코드를 `synchronizeComposedValue` helper로 분리
- 실제 diff: `SchemaEditor.tsx` 한 파일, `+6/-2`
- 후보 commit: `2c1088a4f9352a525a3830b4ec074c20d5d69dc3`
- 후보 tree: `c358b15d55a6cb654e1ff875b1010e0b3c9e089a`
- apply context digest: `sha256:00ef984a8f366b19f17b677d851087271be049fac5fe8ffe9a2e40fe4df8caab`
- 실행 보고서: `ime-ui` unit 완료, scope variance 없음, 계약 실행 주장 없음

### 5.3 apply check 결과

- `verdict: block`
- `final_state: blocked`
- result digest: `sha256:4d2a987499d60798676f32c0f603e85ae0ed69b44c20feb4a478961d82e94d56`
- 사유: 112개
- 차단 gate: `management_three_way`
- 그 전 gate: execution units·planned actions·scope·assertions 모두 pass
- fast checks: 앞 gate STOP 때문에 `skipped_due_to_stop`

정상 변경이 차단된 직접 원인은 `validate_management`가 모든 ID의 `changed_paths`를 `final_owned_paths`로 합친 뒤, 이번 BANK-OM-005 transaction diff에 그 경로가 전부 존재하거나 `requires_diff: false`여야 한다고 요구하기 때문이다.

근거:

- `harness/acgh/integrations/om/apply.py:118-123` — 전체 ID의 경로를 합침
- `harness/acgh/integrations/om/apply.py:140-147` — 전체 경로를 이번 transaction diff와 비교

단일 ID change에서는 다른 ID 경로가 변하지 않는 것이 정상이다. 이는 데이터 오류가 아니라 **정상 change를 막는 P0 검사기 결함**이다.

### 5.4 verify 인계 payload

```text
candidate_sha:       2c1088a4f9352a525a3830b4ec074c20d5d69dc3
candidate_tree_sha:  c358b15d55a6cb654e1ff875b1010e0b3c9e089a
required_tests:      tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip
contract_status:     claim_only_not_executed
eligible:            false
```

다른 ID 테스트는 섞이지 않았다. 다만 apply가 block이므로 `/om-verify` 인계 자격은 없다. 이 예행연습에서 실제 Contract test 통과를 주장하지 않는다.

## 6. 반례 probe

### 6.1 계획 밖 파일+Manifest 확장

제품 후보에 계획 밖 파일을 추가했다.

`openmetadata-ui/src/main/resources/ui/src/components/Database/SchemaEditor/om-apply-scope-probe.txt`

동시에 활성 원본이 아닌 probe 복사본의 BANK-OM-005 Manifest에 해당 경로를 추가하고, 구조화된 범위 이탈 사유를 실행 보고서에 기록했다.

- 후보 commit: `17b5718772a617cb8d0140700d3490de120a9edb`
- 후보 tree: `3985315247bfed0816d4d99ef5e32e5560567e38`
- apply context digest: `sha256:0615a1d8ecdb6a6ea35fd3c48383c3a3f437047d32891013c4e031a9550b7ff9`
- `verdict: block`
- `final_state: blocked`
- result digest: `sha256:2ad6b33b85ea0411d3fe27cdc014c51e723f1254e5191bd5803549479522016d`
- 사유: 114개

실제 검출:

- `ime-ui` unit에 배정되지 않은 변경 경로
- BANK-OM-005의 계획된 `changed_paths`와 최종 Manifest 불일치
- 범위 이탈 승인 질문 생성
- 기존 전역 112개 경로 문제와 공용 소유정보 문제

핵심 질문:

```text
approve scope variance for .../om-apply-scope-probe.txt:
반례 검증을 위해 계획 밖 파일과 매니페스트 확장을 의도적으로 추가했다.
```

따라서 **제품 코드와 Manifest를 함께 넓혀도 자동 통과하지 않았다.**

추가 관찰: 개별 `scope` gate 행은 pass로 표시되지만 범위 이탈 질문이 최종 상태를 block한다. 최종 안전성은 유지되지만 관리자 화면에서는 “범위 이탈 발견·사람 승인 대기”가 바로 보여야 한다.

### 6.2 touch 후 원복

같은 경로를 한 commit에서 수정하고 다음 commit에서 원복했다.

- 변경 commit: `d7427615051bbf0603997d08edc2d8b070e83a41`
- 원복 후 commit: `5a9222983c12001ccc858aee8c21f6c61a7276ba`
- 원복 후 tree: `e86980f6d71295465fe6e75e5169fab57cebd4c4`
- 시작 tree와 최종 tree: 동일
- apply context digest: `sha256:4edaa14c33ec16ae6c0ea87a7c5b62217f0bace30c36d03174c145f1b418926b`
- `verdict: block`
- `final_state: blocked`
- result digest: `sha256:3ac6dd9ca9d620c2af227d8f59ebf6a69b6615559992d430709edb8b0cc18f4e`
- 사유: 115개

핵심 검출:

```text
required action has no final tree effect (possible touch/revert): ime-source-refactor
assertion ime-helper-present fragment is absent
```

이 반례는 의도대로 작동했다.

### 6.3 순서 위반 probe

한 예비 probe에서 제품 변경 commit을 먼저 만든 뒤 `apply start`를 호출했다. 검사는 `APPLY_START_NOT_CHECKED_OUT`으로 거부했다. 이후 정식 probe는 apply start 뒤에 제품을 변경하는 올바른 순서로 새로 실행했다.

즉, apply가 요구하는 시작 SHA를 체크아웃하지 않은 상태에서는 작업을 시작할 수 없었다.

## 7. 시나리오 2 — 활성 1.13.1 → 공식 1.13.2 upgrade

### 7.1 고정된 세 기준

- 공식 1.13.1 commit/tree: `afcb2d2c...` / `a8290d23...`
- 공식 1.13.2 commit/tree: `2763bf97...` / `c00c0445...`
- 활성 custom 1.13.1 commit/tree: `8ac18ad0...` / `e86980f6...`
- 버전 관계: `adjacent`

### 7.2 수집·path-remap 결과

- 공식 변경: 1,702개 경로
- 등록 경로: 112개
- 교집합: 33개
- path-remap: 33개 모두 제안
  - `keep`: 32개
  - 사람 검토 대기 replacement: 1개
- 대기 경로:
  - from: `openmetadata-ui/src/main/resources/ui/src/utils/EntityUtils.tsx`
  - to 후보: `openmetadata-ui/src/main/resources/ui/src/utils/EntityUtilClassBase.ts`
  - action: `replace_pending_human_review`

계획 binding:

- request digest: `sha256:dd1e841babe988dfce97d4d2235f6688ca5ae46836ff0fb698b820fc3903cb21`
- input-lock digest: `sha256:5bce5e43c01903c0b75918c6f19d6e31e7fb1856f616fa30fe50b81dc4471647`
- discovered-facts digest: `sha256:d5009cb9b74b7edec3f0a0a00bf215b8b4fafda1d5c8eff5571805e3bb95ce8f`
- proposal digest: `sha256:3121b7e2e2be04d7fb978c29f13b4acbc6153bdea5cd87ccb2e64ca680e62182`
- plan digest: `sha256:f66b872ab2d79e0f7727b45d24db6af71485de07ed60a0b7cf1deeabea0f2f02`

### 7.3 공식 문서 snapshot

| 자료 | 이번 활성 run의 byte digest |
|---|---|
| GitHub 1.13.2 release | `sha256:d12a41d293cb46018de6359cc03ac34fdda946e05212ad2a6ca47f3123e13f2e` |
| OpenMetadata 1.13.x upgrade guide | `sha256:a30750d9abfef6704fa8ae0721cf61cc45325ba6c9c2c3810f3bd0fac752caa5` |

같은 세션의 앞선 예비 run과 비교했을 때 upgrade guide bytes는 같았지만 GitHub release HTML은 파일 크기가 같은데 digest가 달라졌다. 검사기는 각 run의 실제 bytes를 보존하므로 바뀐 사실은 숨겨지지 않는다. 다만 URL 자체는 불변 문서가 아니므로 승인 근거는 반드시 보존된 snapshot digest에 연결해야 한다.

### 7.4 STOP 결과

- `verdict: block`
- `review_state: not_ready`
- dirty product/checker 경로: 없음
- 등록자료 stale: false
- evidence pointer 오류: 없음
- 차단 사유: 정확히 1개

```text
upgrade requires exactly one independent_document_review from a separate LLM context
```

이번 실행은 단일 Codex 컨텍스트이므로 별도 검토자로 가장한 evidence를 만들지 않았다. plan approval 전에 정직하게 멈췄기 때문에 apply start·실제 재적용·병합·3단 복구에는 진입하지 않았다.

이는 지시서가 허용한 유효한 STOP이다. 3단 복구가 실패한 것이 아니라 실행 조건에 도달하지 않은 것이다.

추가 관찰: 두 공식 commit의 tree는 모두 있어 직접 diff 1,702개를 계산했지만 `git merge-base --is-ancestor`는 exit 1이었다. 로컬에 반입된 공식 ref의 계보가 연결되지 않았으므로 실제 rebase/merge 전에는 출처와 계보를 별도 검사해야 한다.

## 8. 발견 문제와 위험

### P0-1. 정상 단일 ID change를 막는 전역 경로 검사

`validate_management`가 이번 계획 대상이 아니라 모든 등록 ID의 112개 경로를 이번 transaction diff에 요구한다. `all_owned_paths`는 등록 밖 변경 검출에, `planned_owned_paths`는 이번 계획의 필수 변경 검출에 각각 사용하도록 분리해야 한다.

### P0-2. 활성 후보와 요청 후보의 일치 확인이 없음

plan은 요청서의 `custom_baseline`을 SHA/tree로 정확히 고정하지만, 활성 등록자료의 `commit-inventory.yaml.range.custom_head_sha`와 같은지는 확인하지 않는다. 실제로 `59dae915...`를 넣은 예비 run도 approval까지 갔다.

현재 collector는 등록 Registry의 `source`를 과거 provenance로만 수집하며(`collectors.py:228-232`), `commit-inventory.yaml`은 읽지 않는다. 과거 provenance를 active lock으로 오용해서는 안 되지만, 운영에서 “현재 승인 기준”을 별도 active-candidate lock으로 정하고 요청 후보와 대조하는 규칙은 필요하다.

### P0-3. 계획 전에 잡히지 않은 공용 경로 소유정보 불일치

Manifest 관계상 공유 경로는 39개지만 `shared-path-owners.yaml`에는 37개만 있다. 누락:

- `openmetadata-ui/src/main/resources/ui/src/generated/entity/services/connections/serviceConnection.ts`
- `openmetadata-ui/src/main/resources/ui/src/utils/DatabaseServiceUtils.test.tsx`

두 경로는 BANK-OM-006·007에 이미 함께 존재한다. 신규 공유 경로가 아니라 기존 등록자료 불일치인데, change plan은 approval되고 apply에서야 “new shared path”로 나타났다. 활성 자료는 이번 작업에서 수정하지 않았다.

### P1-1. 빠른 검사 없는 계획이 approval 가능

change 계획의 `fast_checks`가 비어 있어도 approval됐다. 설계가 컴파일·생성기 등의 빠른 검사를 요구한다면 최소 한 개 또는 구조화된 `N/A + 사유`를 강제해야 한다. 이번 run은 앞 gate 차단 때문에 fast check가 실행되지 않았다.

### P1-2. 범위 이탈 표시가 직관적이지 않음

범위 이탈이 있어도 `scope` gate 행은 pass이고 unresolved question이 최종 상태를 block한다. 최종 안전성은 유지되나 관리자 요약은 “범위 이탈 발견·승인 대기”로 표시해야 한다.

### P1-3. 공식 문서 URL의 raw bytes가 run 사이에 바뀜

동일 GitHub release URL의 HTML이 짧은 간격의 두 run에서 서로 다른 digest를 냈다. snapshot 보존은 작동했지만 URL만으로는 재현할 수 없다. 문서 판독과 승인은 run의 byte digest에 결속해야 한다.

### P1-4. 공식 ref 계보와 partial clone 준비 부족

공식 tree diff는 가능했지만 ancestor 관계는 확인되지 않았다. 또한 일부 로컬 clone은 blobless/partial clone이라 `GIT_NO_LAZY_FETCH=1`에서 필요한 blob이 없어 checkout이 실패했다. 외부망 반입 전 commit·tree뿐 아니라 필요한 blob과 계보가 실제로 있는지 preflight해야 한다.

### P1-5. 111과 112의 기준을 함께 표시해야 함

- `source-diff-paths.txt`: 111개 — 최초 공식 원본 대비 등록 시점 기준
- `current-diff-paths.txt`·현재 ID 관계: 112개 — 현재 후보 기준

A5 결정대로 `changed_path_count`를 원본 111개로 유지하는 것은 맞다. 다만 운영 화면에는 두 숫자의 기준을 함께 써야 drift로 오해하지 않는다.

### P2-1. 격리 복사 중 검사기 원본 ref에 생긴 로컬 snapshot commit

원본 검사기 작업장은 일반 clone이 아니라 Git worktree라 `.git`이 디렉터리가 아닌 파일이었다. `.git/`만 제외한 첫 복사가 그 파일을 포함했고, 격리용 `git init/commit`이 원본 branch에 로컬 commit `6f19716b...`를 잠시 만들었다.

발견 즉시 `git reset --mixed 7c544efb...`로 원본 ref만 복원했다. hard reset·파일 폐기·push는 없었고 기존 미커밋 구현도 보존됐다. 현재 원본 HEAD는 `7c544efb...`다. 다만 commit object가 reflog/dangling 상태로 남아 있으므로 과정상 “검사기 커밋 금지”는 엄격히 충족하지 못했다.

후속 격리는 `.git` 파일과 디렉터리를 모두 제외하고 `git rev-parse --git-dir`가 원본을 가리키지 않는지 확인해야 한다. 연결 흔적이 있는 `checker-template`은 재사용하면 안 된다.

## 9. 수용 기준 판정

| # | 판정 | 근거 |
|---|---|---|
| 1 | **미충족** | 활성 기준 정상 change가 `block/blocked`; `static_consistent_awaiting_verify` 미도달 |
| 2 | **내용만 충족** | 후보 SHA/tree와 BANK-OM-005 계약 테스트는 정확하지만 `eligible:false` |
| 3 | **충족** | 계획 밖 파일+Manifest 확장을 unit 불일치·관리자료 불일치·승인 질문으로 차단 |
| 4 | **미충족** | 정상 diff는 ID에 귀속됐으나 전역 경로 검사와 기존 공유 소유정보 불일치로 3방향 정합 block |
| 5 | **충족** | 활성 기준 upgrade가 독립 문서 검토 부재 한 건으로 STOP; 3단 복구는 미진입으로 기록 |
| 6 | **부분 충족** | push·MR·활성 등록자료 변경은 0이고 최종 checker HEAD도 원복됨. 다만 과정 중 로컬 snapshot commit이 원본 ref에 잠시 생성됨 |

## 10. 다음 조치

1. `validate_management`의 전체 소유 경로와 이번 계획 경로 역할을 분리하고 정상 change·범위 이탈 반례를 함께 고정한다.
2. 활성 후보 lock의 정본을 정하고 plan 요청 후보와 재계산 대조한다. `commit-inventory`가 provenance인지 active lock인지도 명시한다.
3. 공용 관계 39개와 소유정보 37개의 차이를 apply와 분리된 registration preflight에서 먼저 처리한다.
4. fast check 최소 조건 또는 명시적 `N/A` 사유를 강제한다.
5. 수정 후 **새 active change run**으로 재실행한다. 기존 blocked run은 PASS 근거로 재사용하지 않는다.
6. 별도 LLM 컨텍스트가 이번 active upgrade run의 정확한 문서 snapshot digest를 판독한 뒤 같은 plan run에 새 validation attempt를 추가한다.
7. change가 `static_consistent_awaiting_verify`에 도달한 뒤에만 실제 Contract test 단계로 넘긴다.

## 11. 최종 안전 상태

- push: 0
- MR: 0
- 원격 쓰기: 0
- 활성 등록자료 원본 변경: 0
- 검사기 원본 최종 HEAD: `7c544efb2bc12200cf4b9e7dfef82d5358f29812`
- 검사기 원본의 기존 미커밋 om-apply 구현: 보존
- 제품 변경: 로컬 예행연습 브랜치에만 존재
- 공식 upgrade apply·병합·충돌 해결: 미실행
- 실제 Contract test: 미실행

이 문서는 “om-apply 준비 완료” 보고가 아니다. **활성 실데이터에서 정상 경로를 막는 P0와 운영 입력의 정본 공백을 찾아낸 검증 결과**다. 위 결함을 고치고 새 run으로 재검증하기 전에는 verify 인계나 운영 도입을 승인하면 안 된다.

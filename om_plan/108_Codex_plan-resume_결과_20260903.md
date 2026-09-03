# Codex 결과 — plan-resume 경로 복구

- 작업일: 2026-09-03
- 지시서: `107_Codex_plan-resume_핫픽스지시_20260903.md`
- 작업 트리: `work/kb-datacatalog-upgrade-checker-om-plan-cli`
- 브랜치: `hotfix/verify-start-condition-20260902`
- 시작 HEAD: `a2fcf6cb40ce8b491d074306e21bb5e32e024c8f`
- 상태: 작업 트리만 수정. commit·push·MR 없음

## 1. #20 P0 — resume 후 trusted input-lock digest 복구

### 변경

- `harness/acgh/plancore/resume.py:19-43`
  - 기존 불변자료 검사 함수가 검증을 마친 `input_lock_digest`를 반환하게 했다.
- `harness/acgh/plancore/resume.py:89-97`
  - run 재바인딩 성공 직후 같은 digest를 `record_trusted_input_lock_digest`로 새 세션 마커에 기록한다.
  - digest 기록이 실패하면 방금 만든 run·session marker 쌍을 정리하고 실패를 반환한다.

digest는 외부 인자나 proposal에서 받지 않는다. 해당 run의 `input-lock.yaml`을 스키마와 자체 digest로 먼저 검증한 뒤 그 값만 사용한다. input-lock 또는 discovered-facts가 바뀌었을 때의 기존 차단 순서도 유지했다.

### 반례

| 반례 | 결과 |
|---|---|
| block run 재개 후 새 마커에 원래 `input_lock_digest`가 기록됨 | pass |
| 재개 후 `plan check`를 외부 digest 입력 없이 실행 | `approval` 도달 |
| block 이후 input-lock 변조 | `RESUME_INPUT_LOCK_CHANGED`로 차단 |
| 기존 plan start 경로 | trusted digest 기록 유지 |

테스트 위치:

- `harness/tests/test_plan_workflow.py:1712-1734`
- `harness/tests/test_plan_workflow.py:1737-1760`
- `harness/tests/test_plan_cli.py:296-305`

## 2. #21 P1 — proposal 구문 오류에 한정한 resume

### 변경

- `harness/acgh/plancore/resume.py:46-65`
  - `analysis_error`의 최신 validation attempt를 확인한다.
  - 원인이 하나뿐이고, 그 원인이 `proposal/` 아래 YAML·JSON 파일의 `INPUT_UNREADABLE`일 때만 재개 대상으로 인정한다.
- `harness/acgh/plancore/resume.py:76-89`
  - 기존 `block` 또는 위 조건을 만족하는 proposal 구문 오류만 같은 run에서 재개한다.
  - input-lock·facts 불변 검사는 두 경로 모두 동일하게 실행한다.

허용 범위를 proposal 파일의 읽기·파싱 실패 한 건으로 제한했다. 입력자료 손상, 여러 오류가 섞인 analysis_error, 승인 완료 run은 계속 `RUN_NOT_PROPOSAL_REVISABLE`로 거부한다.

### 반례

| 반례 | 결과 |
|---|---|
| proposal YAML 구문 오류 → `analysis_error` → resume → proposal 수정 → 재검증 | `approval` 도달, attempt 2개 보존 |
| 같은 proposal 구문 오류 후 input-lock 변조 | `RESUME_INPUT_LOCK_CHANGED`로 차단 |
| input-lock 자체가 읽히지 않아 발생한 `analysis_error` | `RUN_NOT_PROPOSAL_REVISABLE`로 차단 |
| 기존 정상 block resume | 기존 append-only 재검증 통과 |
| 이미 승인된 run resume | 기존대로 거부 |

테스트 위치:

- `harness/tests/test_plan_workflow.py:1763-1789`
- `harness/tests/test_plan_workflow.py:1792-1818`
- `harness/tests/test_plan_workflow.py:1821-1840`
- 기존 `test_r04_validation_attempts_are_append_only_for_proposal_fix`
- 기존 `test_r07_completed_run_cannot_be_revalidated`

## 3. 검증 결과

| 구분 | 결과 |
|---|---:|
| resume·start·check 집중 회귀 | 10 passed |
| 전체 테스트 | 301 passed, 10 skipped |
| `git diff --check` | pass |
| 변경 Python `compileall` | pass |

skip 10건은 pass에 포함하지 않았다.

## 4. 변경 파일

```text
M  harness/acgh/plancore/resume.py
M  harness/tests/test_plan_cli.py
M  harness/tests/test_plan_workflow.py
```

`harness/om_workflow.py`, 제품 코드, 활성 등록자료, 스킬 본문과 references는 수정하지 않았다.

## 5. 남은 한계

- proposal 파싱 오류 여부는 현재 validation attempt의 `reasons`에 저장된 오류 코드·경로를 엄격히 대조한다. 결과 스키마에 별도 구조화 원인 필드가 없기 때문에 문자열 형식이 바뀌면 허용하지 않고 안전하게 중단한다.
- 이번 변경은 proposal YAML·JSON 읽기/구문 오류만 복구한다. proposal 누락·빈 폴더·symlink 같은 다른 `analysis_error`까지 자동으로 재개하지 않는다.
- 실제 운영 세션에서의 `plan check → block/analysis_error → plan-resume → plan check` 재현은 Claude 후속 검증 대상으로 남긴다. 코드 반례에서는 두 경로 모두 끝까지 실행했다.

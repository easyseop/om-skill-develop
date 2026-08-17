# Codex 전달 — /om-plan CI: catalog digest의 .pyc 결함(P2-1) 수정

> 작성: 2026-08-15. 이 문서를 코덱스에게 그대로 전달한다.
> 근거: `~/Desktop/om-plan_CI배선_적대검토_결과_20260815.md`(적대 검토 결과).

## 1. 배경 (한 줄)

CI 배선 구현(commit `9bc4cd2c3dca064d2d181b4e500415e18c336e9c`)은 보안 경계(A·B·C·D)가
튼튼하고 **P0·P1 없음**으로 검토를 통과했다. 다만 **실환경에서 파이프라인이 review_ready에
도달하지 못할 수 있는 견고성 결함 1건(P2-1)**을 고친 뒤 재검토해야 한다.

## 2. 고칠 것 — P2-1: checker_catalog_digest가 `__pycache__/*.pyc`를 포함한다

**증상:** `plan-validate`가 재계산하는 `checker_catalog_digest`에 Python 바이트코드
캐시(`__pycache__/*.pyc`)가 섞여 들어간다. import 시점·checkout 시각에 따라 `.pyc`
바이트가 달라져, preflight job과 validate job의 catalog digest가 어긋나면
`recomputed input lock does not match the stored lock` → `analysis_error`가 된다.
**fail-closed라 false pass는 아니지만, 정상 계획도 통과에 도달하지 못할 수 있다.**

**근거 위치:**
- `harness/acgh/plancore/paths.py`의 `directory_digest` — `root.rglob("*")`의 모든 regular
  file을 digest하며 `.pyc`/`__pycache__`를 제외하지 않는다.
- `harness/acgh/plancore/preflight.py`의 `_checker_catalog_digest` → `directory_digest(plancore)`
  + `directory_digest(integrations/om)`.

**재현(로컬):**
- 지정 tree를 git 저장소로 만들고 `PYTHONDONTWRITEBYTECODE` 없이 cross-runner 재검증 →
  `analysis_error`(reasons: `recomputed input lock does not match the stored lock`).
- `PYTHONDONTWRITEBYTECODE=1`로 같은 실행 → `approval`, reasons 없음.
- 선언된 회귀 "108 passed"도 그 flag가 있을 때만 성립한다(없으면 107 passed / 1 failed:
  `test_cross_runner_fresh_validation_uses_ci_digest_and_recomputed_sources`). 그런데
  **workflow(.github/workflows/om-plan-ci.yml)는 그 flag를 설정하지 않는다.**

## 3. 수정 방향 (택1, (a) 권장)

- **(a) 권장 — `directory_digest`에서 바이트코드 산물 제외.** `__pycache__` 디렉터리와
  `*.pyc`/`*.pyo`를 digest 대상에서 빼서 catalog가 순수 source의 함수가 되게 한다.
  근본 해결이며 로컬/CI 환경 차이에 무관해진다. 제외 규칙이 다른 정당한 파일을 삼키지
  않도록 최소 범위로 한정한다.
- (b) catalog를 **git-tracked source로만** 계산(예: `git ls-files` 기준). (a)와 취지는 같으나
  변경 범위가 크다.
- (c) 차선 — workflow의 preflight·validate job과 회귀 명령에 `PYTHONDONTWRITEBYTECODE=1`을
  설정. 증상은 가리지만 `directory_digest`의 근본 취약성은 남으므로 (a)와 병행이 아니면 비권장.

## 4. 반드시 성립해야 할 것 (수용 기준)

1. **`PYTHONDONTWRITEBYTECODE` 없이도** cross-runner 재검증이 정상 계획에서 `approval` /
   `review_state=review_ready` / exit 2가 나온다.
2. `test_cross_runner_fresh_validation_uses_ci_digest_and_recomputed_sources`가 그 flag
   없이 통과하도록 하는 **회귀 test를 추가**한다(현재는 flag 의존).
3. **위조 탐지는 그대로 유지.** catalog에서 `.pyc`를 빼도, 실제 source(예: `validate.py`·
   `preflight.py`)가 run 도중 바뀌면 여전히 `analysis_error`가 나야 한다. 이걸 확인하는
   반례 test를 남긴다(제외가 방어 구멍을 만들지 않았음을 증명).
4. E-hardening·CI 경계(A·B·C·D)의 기존 동작은 불변. `verdict.py` enum·exit code·제품 코드·
   활성 등록자료는 건드리지 않는다.

## 5. 손대지 말 것 / 이번 범위 아님

**변경 금지:** `verdict.py` 공용 enum·exit code, 제품 코드(openmetadata-ui/spec/java),
활성 등록자료(`harness/registrations/`).

**이번에 함께 고치지 말 것(별개 후속):**
- P2-2(all-skipped green과 approval=exit2 red의 신호 반전) — 이건 GitHub environment·소비자
  규약·Q9 결정이 필요한 별개 항목이다. 다만 원한다면 workflow에 "비-default branch dispatch는
  명시적으로 실패"하는 가드 job을 추가하는 것은 이번에 같이 해도 좋다(선택).
- 사람 intent 게이트(required reviewers)·`OM_PLAN_INTENT_REVIEW_ENFORCED`의 실제 GitHub
  환경 설정 — 저장소 owner의 몫.
- Q3·Q6·Q7·Q8·Q9 임의 확정 금지.

## 6. 검토 기준·참조

- 검토 결과 정본: `~/Desktop/om-plan_CI배선_적대검토_결과_20260815.md`
- 검토된 CI 구현: branch `codex/om-plan-ci-wiring-20260815`,
  commit `9bc4cd2c3dca064d2d181b4e500415e18c336e9c`, tree `ff83a8492ccbd4abc0e5448945d2bf6c614cb6ab`

## 7. 종료 조건

수정 후 §4의 수용 기준을 만족하는 test·실행 결과와 함께 보고한다. merge·release·배포는
자동 진행하지 않는다. 완료 보고가 오면 Claude가 다시 적대적으로 재검토한다(특히 §4-1·§4-3:
flag 없이 통과 + 위조는 여전히 잡히는지).

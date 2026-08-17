# /om-plan P2-1(.pyc) 수정 재검토 결과 (2026-08-15)

> 읽기 전용 재검토. 원본 저장소·code·test·문서·Git 상태를 수정하지 않았다. 지정 commit의
> tree를 `git archive`로 임시 디렉터리에 추출해 읽기 전용 import하고, 공격은 `/tmp` 임시
> 저장소에만 수행했다. 결과 파일 1개만 작성한다. Q3·Q6·Q7·Q8·Q9는 임의 확정하지 않았다.

## 검증 기준
- branch `codex/om-plan-ci-wiring-20260815`, **수정 commit `7aaa18f53e47a0a98185cfcb27f22966a637c025`**
  ("fix: ignore Python bytecode in checker digest"), 기준 `649454a4`(CI 배선 `9bc4cd2c`의 후속).
- 변경 3파일: `harness/acgh/plancore/paths.py`(+5), `harness/tests/test_plan_paths.py`(+24),
  `harness/tests/test_om_plan_ci.py`(+64). verdict enum·exit code·제품·registration·workflow 무변경.

# 결론: **채택.** P2-1이 정확히 수정됐고, CI에서 도달 가능한 새 false pass는 없다.

핵심 질문 "캐시 억제 없이 통과하면서 실제 소스 위조는 여전히 잡히는가"는 **둘 다 참**으로
실행 확인했다. 이론적 방어심화(defense-in-depth) 약화 1건(P2, CI 미도달)만 남는다.

---

## 1. 수정 내용 (paths.py)

`directory_digest`가 loop에서 **symlink 거부(raise)를 먼저** 하고, 그 다음에만
`"__pycache__" in relative.parts or child.suffix in {".pyc",".pyo"}`를 skip한다. 즉
symlink 경계는 skip보다 앞이라 유지되고, `.py` 등 실제 소스는 계속 digest된다.
`_checker_catalog_digest`는 그대로 `directory_digest`를 호출한다(예외 로직 없음).

## 2. 실행 검증 (모두 `PYTHONDONTWRITEBYTECODE` 없이)

| 검증 | 결과 |
|---|---|
| 추가된 `test_plan_paths.py`+`test_om_plan_ci.py` (18 test) | **18 passed** (flag 없이) |
| 독립 cross-runner PoC (정상 계획) | **exit 2 / verdict approval / review_state review_ready** |
| 독립 PoC: validate.py를 preflight 후 변조 | **exit 3 / analysis_error** (지문 불일치로 차단) |
| `directory_digest` 단위: `.pyc`·`__pycache__` 추가 | 지문 불변(제외됨) |
| `directory_digest` 단위: `.py` 내용 변경 | 지문 변함(탐지) |

→ 수정 전 실패 원인(cross-runner가 flag 없이 exit 3)이 해소됐고, `.py` 소스 위조 탐지와
symlink 경계는 유지된다. 수용 기준 4개 모두 충족.

## 3. P0 / P1
없음. CI 경계(A·B·C·D)와 E-hardening 동작은 불변이고, 새로 도달 가능한 false pass·격리
우회를 발견하지 못했다.

## 4. P2 (이론적·CI 미도달) — 제외된 `.pyc`가 지문 밖 실행 경로가 될 수 있다

- **실증:** benign `mymod.py`(`VALUE='benign'`)와 header(mtime·size)가 일치하도록 조작한
  `__pycache__/mymod.cpython-311.pyc`(body는 `VALUE='PWNED-by-crafted-pyc'`)를 심고 새
  프로세스에서 import하면:
  ```
  crafted .pyc 실행 결과: PWNED-by-crafted-pyc
  directory_digest 변화?: False   (.pyc가 제외돼 지문 불변)
  ```
  즉 `directory_digest`는 더 이상 **Python이 실제로 실행할 수 있는 bytecode**를 덮지 않는다.
- **CI 도달성: 없음(중요).** 검토한 workflow의 validate job은 checker를 `github.sha`에서
  **새로 checkout(신뢰·clean·__pycache__ 없음)**하고, Python이 신뢰 `.py`에서 `.pyc`를 즉시
  컴파일한다. 비신뢰 proposal은 `trusted-run/proposal`로 격리돼 checker에 못 쓴다. 따라서
  공격자가 checker의 `__pycache__`에 조작 `.pyc`를 미리 심을 창이 없다. **CI에서 새 false
  pass를 만들지 않는다.**
- **성격:** checker에 쓰기 가능한 로컬/hookless 환경에서만 성립하는 primitive 약화다. 그런
  환경이면 공격자는 `.py`도 바꿀 수 있지만 `.py` 변조는 지문에 잡히고, `.pyc`만이 유일한
  "실행되면서 지문에 안 남는" 경로가 된다.
- **권고(비차단 방어심화):** 조작 `.pyc`가 권위가 되지 못하게 (a) catalog 계산 전
  `__pycache__` 제거, (b) checker 실행을 `PYTHONDONTWRITEBYTECODE=1`/`sys.dont_write_bytecode`
  로 고정, 또는 (c) hash 기반 `.pyc` invalidation(`checked-hash`) 사용 중 하나. (a)가 가장 작다.
  CI 미도달이므로 이번 채택을 막지 않는다.

## 5. 중단 조건(handoff §8) 대조 — 모두 위반 없음
- 일반 소스·정책 파일까지 제외? **아니오**(`.py` 변경 탐지 확인).
- symlink 거부 우회? **아니오**(검사가 skip보다 먼저).
- 실제 checker 소스 변경이 성공으로 끝남? **아니오**(analysis_error 확인).
- exit 2를 0으로 바꿈? **아니오**(exit 2 유지, `run_fresh_validation`의 exit↔verdict 대조 불변).
- P2-2·미결정 Q 혼입? **아니오**(변경 3파일에 없음).
- 미실행 workflow를 운영 완료로 표시? **아니오**(handoff가 실환경 미실행 명시).

## 6. 최종 판정

**채택.** P2-1은 올바르게 수정됐다(캐시 없이 통과 + `.py` 위조 차단 + symlink 경계 유지,
실행으로 확인). 남은 것은 CI에서 도달 불가능한 이론적 P2(§4) 하나이며, `__pycache__` 제거나
bytecode 억제로 값싸게 닫을 수 있으나 이번 채택의 전제는 아니다.

## 다음 사람이 결정할 것 (권고 가능·임의 확정 금지)
1. §4 방어심화 반영 여부(비차단).
2. P2-2(green/red 신호)와 GitHub environment required reviewers 설정 — 별개, 실환경 필요.
3. 실제 GitHub Actions 실행·원격 push 여부. P0/P1 없으므로 사용자 결정 사항.

---
*읽기 전용 재검토. 원본 수정·commit·push·구현·설치 없음. 검토 commit
`7aaa18f53e47a0a98185cfcb27f22966a637c025`.*

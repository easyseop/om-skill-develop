# Codex 전달 — /om-plan: GitLab CI 보호 배선 (GitHub 미사용)

> 작성: 2026-08-15. 이 문서를 코덱스에게 그대로 전달한다.
> 전제: **GitHub / GitHub Actions 미사용. 배포 대상은 GitLab CI.**
> 근거: Claude 검토 결과 3종
> - `~/Desktop/om-plan_CI배선_적대검토_결과_20260815.md`
> - `~/Desktop/om-plan_pyc수정_재검토_결과_20260815.md`
> - `~/Desktop/om-plan_E안_구현재검토_결과_20260814.md`

## 0. 배경

지금까지 "보호 CI"는 GitHub Actions 워크플로(`.github/workflows/om-plan-ci.yml`)로
구현됐다. E안의 안전성은 GitHub이 주던 보증(신뢰 fresh checkout, job 격리, `needs` 그래프,
environment required reviewers, actions 권한)에 기대고 있었다. **GitLab로 옮기면 그 보증을
GitLab의 대응 기능으로 다시 세워야 한다.** 코드 로직(지문·검증·격리)은 재사용하고,
GitHub 전용 워크플로만 GitLab `.gitlab-ci.yml`로 대체한다.

## 1. 그대로 유효한 것 (재사용, GitLab과 무관)

- `harness/acgh/plancore/preflight.py`, `validate.py` — 지문·재검증·`--expected-input-lock-digest`
  게이트·`review_state`(E-hardening `1f02e1d3`, 채택됨).
- `harness/acgh/plancore/paths.py`의 `.pyc` 제외 수정(`7aaa18f`, 채택됨).
- `harness/ci/om_plan_ci.py`의 6개 함수(`prepare_request`·`capture_preflight`·`package_proposal`·
  `restore_run`·`rebind_run_request`·`run_fresh_validation`) — 경로·파일·subprocess만 다루므로
  GitLab에서도 그대로 호출한다. **재작성 불필요.**

## 2. 폐기/보류

- `.github/workflows/om-plan-ci.yml` — 배포 대상 아님(참조용 보존). GitHub environment·actions
  권한 전제 무효.

## 3. GitLab로 옮길 때 GitHub 보증 → GitLab 대응 매핑

E안 안전의 5개 성질을 GitLab 기능으로 실현하라. 각 항목에 GitLab의 **정확한 기능명**을 쓰고,
그 기능이 실제로 무엇을 보증하는지(그리고 무엇을 보증하지 않는지) 명시하라.

| 성질 | GitHub이 주던 것 | **GitLab 대응 (코덱스가 구현)** |
|---|---|---|
| ① expected 지문 보관 | job output/secret | **preflight job artifact 또는 dotenv report artifact**로 digest를 넘기되, LLM/제안자가 그 값을 못 바꾸는 stage에 둔다. artifact가 후속 job에만 전달되고 proposal stage에는 노출되지 않도록 `dependencies:`/`needs:`를 최소화 |
| ② 비신뢰 proposal 격리 | 별도 job·권한 | proposal을 **데이터로만** 받는 별도 job. `package_proposal`이 symlink·비 yaml/json·절대·traversal을 차단하는 것을 그대로 쓰되, **proposal job이 checker/expected를 쓸 수 없도록** stage·변수 범위를 분리 |
| ③ 신뢰 checker/product | 매번 fresh checkout | **매 job 새 clone/clean checkout.** GitLab runner 재사용 시 이전 `__pycache__`·잔여물이 남지 않게 `GIT_STRATEGY: clone`(또는 job 시작에 clean). §5 필수와 직결 |
| ④ 사람 intent 게이트 | environment required reviewers | **GitLab `environment:` + protected environment의 "Approvals(배포 승인)" 또는 `when: manual` + protected branch**. 승인자가 `intent_summary`를 본 뒤에만 다음 stage 진행. 승인 없이 자동 진행되지 않게 protected environment 승인 규칙을 건다 |
| ⑤ 판정 근거 | 워크플로 결론 | **`run_fresh_validation`의 종료 코드**로 job 성패 결정. 저장된 `validation-result.json` 바이트를 신뢰하지 않음. exit 2(approval/review_ready)를 파이프라인이 어떻게 취급할지는 §6(주의) |

## 4. GitLab 특유로 반드시 확인/차단할 것

GitLab CI의 알려진 함정을 적대적으로 점검하고 설계에 반영하라.

1. **변수 노출 범위** — `variables:`/CI/CD variables가 proposal job에도 새어 expected를
   읽히지 않는가. protected/masked variable, environment-scoped variable을 정확히 쓴다.
2. **artifact 접근** — preflight의 digest artifact를 proposal job이 `dependencies`로 끌어올
   수 있는가. 끌어오지 못하게 명시적으로 제한한다.
3. **runner 신뢰·재사용** — shared runner에 proposal 데이터가 실행/오염을 일으키지 않는가.
   proposal은 데이터로만 다루고, checker 코드는 신뢰 clone에서만 실행한다. runner 재사용 시
   §5의 `__pycache__` 잔여가 남지 않는지 확인한다.
4. **protected branch/tag·MR 파이프라인** — 비보호 브랜치·fork MR 파이프라인이 사람 승인
   stage를 건너뛰거나, 아무 것도 안 돌고 "성공"으로 표시되지 않는가. 보호 규칙과 rules로 막는다.
5. **승인 우회** — protected environment 승인 없이 `when: manual` 트리거를 아무나 누를 수
   있는가. 승인자·트리거 권한을 분리한다.

## 5. **필수** — .pyc 방어심화 (GitLab에서는 더 이상 선택 아님)

Claude 재검토(P2)에서 남긴 약점: `directory_digest`가 `.pyc`를 제외하므로, checker의
`__pycache__`에 조작된 `.pyc`를 심으면 **지문은 그대로인데 그 코드가 실행**된다(실증됨).
이 약점은 "checker가 매번 신뢰 fresh"라는 전제에서만 도달 불가였다. **GitLab runner는
재사용될 수 있으므로 이 전제가 깨질 수 있다 → 반드시 닫아라.** 택1:

- (a) 권장 — catalog 지문 계산 직전 대상 디렉터리의 `__pycache__` 제거.
- (b) checker 실행을 `PYTHONDONTWRITEBYTECODE=1`로 고정 **하고** 시작 시 기존 `__pycache__`
  제거(생성만 막으면 기존 조작본이 남으므로 제거 병행).
- (c) hash 기반 `.pyc` invalidation(`checked-hash`).

**수용 기준:** benign `.py`와 header가 일치하도록 조작한 `.pyc`를 심어도, 실행 코드가 원본과
다르면 차단/무력화됨을 test로 증명한다. GitLab job에서도 `GIT_STRATEGY: clone` 또는 명시적
clean으로 이전 job 잔여 `__pycache__`가 남지 않음을 함께 보인다.

## 6. 주의 — exit 2(review_ready)의 파이프라인 취급

`plan-validate`는 review_ready를 **exit 2**로 낸다. GitLab job은 0만 성공, 그 외는 실패다.
즉 정상 계획(review_ready)이 job "실패(빨강)"로 뜬다. 이건 Q9(보류) 영역이므로 **임의로
정하지 마라.** 최소한: (i) exit 2를 자동 성공 0으로 뭉개지 않고, (ii) 사람이 `review_state`와
`intent_summary`로 판정하도록 job 로그/artifact에 그 값을 남긴다. exit 2를 "성공"으로
매핑할지 여부는 사용자·조직 결정으로 올린다.

## 7. 손대지 말 것 / 계속 보류

- **변경 금지:** `verdict.py` enum·exit code, 제품 코드, 활성 등록자료, 이미 채택된
  E-hardening·`.pyc` 수정 로직.
- **보류(임의 확정 금지):** Q3·Q6·Q7·Q8·Q9.

## 8. 종료 조건 / 다음

1. `.gitlab-ci.yml`과 (필요 시) 얇은 호출 래퍼, §5 수정, 관련 test·실행 결과를 함께 보고한다.
2. 완료 보고가 오면 Claude가 읽기 전용으로 재검토한다. 재검토 핵심:
   - §3 ①~⑤가 GitLab 기능으로 실제 성립하는가(특히 ① expected가 proposal job 밖,
     ③ 신뢰 clone, ④ protected environment 승인).
   - §4의 GitLab 함정(변수·artifact·runner 재사용·보호 규칙·승인 우회)이 닫혔는가.
   - §5의 `.pyc` 통로가 닫혔는가(조작 `.pyc` 실행 차단).
3. merge·release·배포·운영 완료 표시는 자동 진행 금지. 실제 GitLab 파이프라인 실행·확인은
   사용자 승인 후. 코드 검토 통과를 운영 완료로 표시하지 않는다.

바뀌는 것은 실행 환경(GitHub→GitLab)과 신뢰 경계의 실현 방법이며, 지문·검증·격리의
**코드 로직은 유지**된다.

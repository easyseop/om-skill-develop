# /om-plan GitLab 보호 CI 배선 적대 검토 결과 (2026-08-15)

> 읽기 전용 적대 검토. 원본 branch·file·Git index를 수정하지 않았고 commit·push·merge·
> release·deploy를 하지 않았다. 지정 commit의 tree를 `git archive`로 임시 디렉터리에 추출해
> 읽기 전용으로 검증했고, 공격은 `/tmp` 임시 저장소에만 수행했다. 실제 GitLab 실행/설정이
> 있다고 가정하지 않는다. Q3·Q6·Q7·Q8·Q9는 임의 확정하지 않았다.

## 검증 기준
- branch `codex/om-plan-ci-wiring-20260815`, **구현 commit `ee9f033aea9446e19a0b7420c85a8eb8bd70dc44`**
  ("feat: wire protected om-plan GitLab CI"), 기준 `150a539`.
- 변경 4파일: `.gitlab-ci.yml`(+295), `harness/ci/prepare_trusted_checker.sh`(+47),
  `harness/tests/test_om_plan_gitlab_ci.py`(+201), 운영문서(+212). GitHub 워크플로·neutral
  helper 6개·`paths.py`·`verdict.py`·제품·registration **무변경**(diff로 확인).

# 결론: **채택 (실제 GitLab 설정·pipeline 검증 조건부).**

source 수준에서 5개 신뢰 성질을 GitLab 기능으로 충실히 재구성했고 **P0·P1 없음**이다.
조작 `.pyc` 통로는 이전 검토의 P2였는데 **이번에 닫혔음을 독립 실행으로 확인**했다. 남은 것은
전부 (a) GitLab 외부 설정(승인자·protected runner·branch 보호)과 (b) 사소한 정규식 완화(P2,
비악용)뿐이다.

---

## 1. 필수 공격 결과 — 조작 `.pyc` 통로 (독립 실행)

`prepare_trusted_checker.sh`에 직접 공격:

| 시나리오 | 결과 |
|---|---|
| (A) benign `.py`와 mtime·size 일치하는 악성 `.pyc`(untracked) — **정리 전** import | `PWNED!` 실행됨 |
| (A) `prepare_trusted_checker.sh` 실행 | exit 0, `__pycache__` 제거됨 |
| (A) `PYTHONDONTWRITEBYTECODE=1` 상태 **정리 후** import | `benign`만 실행, `__pycache__` 재생성 안 됨 |
| (B) 악성 `.pyc`를 **commit(tracked)** 후 정리 | **exit 3** ("cache cleanup changed tracked checker files") |

→ 이전 검토의 `.pyc` 이론적 통로가 **닫혔다.** untracked 조작본은 Python 실행 전에 제거되고
캐시 생성이 억제되며, commit된 조작본은 tracked 변경으로 탐지돼 fail-closed(exit 3).

## 2. 질문 1~5 판정 (source 수준)

| # | 질문 | 판정 | 근거(파일·행) |
|---|---|---|---|
| 1 | expected가 trusted preflight에서만 생성·proposal에 미전달 | **성립** | `capture-preflight`가 preflight-result에서 digest 추출→receipt artifact. dotenv report 미사용(`input-lock-output.env`는 `artifacts:reports:dotenv`로 선언 안 됨→변수 전파 없음). proposal job `dependencies: []`→receipt 미다운로드. validate만 `needs: preflight(artifacts:true)` |
| 2 | proposal을 데이터로만, 실행 경로 없음 | **성립** | `package_proposal`(neutral)이 yaml/json만 복사·symlink/절대/traversal 거부(선행 검토 확인). proposal job은 `unset CI_JOB_TOKEN CI_REPOSITORY_URL`, PYTHONPATH=checker/harness(신뢰 코드), 제안 데이터 import 없음 |
| 3 | runner 재사용에도 checker는 CI_COMMIT_SHA fresh clone, 조작 `.pyc` 실행 전 제거 | **성립** | `GIT_STRATEGY: clone`+`before_script` 첫 명령 `prepare_trusted_checker.sh`(HEAD==CI_COMMIT_SHA 확인, `__pycache__`/`.pyc`/`.pyo` 제거, 잔여 확인, tracked 변경 시 exit 3)+`PYTHONDONTWRITEBYTECODE=1`. §1로 실증 |
| 4 | protected default+protected environment면 승인 없이 proposal stage 불가 | **성립(구조)·외부설정 필요** | `.om-plan-protected-job` rule=`web && branch==default && ref_protected==true`. `reject_untrusted_context`가 비보호 web pipeline을 exit 3으로 명시 실패(→all-skipped-green 없음). `intent_review`=`when: manual`+`allow_failure:false`+`environment`, `validate needs intent_review`. **실제 승인은 protected environment Approvers(외부 설정)** |
| 5 | final이 저장 result 아니라 fresh stdout·exit만 신뢰 | **성립** | `run_fresh_validation`이 subprocess stdout+exit+review_state 3중 대조, 저장 `validation-result.json` 미사용. exit 코드가 job 성패 결정. captured-output은 증거용 |

## 3. GitLab 기능별 구분 (§7-2)

| 성질 | source에서 성립 | 외부 설정 필요 | 실제 pipeline 검증 필요 |
|---|---|---|---|
| expected 격리 | ✅ dependencies:[]·dotenv 미사용 | — | 변수/artifact 실노출 확인 |
| proposal 격리 | ✅ neutral packaging·별도 job | protected runner 잠금 | runner 격리 실동작 |
| 신뢰 checker·`.pyc` | ✅ clone+cleanup+DONTWRITE | `om-plan-protected` runner 등록 | runner 재사용 실환경 |
| 사람 intent 승인 | ✅ manual+environment 구조 | **protected environment Approvers·Allowed to deploy·triggerer self-approve 금지(GitLab Premium+)** | 승인 없이 진행 불가 실확인 |
| fresh 판정 | ✅ stdout/exit only | — | exit 2 pipeline 취급(Q9) |
| 보호 진입 | ✅ web-only+reject job | **default branch protection·`.gitlab-ci.yml`/`harness/**` 변경은 승인 MR로만** | MR/push/schedule 미실행 실확인 |

## 4. P0 / P1 / P2

**P0·P1: 없음.** source 수준에서 false pass·격리 우회·expected 유도·조작 `.pyc` 실행을
발견하지 못했다.

**P2-1 (minor, 비악용) — path 입력 정규식이 `..`를 허용한다.**
- `.gitlab-ci.yml` `spec:inputs`의 `request-path`/`proposal-path` 정규식이
  `^[A-Za-z0-9._/-]+$`라 `../`가 통과한다.
- **그러나 traversal은 neutral 함수가 막는다:** `prepare_request`·`package_proposal`이
  `_within(root, source)`로 root 밖 경로를 `PlanCIError`로 거부(선행 검토에서 traversal
  차단 실증). 따라서 **악용 불가**이나, 방어심화로 정규식에서 `\.\.`를 배제하면 belt가 하나 더 는다.

**참고(결함 아님):** validate job이 artifact를 `$CI_PROJECT_DIR/.om-plan-artifacts`(checker 안)로
받았다가 validate 전에 `rm -rf`한다. dirty 스캔은 그 뒤 subprocess에서 도므로 false positive가
없다(순서 의존적이나 현재 올바름). `.om-plan-artifacts`는 catalog 경로(plancore·integrations)
밖이라 지문에도 무영향.

## 5. GitLab CI Lint/pipeline 전에 반드시 할 것 (§7-4)

이건 코드 결함이 아니라 **외부 설정·검증** 항목이다. 하나라도 빠지면 안전장치가 무력화된다.

1. **default branch protection** + `.gitlab-ci.yml`·`harness/ci/`·`harness/acgh/` 변경을
   승인 MR로만 허용(이 파일들이 신뢰 루트다 — 바뀌면 모든 guard가 무력).
2. **protected environment `om-plan-intent-review`**: Allowed to deploy·Approvers·required
   approvals 설정, triggerer self-approve 금지(GitLab Premium/Ultimate 필요).
3. **`om-plan-protected` runner**를 protected runner로 등록하고 타 project 사용 차단.
4. CI/CD variable **`OM_PLAN_INTENT_REVIEW_ENFORCED=protected`**를 protected·masked·
   environment-scoped로 설정(단, 이건 marker일 뿐 실승인은 2번이 담당).
5. **exit 2(review_ready) 취급 결정(Q9)**: 자동 성공 0으로 뭉개지 않기. pipeline 성공/실패
   표시 규칙을 조직이 정한다.
6. GitLab **CI Lint**로 `.gitlab-ci.yml` 검증 후 실제 protected 수동 pipeline 1회 실행해
   §3의 "실제 pipeline 검증 필요" 열을 닫는다.

## 6. 로컬 회귀 재현 (캐시 억제 없이)

- GitLab 전용 `test_om_plan_gitlab_ci.py`: **5 passed**(선언과 일치).
- 계획·boundary·CI·paths·GitLab 집중: **107 passed / 0 fail**(내 추출 사본, `test_om_workflow.py`
  제외 — repo-root import 필요분). 선언된 115는 그 파일 포함분으로 정합.
- `.pyc` 공격 test 포함 전부 **`PYTHONDONTWRITEBYTECODE` 없이** 통과 → `.pyc` 수정+cleanup이
  실환경 flag에 의존하지 않음을 확인.
- 운영문서가 "실제 GitLab 설정·pipeline 미완료", "승인자 목록은 YAML이 못 만듦", "exit 2는
  Q9로 미결정", "`.gitlab-ci.yml` 악성 변경은 branch 보호가 막아야 함"을 명시 — **과대주장 없음.**

## 7. 최종 판정

**채택 — 단, 운영 표시는 §5의 외부 설정·실제 pipeline 검증을 마친 뒤에만.**

- source 수준 배선은 5개 신뢰 성질을 GitLab 기능으로 정확히 재현했고 P0·P1 없음.
- 이전 검토의 `.pyc` 이론적 약점은 닫혔다(독립 실증).
- 남은 것은 코드가 아니라 **GitLab 외부 설정(§5)** 과 **실제 pipeline 1회 검증**이다.
  그 전에는 "CI 운영 완료/최종 PASS/배포 승인"으로 표시하지 않는다.
- P2-1(정규식 `..`)은 비악용, 방어심화로만 권고.

## 다음 사람이 결정할 것 (권고 가능·임의 확정 금지)
1. §5의 GitLab 외부 설정(승인자·runner·branch 보호·변수)과 CI Lint·실pipeline 실행.
2. Q9(exit 2 취급)와 승인자·필요 승인 수 — 조직 결정.
3. P2-1 정규식 강화 여부(비차단).
4. Q3·Q6·Q7·Q8은 계속 보류.

---
*읽기 전용 검토. 원본 수정·commit·push·merge·release·deploy 없음. 검토 commit
`ee9f033aea9446e19a0b7420c85a8eb8bd70dc44`.*

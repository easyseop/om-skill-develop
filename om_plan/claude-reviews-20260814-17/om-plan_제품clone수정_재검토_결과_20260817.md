# /om-plan GitLab 제품 clone blob 확보 수정 재검토 결과 (2026-08-17)

> 읽기 전용 적대 검토. 원본 저장소·코드·문서·Git index를 수정하지 않았고 commit·push·
> 실제 GitLab pipeline 실행을 하지 않았다. 지정 commit의 tree를 `git archive`로 임시
> 디렉터리에 추출해 검증했고, 공격은 `/tmp` 임시 저장소에만 했다. Q3·Q6·Q7·Q8·Q9는 임의
> 확정하지 않았다.

## 검증 기준
- branch `codex/om-plan-gitlab-product-clone-20260817`,
  **구현 commit `ef48a8eee0b3c6b99f2fcd0d4f8f5d70ebc57d3a`**, tree `c34fdae7…`(요청서와 일치).
- 변경(코드 관련): `.gitlab-ci.yml`(제품 clone 2줄), `harness/tests/test_om_plan_gitlab_ci.py`(+95).
  `collectors.py`·`verdict.py`·제품·registration·격리·`.pyc` 로직 **무변경**(diff 확인).

# 결론: **채택.** 수정은 목표(제품 clone blob 확보)를 정확히 달성했고 P0·P1 없음.

단, 이 수정과 **독립적으로 이미 존재하던** 방어 약화 1건(P2)을 실증했다. full-clone
방식이라 이번 파이프라인에서는 발현되지 않지만, 실 runner git 의존이라 기록·후속 권고한다.

---

## 1. 사실 정정
- 없음. 구현자 §3(제품 clone 2곳만 filter 제거, source clone·fail-closed 코드 무변경)은
  diff와 일치한다. 요청서의 회귀 수치(집중 116/전체 536)는 재현 범위에서 정합적이다.

## 2. P0 / P1 / P2

**P0·P1: 없음.** 제품 clone에서 `--filter=blob:none`을 제거해 full clone이 되므로, pinned
base·target의 blob이 지연 fetch 없이 로컬에 존재한다. request/proposal clone과 fail-closed
코드는 그대로다.

**P2 (실증·실 runner 의존·이 수정과 무관하게 기존) — `SOURCE_BLOBS_UNAVAILABLE` fail-closed가
`GIT_NO_LAZY_FETCH`를 무시하는 git에서 실효가 없다.**
- 근거: `collectors.py`의 fail-closed는 `environment["GIT_NO_LAZY_FETCH"]="1"` + `git cat-file
  --batch-check`에 의존한다(변경 없음). 그런데 **Apple Git 2.39.3은 GIT_NO_LAZY_FETCH를
  존중하지 않는다** — 추가된 test 본문 주석도 이를 인정하며, blobless-실패 test는 promisor
  설정을 `--unset`해 우회한다.
- 실증(내 PoC, promisor를 살려둔 채 — test와 달리):
  ```
  promisor 설정: true
  _materialized_objects(blobless clone) → 성공(레코드 2개)
  = GIT_NO_LAZY_FETCH 무시하고 LAZY-FETCH 됨 = fail-closed 깨짐
  ```
  같은 상황에서 promisor를 제거하면(=test 방식) `SOURCE_BLOBS_UNAVAILABLE`로 실패한다.
  즉 fail-closed의 실효는 **promisor 부재**에 달렸지 `GIT_NO_LAZY_FETCH`에 달린 게 아니다.
- **이 수정에 대한 영향:** full clone은 promisor 없이 모든 blob을 확보하므로 이 경로가
  **발현되지 않는다**(missing blob 자체가 없어 fetch 시도가 없다). 따라서 이번 파이프라인은
  정상 동작하고 false pass도 없다.
- **잔여 위험:** 만약 이후 누군가 제품 clone에 filter를 재도입하거나, GitLab 제품 프로젝트가
  자체적으로 부분 미러이거나, source가 아닌 곳에서 promisor clone이 재발하면, `GIT_NO_LAZY_FETCH`를
  무시하는 runner에서 collector가 **조용히 lazy-fetch**해 "실제로 있던 blob만 인증한다"는
  무결성 의도가 무너진다. **실 GitLab runner의 git 버전에서 이 방어의 실효 여부는 미검증.**
- 권고(비차단): (a) 제품 clone이 full(=promisor 없음)임을 collect 전에 단언
  (`remote.origin.promisor` 부재 확인) — full-clone 전제를 코드로 못박음, 또는 (b) 실 runner
  git이 `GIT_NO_LAZY_FETCH`를 존중하는지 검증. (a)가 버전 무관하게 견고하다.

**P2 (성능/부작용, 요청서가 이미 인정) — 제품 full clone이 run당 2회 발생.**
- preflight·validate 각각 제품 full clone(로컬 실측 1.13.1 전체 ~2.7GB). run당 2회
  대용량 clone + 기본 checkout(파일 1만4천 개). 정확성·보안 영향은 없으나 job 시간·디스크 큼.
- 권고(비차단): 필요 시 `--no-checkout` 추가(작업트리 미사용) 또는 pinned ref 범위만 확보하는
  최적화. 이번 정확성 판정과 무관.

## 3. 발견별 파일·행·반례
- P2-1: `.gitlab-ci.yml` 128·257행(수정 후 full clone) + `collectors.py:117`(`GIT_NO_LAZY_FETCH`)
  ·`:149`(`SOURCE_BLOBS_UNAVAILABLE`). 반례: 위 PoC(promisor 유지 blobless → 성공/lazy-fetch).
  test `test_blobless_..._fails_closed_...`는 promisor를 unset해 이 경로를 우회.
- P2-2: `.gitlab-ci.yml` 128·257행(제품 full clone, `--no-checkout` 없음).

## 4. 수용 기준 1~5 판정

| # | 질문 | 판정 | 근거 |
|---|---|---|---|
| 1 | 제품 clone 직후 pinned base·target blob이 지연 fetch 없이 로컬에 존재 | **성립** | full clone은 도달 가능한 모든 blob을 받음. full-clone 성공 test + 내 확인 |
| 2 | GIT_NO_LAZY_FETCH·SOURCE_BLOBS_UNAVAILABLE fail-closed가 완화되지 않음 | **이 변경으로는 완화 안 됨(코드 무변경)**, 단 **원래부터 Apple Git 2.39에서 실효 없음**(P2-1) | collectors.py diff 없음. 그러나 PoC로 실효 부재 실증 → 실 runner 미검증 |
| 3 | request·proposal clone이 정확히 기존 부분 clone으로 남음 | **성립** | 118·203행 `--filter=blob:none --no-checkout` 그대로(diff 없음) |
| 4 | 정적 test가 잘못된 문자열 비교로 통과시키는 반례 | **없음(경미)** | `test_gitlab_product_clones_...`가 제품 라인의 filter 부재 + source 라인의 filter 존재를 정확 문자열로 단언. 주석 아닌 실제 명령이라 오통과 아님 |
| 5 | blobless 실패/full 성공 test가 실제 collector 경계를 증명 vs fixture-실 GitLab 차이 | **부분** | full-clone 성공 경로는 실증(정확). **blobless-실패 경로는 promisor를 unset해 우회**하므로 `GIT_NO_LAZY_FETCH` 기제 자체는 증명하지 않음 = fixture와 실 runner 차이 존재(P2-1). 단 full-clone 수정이 그 경로를 회피하므로 **P0/P1 아님** |

## 5. 빠진 회귀 test
- **`GIT_NO_LAZY_FETCH` 실효 test 부재:** promisor를 유지한 채 fail-closed가 실제로 나는지
  검증하는 test가 없다. 현재 test는 promisor를 unset해 우회 → 실 runner에서의 fail-closed를
  증명하지 못한다. (실 runner git 의존이라 CI에서만 최종 확인 가능.)
- **제품 clone이 full(promisor 없음)임을 단언하는 test 부재:** 향후 filter 재도입 회귀를
  잡으려면 "제품 clone 결과에 promisor remote 없음"을 확인하는 test가 권장(P2-1 권고 (a)와 짝).

## 6. 미검증 (실 GitLab 미실행)
- 실 GitLab runner의 git 버전과 그 버전의 `GIT_NO_LAZY_FETCH` 존중 여부 → **미검증.**
- 실제 pipeline에서 제품 full clone 시간·디스크 비용, job token 접근 → **미검증.**
- CI Lint·실 pipeline은 사용자 승인 전이라 미실행. 운영 통과로 추정하지 않음.

## 7. 최종 판정

**채택.** 이 수정은 blob 미확보로 파이프라인이 `SOURCE_BLOBS_UNAVAILABLE`로 멈추던 문제를
full clone으로 정확히 해소했고, source clone·fail-closed 코드·격리·`.pyc`·expected digest·
exit 2 의미를 바꾸지 않았다(수용 기준 1·3·4 성립, 6 무변경). P0·P1 없음.

- **후속(비차단):** P2-1(제품 clone이 promisor 없음을 단언 / 실 runner의 GIT_NO_LAZY_FETCH
  검증)과 P2-2(full clone 2회 비용 최적화). 둘 다 이번 채택을 막지 않는다.
- 실 GitLab 실행 전 확인 목록은 이전 GitLab 검토 결과(§5 외부 설정)와 함께 처리한다.

## 다음 사람이 결정할 것 (권고 가능·임의 확정 금지)
1. P2-1 방어심화 반영 여부(promisor 부재 단언 권장).
2. P2-2 clone 비용 최적화 여부.
3. 실 GitLab runner git 버전 확인.
4. Q3·Q6·Q7·Q8·Q9 계속 보류.

---
*읽기 전용 재검토. 원본 수정·commit·push·pipeline 실행 없음. 검토 commit
`ef48a8eee0b3c6b99f2fcd0d4f8f5d70ebc57d3a`, tree `c34fdae790602c139cb6868b34b1d6ab511d8797`.*

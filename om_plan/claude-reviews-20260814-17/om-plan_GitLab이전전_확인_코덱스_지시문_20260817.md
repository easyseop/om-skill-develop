# Codex 확인 요청 — /om-plan GitLab 이전 전 점검 (GitHub 미사용, 1.13.1만)

> 작성: 2026-08-17. 이 문서를 코덱스에게 그대로 전달한다.
> 성격: **읽기 전용 확인.** 아직 이전·push·삭제·fetch를 실행하지 말고, 아래를 확인해
> 사실/차이를 보고하라. 실제 이전은 확인이 끝난 뒤 별도 승인으로 한다.

## 0. 배경 (확정된 결정)

- GitHub / GitHub Actions를 **더는 쓰지 않는다.** CI는 **GitLab**로 간다.
- 지금 목표는 **1.13.1만** GitLab에 올려 `/om-plan`을 돌릴 수 있게 하는 것이다.
  1.13.2(upgrade)는 이번 범위가 **아니다.**
- `/om-plan` 코드 저장소 경계는 **D안 확정**: 별도 저장소가 아니라 `openmetadata-test`
  안의 `harness/acgh/plancore`(제품 무관) + `harness/acgh/integrations/om`(OM 전용).
  근거: `REVIEW_구현저장소_경계_결과_20260814.md` §5(B안=별도 저장소는 기각).

## 1. Claude가 사용자 로컬에서 확인한 것 (코덱스가 재확인)

Claude가 사용자 머신에서 read-only로 확인한 사실이다. 코덱스는 자기 환경(전체 클론/원격)
에서 같은 것을 확인해 **일치/차이**를 보고하라.

1. **1.13.1 지정 커밋이 있는 로컬 제품 저장소는 `~/om-work/OpenMetadata`(easyseop/OpenMetadata)
   하나뿐.** `~/om-work/OM_TEMP`·`~/om-work/om-temp-real-1.13.1`에는 그 커밋이 없다(다른 계보,
   `custom/om-1.13.1` tip이 `59dae915`로 다름).
2. **1.13.1이 필요로 하는 두 커밋:** 공식 `afcb2d2cd7e7c28f1d0ce60538c60a96f4eb9dc9`,
   커스텀 `8ac18ad053d9274774e274ba17b35911ac0b9dcb`.
3. **치명적:** `~/om-work/OpenMetadata`는 **부분 클론(promisor, blob:none)**이라 **파일 내용이
   로컬에 없다.** `GIT_NO_LAZY_FETCH=1`로 확인 시 custom `8ac18ad0` 트리 **14,031개 blob 전부
   없음**, official `afcb2d2c`는 `ls-tree -r`가 0건(트리도 미materialize). 즉 목록만 있고
   알맹이는 GitHub에만 있다. **GitHub을 끊으면 못 받는다.**

## 2. 코덱스가 확인·답할 것

각 항목에 "확인됨 / 차이 있음 / 알 수 없음"과 근거(명령·경로·SHA)를 달아라.

### 2.1 저장소 경계
- `/om-plan` 코드가 실제로 `openmetadata-test/harness/acgh/plancore` + `integrations/om`에만
  있고, 별도 저장소 의존이 없는가? (D안대로인지)

### 2.2 GitLab 이전 범위 (1.13.1 한정)
- GitLab에 올려야 하는 저장소가 **둘**인지 확인: (a) 검사기 `openmetadata-test`,
  (b) 제품 `OpenMetadata`(1.13.1 커밋 포함). 그 외에 `/om-plan` 1.13.1이 의존하는 저장소가
  더 있는가?
- 제품 저장소에서 **1.13.1에 필요한 ref**가 정확히 무엇인가? (최소: `afcb2d2c`, `8ac18ad0`를
  가리키는 브랜치. run-request가 `refs.official`/`refs.current_custom`로 쓰는 그 값.)
- **1.13.2 브랜치·객체는 이번에 빼도 되는가?** (`official/om-1.13.2`,
  `codex/om-1.13.2-rehearsal-vendor-merge-*` 등) — 빼도 1.13.1 `/om-plan`에 지장 없는지 확인.

### 2.3 콘텐츠 완전성 (가장 중요)
- `/om-plan`은 blob을 **lazy-fetch 없이**(collectors의 `GIT_NO_LAZY_FETCH=1`,
  `SOURCE_BLOBS_UNAVAILABLE`로 fail-closed) 읽는다. 따라서 GitLab 제품 프로젝트에는
  **1.13.1 두 커밋의 모든 tree·blob이 물리적으로 존재**해야 한다.
- 코덱스 환경 어딘가에 **1.13.1 전체 객체를 가진 완전 클론**이 있는가? 없다면, GitHub이
  살아 있는 지금 **어떤 명령으로 materialize**해야 하는가(예: `git fetch` unshallow/
  full-blob refetch, 또는 `git clone` without filter)? 정확한 명령을 제시하라.
- 검사기 `openmetadata-test` 쪽도 확인: `/om-plan` 1.13.1이 읽는 **등록자료·계약·정책**
  (`harness/registrations/om-temp-1.13.1/`, `contracts.yaml`, `policies/` 등)과 GitLab
  파이프라인 파일(`.gitlab-ci.yml`, `harness/ci/`, `harness/acgh/plancore`·`integrations/om`)이
  전부 그 저장소에 커밋돼 있어(빈 부분/부분 클론 아님) GitLab에 통째로 올라가는가?

### 2.4 GitLab 파이프라인 요구와의 정합
- `.gitlab-ci.yml`(commit `ee9f033`)은 제품을 `CI_SERVER_FQDN/${OM_PLAN_PRODUCT_PROJECT}`
  에서 `CI_JOB_TOKEN`으로 clone한다. 즉 제품은 **같은 GitLab 인스턴스의 프로젝트**여야 하고,
  pinned SHA가 그 프로젝트에 **도달 가능 + blob 존재**해야 한다. 이전 계획이 이 요구를
  만족하는가(제품이 GitLab에 있고, 1.13.1 커밋이 full 객체로 있는가)?

### 2.5 순서·위험
- "GitHub 살아 있을 때 1.13.1 객체 materialize → 그다음 GitLab 이전"이라는 순서가 맞는가?
  GitHub을 먼저 끊으면 복구 불가한 지점이 어디인가? materialize를 놓치면 어떤 증상으로
  드러나는가(`SOURCE_BLOBS_UNAVAILABLE`)?

## 3. 하지 말 것 (이번 확인 단계)

- 이전·push·merge·삭제·강제 fetch를 **아직 실행하지 마라.** 확인·명령 제시까지만.
- 1.13.2를 이번 범위로 끌어들이지 마라(필요 여부만 판정).
- Q3·Q6·Q7·Q8·Q9는 임의 확정하지 마라.
- 부분 클론에 객체가 없는 것을 "있다"고 추정하지 마라. 실제 `GIT_NO_LAZY_FETCH=1`로 확인.

## 4. 원하는 출력

1. §2.1~§2.5 각 항목의 확인 결과(확인됨/차이/모름 + 근거).
2. **1.13.1 콘텐츠를 GitHub이 살아 있는 동안 완전 materialize하는 정확한 명령**(제품·검사기 각각).
3. GitLab에 올릴 최종 목록: 저장소·브랜치·커밋(1.13.2 제외 여부 명시).
4. 이전 순서와, materialize를 놓쳤을 때의 실패 증상.
5. Claude가 확인한 §1과 **다른 점이 있으면** 정확히 지적.

이 확인이 끝나면 사용자가 (a) materialize 실행, (b) GitLab 이전을 별도로 승인한다.

# Claude 세션 인계 — OpenMetadata 슬래시 명령 개발

> 다른 컴퓨터·새 세션의 Claude는 이 파일부터 읽는다. 로컬 메모리(`~/.claude`)는 이 컴퓨터에만 있으므로 여기가 인계의 정본이다.

## 너(Claude)의 역할

이 프로젝트에서 Claude는 **구현자가 아니라 검증자·기록자·번역자**다. 구현은 코덱스(별도 에이전트)가 한다.

1. **적대적 검토자** — 코덱스의 설계·구현을 호의적으로 보완 해석하지 않고, 실제 파일·코드·import를 열어 우회·누락·잘못된 통과를 찾는다. 판정에는 반드시 파일·줄 근거를 단다. 일반론으로 결론내지 않는다.
2. **히스토리 기록자** — 세부 문제마다 "문제→논의 경과→결정→고려한 것→반영 위치→미결" 형식으로 이 저장소에 남긴다 (README.md의 규약).
3. **두 갈래 보고자** — 사용자에게는 비개발자 눈높이 보고(결론 먼저·정확한 비유·"즉, ~입니다" 의역), 기술 상세는 코덱스가 읽을 파일로 분리한다. 사용자가 용어를 하나하나 되묻기 시작하면 보고가 실패한 것 — 다시 풀고, 구체화된 내용은 코덱스용 파일에 소급 반영한다. (스킬: `two-track-report`)
4. **깃 대행** — 사용자의 요청으로 커밋·푸시를 대행한다. 단 **코덱스 전용 저장소에는 커밋하지 않는다**(아래 저장소 지도).

## 저장소 지도 (누가 소유하나)

| 저장소 | 위치 | 소유 | Claude가 커밋? |
|---|---|---|---|
| **om-skill-develop** (이 저장소) | `sites-plugin-sites-openai-bundled/skill_develop/` → github.com/easyseop/om-skill-develop | 사용자·Claude | ✅ 히스토리·검토 결과 |
| openmetadata-test | `…/work/review-openmetadata-test` → github.com/easyseop/openmetadata-test | 코덱스(작업) + 사용자(원격) | ⚠️ 코덱스가 준비한 브랜치의 push 대행만 (직접 stage 금지) |
| 코덱스 워크스페이스 | `sites-plugin-sites-openai-bundled/` 자체 (원격 git.chatgpt-team.site) | 코덱스 전용 | ❌ 절대 커밋 금지. 파일 읽기·검토 결과 파일 작성만 |
| openmetadata-lab-docs | `~/openmetadata-lab` → github.com/easyseop/openmetadata-lab-docs | 사용자·Claude | ✅ 발표덱·핸드오프 |
| claude-skill-by-seop | `~/claude-skill-by-seop` → github.com/easyseop/claude-skill-by-seop | 사용자·Claude | ✅ 스킬 배포용 |

## 코덱스와의 협업 프로토콜

- 코덱스에게는 **파일 경로 + 지시문**으로 전달한다. 지시문에는 금지사항(구현 금지/수정 금지/push 금지 등)과 결과 파일명을 명시한다.
- 검토 요청이 오면: 지시문을 처음부터 끝까지 읽고 그대로 따른다 → 전체 파일 목록(숨김 포함) 확인 → 읽지 못한 파일은 결과 첫머리에 명시(그 상태로 "전체 검토 완료" 금지) → 지정된 결과 파일 하나만 작성 → 중단.
- 사람 결정 항목(Q1~Q9 류)은 Claude도 코덱스도 임의 확정하지 않는다. 선택지·권고안으로 사용자에게 올린다.
- "테스트 N개 전부 green"은 신뢰 근거가 아니다. 알려진 mutant/반례가 실제로 실패하는지가 기준.

## 현재 상태 (2026-08-21 기준) — 다른 컴퓨터/새 세션은 여기부터

### 전체 그림
4단계 파이프라인(계획 `/om-plan` → 반영 `/om-apply` → 검증 `/om-verify` → 요약 `/om-report`)을 만든다. 기능을 **커스텀 ID로 등록·관리**하고 그 기준으로 자동 운영하는 게 목표. **공유용 요약은 `skill_develop/공유정리/`**(주간보고 메인 + om-plan 하위 + README)에 있고 om-skill-develop(GitHub)에 push됨.

### 역할 (재확인)
- **Claude = 검증자·기록자·번역자·git대행(구현 아님).** 코덱스 트리엔 커밋 금지, om-skill-develop 등 사용자 저장소만 커밋.
- **Codex = 구현자.**

### 참고 저장소 (위 "저장소 지도" 참조)
- **om-skill-develop**(GitHub, `skill_develop/`) = **원천**. 설계·검토·결정 문서 + 스킬 원본. Claude가 커밋. **PUBLIC 이므로 내부 GitLab 주소 금지.**
- **검사기 clean-export**(`work/kb-datacatalog-upgrade-checker-om-plan-cli`, remote gitlab-checker) = 배포용 "도구". Codex 소유. `plan` 계열만 있음(apply/watch 등은 트림됨).
- **완전본 트리**(`work/review-openmetadata-test-om-plan`) = 예전 넓은 하네스(apply·bootstrap·watch·risk 등 ~20명령, om-plan보다 먼저 2026-07-30 생성). apply 코드가 여기 있음.

### 어디까지 했나
- **/om-plan(계획, 1단계): 기능 완성·Claude 검증 완료.** 4모드(initial/feature/change/upgrade) + 품질 게이트 전부:
  - 커스텀별 관계 fact 보존 / direct_tests 제거(계약only) / 수정 시 테스트 누락 차단 / 계획서 타입검사 / 경로수 기준 정정 / 업그레이드 3층 검증.
  - 검증완료 3커밋으로 분리 완료(clean-export 브랜치 `codex/om-plan-verified-gates-20260820`, HEAD `7c544efb2b`, 전체 210 테스트 green). **push 안 함**(GitLab 보류).
- **/om-apply(반영, 2단계): v1 완성 — 실데이터 재리허설 통과(2026-08-24).** 리허설이 찾은 P0(전역경로 검사 결함) 수정 + 활성 공유소유맵 정정(사람 승인) 후, 실제 BANK-OM-005 change가 **`static_consistent_awaiting_verify` + eligible:true 도달**, 우회 반례는 여전히 차단, 239 테스트 green(Claude 재검증 69). 여전히 clean-export **미커밋 12파일**. 계약 테스트는 미실행(=om-verify 몫). clean-export 브랜치 위 **미커밋** 작업트리(applycore/ + integrations/om/apply.py + skills/om-apply + 반례 24개, 전체 235 테스트 green). 성공 상태는 `static_consistent_awaiting_verify` 뿐(계약충족은 claim만). 결과서 `om_plan/63_...`(Codex가 작성 시)·검증 `om_plan/64_...`. **다음 = 실데이터(1.13.1 change/1.13.2 upgrade) 예행연습 → 통과 후 commit·push 사람 결정.**
  - (설계 경위) 설계+적대검토+사람결정:
  - 설계서 `om_plan/58_...`, Codex 검토 `om_plan/60_...`(P0 3건 등), 공유용 논의정리 `공유정리/하위_om-apply_논의정리_20260821.md`.
  - **확정 결정**: 방식 A(LLM이 코드 작성) / 관리파일 **3방향 대조**(계획·실제diff·관리파일 — 코드=매니페스트 2개만 보면 자기세탁 가능) / 계획범위 우선 개발, 이탈 시 **민감=STOP·그외=사유+사람승인**(v1) / **독립검토 sub-agent는 우선순위0 향후개선** / 병합실패 **3단복구**(LLM고치기→재계획→사람) / 실행검사 **컴파일·생성기 포함** / 예전 apply **부분 재활용만**(안전 primitive) / **clean-export 편입** / 계약충족은 verify 몫(claim만).
  - **검토 중**: 계획 밖 '안전 유형 자동통과' 도입 여부.
- **/om-verify(검증, 3단계): v1 완성 — 실데이터 리허설 통과 + Claude 재검증 통과(2026-08-24).** 실제 BANK-OM-005 후보를 이미지로 빌드(build receipt local-issued)→fresh compose(`omv005-*`, :18585)로 기동→브라우저 계약 테스트(`test_hangul_composition_roundtrip`) 실서버 pass 1/skip 0→`verified(0)` receipt(digest 재계산 일치). probe 4종(딴 서버/등록자료 변조/전부 skip/run-dir 재사용) 전부 정확한 코드로 차단 — run-dir 재사용은 Claude가 직접 재현. 상주 lab·활성 등록자료·검사기 HEAD `7c544efb2b` 불변, fresh 환경 정리 완료. **파이프라인 ①plan→②apply→③verify 실물 연결 확인.** clean-export 미커밋(verifycore/ + om/verify.py + 반례 19, 전체 259 green). 설계 `73`·적대검토 `75`→정본 `76`→구현 77·검증 78→리허설 지시 79·결과 80·**재검증 81**. 남은위험 6건은 81 §참조(공식 Dockerfile 재현성·local-issued 한계·test-agent meta.run_id 등). **test-agent 채택 확정**(파일럿 72; 이번 리허설에선 meta.run_id null로 UI 부품 미결속 — not_configured). **다음 = om-verify 커밋분리·push 여부 사람 결정.**
- **/om-report: 미착수.**

### 지금 단계 (다음 액션)
1. **커밋분리 완료(2026-08-24, 결과서 82)** — clean-export 브랜치에 3커밋 추가(등록자료 P0-3 정정 `411c7cfe1f` 210 green → om-apply v1 `093c5ac8b1` 239 green → om-verify v1 `82be68e733` 259 green). 누적 diff=기준선·교차오염 0 검증됨. 작업트리 clean.
2. **GitLab push·MR 완료(2026-08-24)** — 사용자 단기 PAT로 Claude가 push 대행, 브랜치 6커밋(om-plan 3 + 이번 3) 신규 등록, **MR !2 open(target=main, 충돌 없음)**. **파이프라인은 MR에서 안 도는 게 설계**(workflow rules: web 수동 실행 + 보호된 main 한정, 82 §3). 다음 = **사람이 MR !2 검토·merge** → (원하면) 웹 UI에서 파이프라인 수동 실행. **주의: 파이프라인은 om-plan 전용이라 apply/verify 259 테스트는 CI에서 안 돎(82 §4) — CI 확장 여부 후속 결정.** 사용 후 PAT는 사용자가 revoke(1주 자동만료).
3. **om-report(4단계) 설계 착수 여부 — 사람 결정.** (설계→적대검토→구현 순서 유지.)
4. 기록된 후속: P0-2 활성잠금(R-1·R-2와 함께)·P1-2~5·upgrade 독립 문서검토 배선(리허설 66·68)·om-verify 남은위험 6건(81).

### 보류·열린 것
- **GitLab push + 파이프라인 초록불**: Q9(exit2→CI성공, 이미 로컬 구현 `2194c414ab`)를 GitLab 반영 후 4모드 파이프라인 통과해야 배포. 지금 보류.
- **사람 결정 대기**: Q10 재베이스 단계(A~D, D 권고), Q21 등록밖 게이트 배선, 8-2 om-apply 패키징 위치.
- **★ 예전 apply 재검토 필요(사용자 지시)**: `work/review-openmetadata-test-om-plan/harness/acgh/registration_prep.py`는 **다른 용도로 만든 구버전(2026-07-30)**이라 **현재 관리파일 구조·정책(트랙A 게이트·direct_tests 정리·매니페스트 구조 등)과 맞는지 불명.** om-apply가 재활용하려면 **그 전에 이 코드가 지금 정책에 부합하는지 별도 재검토** 필요. 부합 안 하면 재활용 대신 A형 전용 신규 로직. (Codex 적대검토 59 항목6에서 착수하되, 그 이상으로 "현재 정책 부합 여부"를 별도로 봐야 함.)
- **확정된 임시결정·재검토 대상**: `om_plan/24_...`의 "★ 재검토 대상(R-1~R-5)" 표가 정본. (기준선 잠금 릴리즈-only 등)

### 핵심 문서 포인터
- **결정기록(정본)**: `om_plan/24_누락감사_사람결정과_기록_20260820.md`
- **공유 요약**: `공유정리/` (README부터)
- **om-apply 설계**: `om_plan/58_...` / **커밋분리 결과**: `om_plan/57_...` / **재베이스·게이트 조사**: `om_plan/55_...`
- (구 08-14 인수인계·경위는 `om_plan/07·08`, `00~06` 참고 — 역사)

## 환경 주의사항 (이 컴퓨터 한정 이슈 포함)

- 이 Mac의 셸은 `grep`/`rg`/`find` 함수 래퍼가 깨져 있을 수 있음("claude native binary not installed") → `command grep`, `command find`로 우회.
- openmetadata-test의 fetch refspec이 브랜치별 제한이라, 새 브랜치 push 후 `@{upstream}` 조회가 실패하면 해당 브랜치의 refspec 한 줄을 추가해야 한다 (기존 관례 따름).
- 관련 로컬 랩: `~/openmetadata-lab` (OM 1.13.2 서버가 Colima 위에 상주 — 계약 테스트 실환경).
- `/private/tmp/om-plan-rehearsal.*` 아래 예행연습 결과는 임시 증거다. 다른 컴퓨터에는 전달되지 않으므로 원격 코드·테스트·인수인계 문서를 정본으로 사용한다.

## 스킬 (claude-skill-by-seop에서 설치)

```bash
git clone https://github.com/easyseop/claude-skill-by-seop.git
cp -r claude-skill-by-seop/skills/* ~/.claude/skills/
```

- `two-track-report` — 위 역할 3의 규약
- `plain-report` — 기술용어 + "즉" 의역 병기 보고
- `reader-doubt-check` — 문장을 내놓기 전 독자 의문 7종 자기검토 (비유 남발 금지·확인 없는 단정 금지 등)

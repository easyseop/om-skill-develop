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

## 현재 상태 (2026-08-14 마지막 세션 기준 — 이후는 git log와 om_plan/06 참조)

- `/om-plan` 설계: 1차 적대적 검토 → 코덱스 수정 → **2차 재검토 통과("구현 승인 검토 가능")**
- 저장소 경계: **D안 확정** — openmetadata-test 안에 `acgh/plancore`(제품 무관) + `acgh/integrations/om` 경계, 경계 lint test, 도입 시 clean export
- 사람 결정: **Q1=C, Q2=A, Q4=B, Q5=B 확정**, Q3·Q6·Q7·Q8·Q9 보류
- 코덱스에게 구현 착수 프롬프트 전달됨 → **다음 단계: 코덱스의 구현 보고가 오면 3차 검토**(반례가 진짜 잡는지 / plancore 경계 준수 / 1.13.1 initial·change + 1.13.2 upgrade 예행연습 결과) 후 push 승인
- 상세 경위: `om_plan/00~06`, 경계 검토: `REVIEW_구현저장소_경계_결과_20260814.md`
- 설계 패키지·1/2차 검토 결과 원본: `../deliverables/claude-om-command-review-20260814/`

## 환경 주의사항 (이 컴퓨터 한정 이슈 포함)

- 이 Mac의 셸은 `grep`/`rg`/`find` 함수 래퍼가 깨져 있을 수 있음("claude native binary not installed") → `command grep`, `command find`로 우회.
- openmetadata-test의 fetch refspec이 브랜치별 제한이라, 새 브랜치 push 후 `@{upstream}` 조회가 실패하면 해당 브랜치의 refspec 한 줄을 추가해야 한다 (기존 관례 따름).
- 관련 로컬 랩: `~/openmetadata-lab` (OM 1.13.2 서버가 Colima 위에 상주 — 계약 테스트 실환경).

## 스킬 (claude-skill-by-seop에서 설치)

```bash
git clone https://github.com/easyseop/claude-skill-by-seop.git
cp -r claude-skill-by-seop/skills/* ~/.claude/skills/
```

- `two-track-report` — 위 역할 3의 규약
- `plain-report` — 기술용어 + "즉" 의역 병기 보고
- `reader-doubt-check` — 문장을 내놓기 전 독자 의문 7종 자기검토 (비유 남발 금지·확인 없는 단정 금지 등)

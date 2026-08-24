# om-apply·om-verify 커밋분리 결과 + GitLab 반영 준비 상태

작성일: 2026-08-24. 작성자: Claude(git 대행). 사람 결정(24 확정 항목): verify까지 완료된 구현을 커밋분리하고 GitLab에 반영.

## 1. 커밋 3개 (clean-export `codex/om-plan-verified-gates-20260820` 위)

| 커밋 | 내용 | 시점 테스트 |
|---|---|---|
| `411c7cfe1f` | **등록자료 P0-3 정정** — shared-path-owners에 serviceConnection.ts·DatabaseServiceUtils.test.tsx 추가(BANK-OM-006/007), 39=39. 사람 승인분(67-69) | 210 passed, exit 0 |
| `093c5ac8b1` | **om-apply v1** — applycore/ + om/apply.py + apply CLI + SKILL.md + 반례. settings.json apply-deny 해제, CLAUDE/README 갱신 | 239 passed, exit 0 |
| `82be68e733` | **om-verify v1** — verifycore/ + om/verify.py + verify CLI + 반례 19(OV-01~14) | 259 passed, exit 0 |

테스트 수가 각 단계 문서의 기록(om-plan 210 → apply 239 → verify 259)과 정확히 일치.

## 2. 분리 검증 (om-plan 커밋분리와 동일 기준)

- **누적 diff = 기준선**: 분리 전 `git diff` 저장본과 3커밋 누적 diff가 동일(diff 대조, index 줄 제외).
- **교차오염 0**: 커밋2 트리에 verifycore/·om/verify.py·verify 반례 부재, om_workflow.py에 `verify_action` 0회(커밋3에서 3회). 혼합 3파일(om_workflow.py·om/__init__.py·test_plan_cli.py)은 apply-only 중간본으로 hunk 분리.
- **각 커밋 시점 전체 테스트 green**: 위 표 — 격리 worktree에서 실행.
- 작업트리 잔여 변경 0(분리 후 clean).

## 3. GitLab 반영 — 완료 (2026-08-24, 사용자 PAT로 Claude 대행)

- 서버 회복 확인(401 응답) → 사용자가 단기 PAT(write_repository+read_api, 1주 만료) 발급 → Claude가 push 대행.
- **push 완료**: 브랜치 `codex/om-plan-verified-gates-20260820` 6커밋(om-plan 3 + 이번 3) 신규 등록. 토큰은 파일·저장소에 기록하지 않음.
- **MR !2 생성**(push option): target=main, conflicts 없음(원격 main은 우리 조상 69dd83cf + 병합커밋 1개, tree 동일 확인). merge는 사람 몫.
- **파이프라인 미생성 = 설계된 동작**: workflow rules가 `$CI_PIPELINE_SOURCE == "web"`만 허용(그 외 never), 잡 rules는 추가로 보호된 기본 브랜치 한정. 즉 신뢰 재실행 파이프라인은 **merge 후 사람이 웹 UI에서 spec inputs(request-ref·proposal-ref·product-project 등)를 지정해 수동 실행**하는 구조. MR 단계 CI 게이트는 원래 없음(lint API로 확인: "workflow:rules로 인해 미실행").

## 4. 갭 (반영 전 알아야 할 것)

- **파이프라인은 om-plan 전용**(.gitlab-ci.yml: guard→preflight→intent_review→proposal→validate, om_plan_ci.py 호출). **apply/verify의 259 테스트는 CI에서 돌지 않는다** — 초록불이 apply/verify 품질을 보증하지 않음. 후속 결정 항목: CI에 pytest 잡 추가 여부.
- 남은위험 6건(81)은 push를 막는 사유 아님(기록 관리).

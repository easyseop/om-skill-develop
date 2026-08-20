# /om-plan 스킬 — 개발 산출물

OpenMetadata 커스터마이징 등록 계획을 만드는 Claude Code 슬래시 명령 `/om-plan`. initial·feature·change·upgrade 4모드.

## 폴더 구조

```
20_skill_draft/
├── README.md                     # 이 파일
├── AUTHORING_REVIEW.md           # 작성 근거·검증 기록 (개발용, GitLab에 넣지 않음)
└── .claude/skills/om-plan/       # ★ 런타임 스킬 = GitLab에 이 폴더째 복사
    ├── SKILL.md                  # 커맨드 본체 (절차·완료조건·보고형식)
    └── references/               # 스킬이 실행 중 읽는 참조
        ├── request-format.md         # 입력(요청 YAML) 형식
        ├── proposal-format.md        # 출력(제안서) 형식·검사기 강제 규칙
        ├── initial.md / feature.md / change.md / upgrade.md   # 모드별 지침
        └── example-<mode>-proposal.md                          # 모드별 통과 완성 예시
```

## GitLab(실사용 검사기 저장소) 설치

`.claude/skills/om-plan/` 폴더를 검사기 저장소 루트에 **그대로 복사**한다. `AUTHORING_REVIEW.md`와 이 `README.md`는 런타임에 필요 없으므로 넣지 않는다("완성본만" 원칙).

설치 후 남은 배선(Codex 몫): 세션/런 marker Hook, `UserPromptExpansion`, apply 차단 permissions, Python 3.11+ 실행 래퍼. 상세는 `AUTHORING_REVIEW.md` §7.

## 검증 상태 (2026-08-20)

4모드 모두 clean-room(대화 맥락 없는 별도 에이전트)에서 작성 → `plan check` **approval** 통과. 근거·회고는 `AUTHORING_REVIEW.md` §9~13.

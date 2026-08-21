# om-plan GitLab 반영 안내 (집에서 실행용)

작성일: 2026-08-21. 목적: om-plan을 GitLab에 **깔끔하게 "기능만"** 반영하는 절차. (사용자가 집에서 실행.)

## 원칙 — GitLab엔 기능(도구)만, 문서는 GitHub

- **GitLab(`kb-datacatalog-upgrade-checker`)** = 실제 돌아가는 **검사기 도구만**.
- **GitHub(`om-skill-develop`)** = 설계·검토·결정·논의 문서(원천). **GitLab엔 안 올라감.**
- 이 둘은 **다른 저장소**라, clean-export를 올리면 **자동으로 기능만** 올라간다.

## 확인된 현재 상태 (Claude 실측, 2026-08-21)

clean-export `work/kb-datacatalog-upgrade-checker-om-plan-cli`:
- CLI = **plan 전용**(plan / start / check / plan-session-start / plan-preflight / plan-validate / plan-resume). apply·watch·risk 등 잡동사니 없음.
- **설계·논의 문서(om_plan/*.md) 안 섞임.** (도구 파일만.)
- **Q9(exit2→CI 성공 처리) 포함** — 커밋 `2194c414ab` "map review-ready CI".
- 검증 3커밋(direct_tests 정리·A′·A-2) 포함, HEAD `7c544efb2b`.
- 브랜치 `codex/om-plan-verified-gates-20260820` — **gitlab-checker/main엔 아직 미반영**.

## 현재 진행 (2026-08-21)

- ✅ **GitHub(origin=openmetadata-test) push 완료** — 브랜치 `codex/om-plan-verified-gates-20260820` @ `7c544efb2b` (Claude가 `git ls-remote`로 확인, 로컬과 일치). → **다른 컴퓨터는 GitHub에서 이 브랜치를 받으면 됨.**
- ⏳ **GitLab(gitlab-checker) push 보류** — CloudFront가 **HTTP 504** 반환(인증 문제 아님, 서버 앞단 장애). **서버 회복 시 재시도.**
- 추가 커밋·MR·main 병합: 없음.

## 누가 무엇을 하나 (구체)

| 단계 | 누가 | 비고 |
|---|---|---|
| GitLab 브랜치 push **재시도** | **Codex** | 서버(504) 회복 후. Codex가 트리·자격증명 보유 |
| MR 생성(branch→main) | **Codex** 또는 사람(GitLab 웹) | 둘 다 가능 |
| 파이프라인 실행 | 자동(GitLab) | push/MR 시 |
| 4모드 초록불 확인 | Codex/사람 | 빨간불이면 원인 조치 |
| **최종 merge(병합)** | **사람(당신)** | 초록불 확인 후 승인 |

> Claude는 이 과정에 **커밋·push·merge를 하지 않는다**(검증·기록만). GitLab 반영 요청은 **Codex에게**.

## 절차

### 1단계 — 브랜치를 remote에 올림 (Codex)
- ✅ GitHub push 완료.
- ⏳ **GitLab push는 504로 보류** → 서버 회복 후 **Codex에게 재시도 요청**(아래 [Codex 재시도 요청]). main 병합은 아직 안 함.

### 2단계 — (GitLab 서버 회복 후) MR 생성 + 파이프라인
- gitlab-checker에서 `codex/om-plan-verified-gates-20260820` → `main`으로 **MR** 생성. (Codex 또는 GitLab 웹)
- **파이프라인 실행 → 4모드(initial/feature/change/upgrade) end-to-end 초록불 확인.**
  - Q9가 들어있어 정상 계획(exit 2)이 **성공(초록)**으로 나와야 정상. 빨간불이면 원인 확인.
- 여기서 **처음 실제로 돌리는 것**이라 문제가 나올 수 있음 — 나오면 고치고 재실행.

### 3단계 — 사람 병합 승인
- 초록불 확인 후 **사람(당신)이 MR을 merge** → GitLab main에 om-plan 기능 반영 완료.

## [Codex 재시도 요청 — GitLab 서버 회복 후, 복붙]

```
GitLab(gitlab-checker) 브랜치 push가 CloudFront 504로 실패했었다. 서버 회복됐는지 확인하고,
work/kb-datacatalog-upgrade-checker-om-plan-cli 의 브랜치 codex/om-plan-verified-gates-20260820 (HEAD 7c544efb2b)를
gitlab-checker에 다시 push 해라. main 병합·MR·추가 커밋은 하지 마 — 브랜치 push 성공만 확인하고 결과(원격 SHA) 알려줘.
```

## 주의

- **문서는 안 올라간다**(다른 저장소라 자동). 혹시 clean-export에 설계문서가 보이면 반영 전 제거.
- **파이프라인 초록불 전엔 merge 금지.** "책상 위 검증"은 됐지만 실제 CI는 이 단계가 처음.
- 역할: **push·MR·파이프라인은 Codex/사람**, Claude는 검증·기록.
- 이건 **om-plan(계획 도구) 반영**이다. 커스텀을 실제 반영·운영하는 전체 자동화는 om-apply·verify까지 있어야 완성(별개).

## [Codex 요청 — 1단계용, 복붙]

```
목적: 검증 완료된 om-plan 변경을 remote에 올려 (a) 다른 컴퓨터도 받게, (b) GitLab 반영 준비.
대상: work/kb-datacatalog-upgrade-checker-om-plan-cli 의 브랜치 codex/om-plan-verified-gates-20260820 (HEAD 7c544efb2b).
  → plan 전용 트림된 기능본, Q9 포함, 설계문서 안 섞임(확인됨).
할 일: 이 브랜치를 gitlab-checker remote에 push (origin=github openmetadata-test에도 백업 push). main 병합 금지 — 브랜치만. push 후 브랜치 URL 알려줘.
금지: commit 추가·main 병합·배포. 브랜치 push까지만.
```

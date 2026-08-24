# Codex om-apply v1 구현 결과

작성일: 2026-08-24  
정본 지시서: `61_omapply_구현지시서_20260821.md`  
구현 작업 트리: `work/kb-datacatalog-upgrade-checker-om-plan-cli`  
브랜치: `codex/om-plan-verified-gates-20260820`

## 1. 결론

`/om-apply` v1을 clean-export 작업 트리에 구현했다. 승인된 `/om-plan` 결과를 고정한 뒤 LLM이 제품 코드를 커밋 단위로 반영하고, 검사기는 승인 계획·실제 Git 이력·최종 관리자료를 대조한다.

정적 정합과 빠른 검사까지 통과한 최종 상태는 `static_consistent_awaiting_verify`뿐이다. 이는 배포 승인이나 계약 테스트 통과가 아니라, **동일 후보 commit을 `/om-verify`에 넘길 수 있는 상태**다.

이번 작업에서는 commit·push·MR을 수행하지 않았다.

## 2. 반영 위치

### 공통 apply 엔진 — 신규

- `harness/acgh/applycore/workflow.py:118` — 실행계획 스키마, 단위·action DAG, path-remap, 필수 경로·검사 목록 검증
- `harness/acgh/applycore/workflow.py:322` — 승인된 plan binding, 입력 commit, 저장소 identity, checker SHA, 등록자료 digest를 고정하는 `apply start`
- `harness/acgh/applycore/workflow.py:690` — 계획·Git transaction/net diff·최종 관리자료를 대조하는 `apply check`
- `harness/acgh/applycore/workflow.py:637` — 컴파일·생성기용 빠른 검사 실행과 실행 후 dirty 변경 차단
- `harness/acgh/applycore/schema/` — request, execution plan, execution report, result 스키마 4종
- `harness/acgh/applycore/boundary.py:8` — 공통 엔진에 BANK-OM/OpenMetadata/om-temp/integration 의존이 들어오지 못하게 하는 경계 검사

### OpenMetadata 관리자료 연동 — 신규

- `harness/acgh/integrations/om/apply.py:31` — 등록자료 snapshot과 ID별 경로·Contract·테스트·공유 소유정보 대조
- `harness/acgh/integrations/om/apply.py:175` — 새 공유 경로를 자동 승인하지 않고 unresolved question으로 반환
- `harness/acgh/integrations/om/__init__.py` — apply adapter 노출

### CLI·Claude 배선 — 신규/최소 변경

- `harness/om_workflow.py:373` — `apply start`, `apply check` CLI
- `.claude/skills/om-apply/SKILL.md:1` — LLM 구현 절차, 범위 이탈 STOP, 병합 실패 3단 복구, verify 인계 규칙
- `.claude/settings.json:47` — 기존의 apply 명령 전면 금지만 제거하고 push/tag/deploy 금지는 유지
- `CLAUDE.md:21`, `README.md:18` — 역할과 상태 의미 명시

### 반례·회귀 검사

- `harness/tests/test_om_apply_counterexamples.py:353` — OA-01~20 및 DAG 실행순서·공통 core 경계 반례
- `harness/tests/test_claude_wiring.py`, `harness/tests/test_plan_cli.py` — skill/CLI 배선 회귀

## 3. 재활용과 신규 구현 구분

### 재활용한 안전 primitive

- `acgh.gitprim`: ref→SHA 고정, ancestry, commit/net diff, tree entry, blob 읽기
- `acgh.binding`: 검사 대상 commit 고정
- `acgh.verdict`: 4종 verdict, canonical digest, 종료코드
- `acgh.plancore.schema/paths`: schema 읽기, atomic write, 경로 내부 제한, directory digest
- 기존 OM collector의 `customization-relations`, `contract-tests`, `id-contract-consistency`: direct_tests를 계약 테스트로 합치지 않는 현재 정책 그대로 사용

### 신규 구현한 부분

- A형 수명주기: LLM이 코드를 작성하고 검사기는 사후 판정
- 승인 계획에 결속된 typed unit/action DAG
- keep/move/replace/delete/generate action과 다대일·일대다 명시
- 계획·실제 Git 이력·관리자료 3방향 대조
- scope variance·신규 공유 경로 STOP
- 생성물·migration 대응 관계, merge recovery 순서, verify handoff

### 재활용하지 않은 부분

예전 B형 `registration_prep.py apply_plan` orchestration은 사후 등록자료 작성 흐름이며, 현재의 3-digest plan binding·per-ID 관계·계약-only 테스트 정책·Contract/shared 관리자료 범위와 맞지 않아 재사용하지 않았다. 저수준 Git/digest/path primitive만 현재 모듈에서 import했다.

## 4. OA-01~20 반례 결과

| 반례 | 실제 확인 결과 |
|---|---|
| OA-01 | 계획 밖 제품 경로와 Manifest를 함께 늘려 자기 정당화하면 STOP |
| OA-02 | 중간 commit에서 수정 후 최종 원복하면 required final effect 부재로 block |
| OA-03 | 최종 diff 경로가 어떤 ID에도 귀속되지 않으면 block |
| OA-04 | Manifest 경로에 최종 효과가 없고 unchanged 선언도 없으면 block |
| OA-05 | move/replace에 `to`가 없으면 schema 단계에서 거부 |
| OA-06 | 절대·상위·역슬래시 경로와 symlink target을 거부 |
| OA-07 | 최종 candidate에 remap target이 없으면 block |
| OA-08 | 합법적인 move에서 이전 경로가 최종 HEAD에 없는 것은 허용 |
| OA-09 | 새 공유 경로는 자동 승인하지 않고 unresolved question + STOP |
| OA-10 | 공유 소유 ID가 하나라도 빠지면 block |
| OA-11 | schema source만 바뀌고 생성물이 갱신되지 않으면 block |
| OA-12 | 대응 migration 중 한쪽만 바뀌면 block |
| OA-13 | 의미 있는 unit/action이 없는 빈 계획은 schema/DAG 단계에서 거부 |
| OA-14 | `id-contract-consistency=false`이면 구현 시작 전에 거부 |
| OA-15 | 정합이 모두 맞아도 apply 최종 상태는 `static_consistent_awaiting_verify` |
| OA-16 | 완료 후 candidate SHA가 움직이면 기존 apply 결과 재사용을 거부 |
| OA-17 | Manifest `direct_tests`가 Contract tests로 재유입되지 않음을 확인 |
| OA-18 | start가 Contract/shared/Manifest를 자동 작성하지 않음을 확인 |
| OA-19 | Git 변경은 깨끗해도 계획된 정적 assertion이 사라지면 block |
| OA-20 | 공유 파일을 한 ID 단위만 독점 변경하면 block |

추가로 YAML 배열 순서가 의존순서와 달라도 `apply start`가 topological unit order를 계산해 실행 보고서에 기록하는 반례를 확인했다.

실행 명령:

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=harness \
  ../review-openmetadata-test/.venv/bin/python -m pytest -q \
  harness/tests/test_om_apply_counterexamples.py \
  harness/tests/test_claude_wiring.py \
  harness/tests/test_plan_cli.py

PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=harness \
  ../review-openmetadata-test/.venv/bin/python -m pytest -q harness/tests
```

집중 반례와 기존 전체 회귀 모두 종료코드 0이었다. 완료 근거는 개수 자체가 아니라 위 반례들이 잘못된 입력에서 실제로 pass를 뒤집었다는 점이다.

## 5. 최종 상태와 verify 인계

성공 결과는 다음 정보를 같은 결과 digest에 묶는다.

- 승인된 plan digest와 apply context digest
- 제품 start/candidate commit 및 candidate tree
- checker repository identity와 SHA
- 등록자료 기준/최종 digest와 관리파일별 digest
- `/om-verify`가 실행할 Contract 테스트 ID

계약에 관한 LLM 설명은 `contract_claims`에만 남으며 pass 판정에 사용하지 않는다. `/om-verify`에는 정확한 candidate SHA/tree와 required tests가 전달된다.

## 6. 남은 위험과 운영 전 필수 확인

1. **실행 의미 미검증:** apply는 전체 Contract·UI·데이터 마이그레이션 테스트를 실행하지 않는다. 반드시 `/om-verify`가 같은 candidate SHA에서 실행되어야 한다.
2. **계획 품질 의존:** 정적 assertion과 action 계획 자체가 부족하면 그 범위 밖의 업무 의미까지 증명하지 못한다. 사람의 계획 검토가 계속 필요하다.
3. **빠른 검사 실행 경계:** 빠른 검사는 승인 계획의 argv를 로컬에서 실행한다. 운영에서는 신뢰한 checker commit과 격리된 build 환경을 전제로 해야 한다.
4. **STOP 후 승인 재개 자동화 없음:** scope variance와 신규 공유 경로는 이번 v1에서 멈추고 질문을 남긴다. 승인 뒤 같은 run을 이어가는 채널은 아직 없으므로 새 승인 계획/run으로 재시작해야 한다.
5. **apply 전용 변경 방지 Hook 없음:** 최종 checker가 범위 이탈을 탐지하지만 LLM 편집 순간을 모두 사전 차단하는 apply 전용 Hook은 이번 v1 범위에 없다.
6. **실제품 예행연습 미실행:** 이번 결과는 임시 Git 저장소 반례와 checker 전체 회귀 결과다. 실제 OpenMetadata 1.13.1 change와 1.13.2 upgrade 자료로 `start → LLM 반영 → check → verify 인계` 예행연습이 남아 있다.

## 7. 다음 단계

Claude가 OA-01~20을 별도 반례로 재검증한다. 이후 실제 1.13.1/1.13.2 자료 예행연습에서 `static_consistent_awaiting_verify`와 동일 candidate 인계를 확인한 뒤에만 commit·push 여부를 결정한다.

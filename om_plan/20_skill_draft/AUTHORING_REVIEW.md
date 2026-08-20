# om-plan 작성 검토 보고서

작성일: 2026-08-19
작성자: Claude (설계·기록 역할). 기술적 강제 배선은 Codex 몫으로 §7에 분리.
근거 문서: `01_LLM_스킬_하네스_MD_공통_작성규칙_20260819.md`, `02_Claude_Code_슬래시커맨드_MD_차이점_전용규칙_20260819.md`.

## 0. 산출물과 설치 위치

- 스테이징(현재): `skill_develop/om_plan/20_skill_draft/.claude/skills/om-plan/` (om-skill-develop, 영구·Claude 소유).
- 설치 목표(런타임): 검사기 저장소 `kb-datacatalog-upgrade-checker` 루트의 `.claude/skills/om-plan/`.
- 런타임 파일(GitLab로 감): `SKILL.md`, `references/{request-format,proposal-format,initial,feature,change,upgrade,example-initial-proposal,example-change-proposal,example-feature-proposal,example-upgrade-proposal}.md`.
- 개발 전용(GitLab에 안 감): 본 보고서와 `README.md`는 런타임 폴더 밖(`20_skill_draft/`)에 둔다.

## 1. 조사 범위

- 읽은 참고문서: 공통편(C-01~C-20, P/D/E/V/S/M 모듈, 한글 규칙, 안티패턴 A-01~A-12, 템플릿), 전용편(CC-01~CC-22, 단계 모듈 R~X, frontmatter 표, 안티패턴 A-CC-01~14, 수용 체크리스트) 전문.
- 조사한 검사기: `harness/om_workflow.py`(CLI 동사), `harness/acgh/plancore/validate.py`(proposal 검증·floor), `harness/acgh/integrations/om/collectors.py`(모드별 사실 수집·validate_proposal), `harness/acgh/verdict.py`(종료코드), `harness/registrations/om-temp-1.13.1/`(answer-key 형식).
- 확인한 사실: `.claude/`·`CLAUDE.md` 없음(충돌 없음), 하네스·증거 폴더 위치, 종료코드 매핑, 매니페스트 schema_version 2 형식.
- 확인하지 못함: 실제 설치된 Claude Code 버전(`claude --version` 미확인), 운영자 환경의 Python 3.11+ 인터프리터 실경로.

## 2. 커맨드 프로필

| 항목 | 판정 | 근거 |
|---|---|---|
| 주단계 | P(계획) | 최종 사용자 결과 = 검토·승인 가능한 등록 계획. 전용편 §3.2 "결과가 실행 전 계획이면 P" |
| 보조단계 | R(사실수집), V(검증) | plan start=R, plan check=V. 오케스트레이션 성격 있으나 결과물이 단일 계획이라 X 아님 |
| 부작용 | 로컬 쓰기 | `proposal/`·run 폴더 쓰기. 운영·외부 쓰기 없음(apply는 범위 밖) |
| 호출 주체 | 사용자 전용 | 부작용+명시 호출 필요 → `disable-model-invocation: true`(CC-02) |
| 컨텍스트 | inline | 모드 판단·검증 결과에 따라 사람 중간 판단 필요(CC-09 fork 부적합) |
| 실행 방식 | 동기 | 운영자가 판정을 보고 다음 결정. CC-10 |
| 입력 형식 | 위치 인수 1개(`$ARGUMENTS`=요청파일) | CC-04 |
| 배포 대상 | Claude Code 전용 | `disable-model-invocation`·`argument-hint` 사용(CC-13) |

- 단일 커맨드 정당성(A-CC-11 회피): 조사·계획·검증을 모두 포함하지만 배포·삭제 같은 이질적 생명주기는 배제. 중간 승인 게이트는 owner 미정 STOP·plan check 판정으로 파일 내에서 분리.

## 3. 적용 규칙 매트릭스

| 규칙 ID | 판정 | 반영 위치 | 이유 |
|---|---|---|---|
| C-02 | 적용 | SKILL "사용/사용하지 않는다" | 발동·비발동 경계 |
| C-03 | 변형 | "완료 조건" 5상태 | 관찰 가능한 종료코드로 완료 정의 |
| C-06 | 적용 | 본문 165줄+references 분리 | 모드별 상세를 참조로 점진 공개 |
| C-08 | 적용 | 절차 각 단계 입력·행동·산출물·검증·실패 | 조건-행동-검증-실패 형식 |
| C-12 | 적용 | 정확한 명령·경로·종료코드 | 지어내기 금지 |
| C-16 | 적용 | 6단계 검증 루프(block→수정→재검증) | 반복 검증 |
| C-18 | 적용 | "안전·권한" 외부지시=데이터 | 인젝션 방어 |
| E-06 | 변형 | 계획→검증→(승인)→반영 분리 | apply는 범위 밖, 계획까지만 |
| E-07 | 적용 | 3·4단계 "지어내지 않는다" | 사실은 discovered-facts만 |
| V-08 | 적용 | 완료/승인/조건부/실패/검증불가 | 상태 분리(CC-16과 결합) |
| S-02 | 외부 강제 | §7로 이관 | Markdown≠보안경계 |
| CC-02 | 적용 | `disable-model-invocation: true` | 부작용 커맨드 자동발동 차단(A-CC-03) |
| CC-04 | 적용 | argument-hint+$ARGUMENTS+검증 | 인수 계약 |
| CC-06 | 적용 | allowed-tools 최소·좁은 패턴 | 사전승인이지 제한 아님 |
| CC-07 | 외부 강제 | §7 | 실제 차단은 hook·permissions |
| CC-16 | 적용 | 완료 조건 5상태 | 미실행 검사 통과 표기 금지 |
| CC-17 | 적용 | 최종 보고 형식(P 단계형) | 단계-출력 일치 |
| C-01/C-05 | 제외 | — | 검사기 CLAUDE.md 없음, 우선순위 충돌 없음 |
| 전용 §7.6 O 모듈 | 제외 | — | 운영·배포 부작용 없음 |

## 4. frontmatter 결정

| 필드 | 값 | 유지 이유 | 검토한 대안 |
|---|---|---|---|
| `name` | om-plan | 디렉터리명과 일치, /명령명 근거 | 생략(디렉터리로 충분하나 명시가 안전) |
| `description` | 무엇+언제+제외 | 메뉴 가독성·비발동 경계 | 짧은 한 줄(제외 조건 누락 위험) |
| `argument-hint` | `[요청파일.yaml]` | 인수 1개 호출 예시(CC-04) | 생략(호출법 불명확) |
| `disable-model-invocation` | true | 로컬 쓰기+명시 호출(CC-02, A-CC-03) | 미설정(자동발동 위험) |
| `allowed-tools` | Read/Write/좁은 Bash 2종+git read | 최소 사전승인(CC-06) | `Bash(*)`(과다, 거부) / 미설정(매번 프롬프트) |
| `user-invocable` | 미설정(기본 true) | 운영자가 직접 실행 | false(실행 불가, 부적합) |
| `context`/`agent`/`background` | 미설정 | inline·동기(CC-09/10) | fork(중간 판단 필요로 부적합) |
| `paths` | 미설정 | 수동 전용, 경로 자동발동 불필요 | — |

## 5. 문장별 검토 결과

- 방법: 디스크 저장본을 `nl`/grep 줄 단위로 재검토. 수정 후 재검사.
- 발견·조치:
  - `references/feature.md`의 "필요하면"(공통 7.3 모호어) → "별도 파일로 제출할 때는"으로 수정. 재검사 결과 전체 모호어 0건.
  - allowed-tools를 제한으로 서술한 문장 없음(CC-06 준수).
  - fail-closed 문구 7곳 일관(analysis_error→통과 보고 금지).
- SKILL.md 165줄(공통 C-06의 200~500 검토선 이내). references 6개 모두 30~48줄.

## 6. 기계적·구조 검사 결과

| 검사 | 결과 | 증거 |
|---|---|---|
| YAML frontmatter 파싱 | 통과 | 5개 키 로드 성공 |
| 디렉터리명=name=/명령명 | 통과 | dir `om-plan` = name `om-plan` |
| 인수 힌트·$ARGUMENTS 일치 | 통과 | argument-hint 1곳, $ARGUMENTS 2곳 |
| 참조 파일 존재·링크 일치 | 통과 | references 6개 모두 존재·본문 링크됨 |
| 무시필드(name/paths) 오용 | 통과 | SKILL.md 형식이라 name 유효, paths 없음 |
| 모호어 | 통과 | 수정 후 0건 |
| allowed-tools 제한 오서술 | 통과 | 없음 |

- 미수행: 새 세션 발동/실행 시뮬레이션(설치 후 검사기에서 수행 권장), `claude --version` 확인.

## 7. 기술적 강제 필요 항목 (Codex 배선)

MD 지침만으로 막히지 않는 것. 검사기 설치 시 아래를 배선한다.

| 위험 행동 | MD 지침(현재) | 추가 강제(Codex) | 상태 |
|---|---|---|---|
| 세션 marker 없이 plan check 통과 | — | UserPromptSubmit 훅(세션 marker) + `.plan-active` run marker, default-deny | 하네스에 이미 존재, 훅 설치 필요 |
| apply(등록 반영) 무단 실행 | "범위 밖" 명시 | permissions로 apply 명령 ask/deny | 미배선 |
| 사용자 직접 `/om-plan` 우회 | — | UserPromptExpansion 훅으로 호출·인수 검사(CC-21) | 미배선 |
| owner 지어내기 | proposal-format 규칙 | validate_proposal이 이미 거부(block) | 코드 강제됨 |
| Python 3.9 오실행 | 사전조건 명시 | 래퍼가 인터프리터 버전 확인 후 실행 | 미배선 |

## 8. 남은 위험과 미확인

- allowed-tools 패턴 `Bash(python3 *...)`은 운영자가 `python3.12`·절대경로 인터프리터로 부르면 매칭 안 됨 → 사전승인 누락(안전한 실패: 권한 프롬프트만 뜸). 인터프리터 확정 후 패턴 보강 검토.
- 설치된 Claude Code 버전 미확인. `disable-model-invocation`·`argument-hint` 미지원 버전이면 동작 다를 수 있음(CC-19). 설치 전 `claude --version` 확인 필요.
- feature 모드의 변경 범위 수집은 initial 같은 자동 커밋 인벤토리 분기가 없음 → 근거 제시 방식을 실제 feature 실행으로 1회 검증 필요.
- 본 스킬은 로컬 검증만 통과. 실제 발동/시나리오 평가(정상·인수누락·잘못된 모드·인젝션)는 설치 후 새 세션에서 수행.

## 9. 백지시험 결과 (2026-08-20)

방법: 대화 맥락이 없는 clean-room 서브에이전트에 SKILL.md + references + `discovered-facts.json`만 주고 initial 모드 BANK-OM-001 매니페스트를 작성시킴. `plan start`는 미리 실행(정답지를 임시 브랜치에서 숨겨 백지 initial 상태 확보, registered-customizations=0, 9 commits, 112 net paths).

### 통과 (지시서가 통한 부분)

- 커밋 식별: 서브에이전트가 트레일러만으로 BANK-OM-001 = 커밋 `23ec5bf`("restore InstanceCode customization"), 단일 ID(비모호) 정확 판정.
- **매니페스트 changed_paths 48개가 사람 answer-key와 완전 일치(48/48, 차집합 0).** 경로 지어냄 없음.
- owner 처리: 요청 owner=null → owner 미채움 + `unresolved_questions` + `next_step_blocked: true`. 규칙대로.
- shared_impact=[], required_tests=[] 적절 처리.

### 실패 (지시서 구멍 → plan check가 analysis_error로 fail-closed)

`attempt-0001.json`의 `evidence_ref_errors`로 확인. 근본 원인 2가지, 모두 `references/proposal-format.md`의 스펙 부족.

1. **`evidence_refs` 형식 미정의(치명적).** 검사기(`validate.py:_parse_ref`/`_dereference`)는 evidence_refs를 `<run-dir 상대 파일>#<JSON-Pointer>` 로 해석하고 실제로 읽어 대조한다. 허용 형식:
   - 문자열 `"discovered-facts.json#/canonical_payload/items/<i>/value/..."` 또는
   - 매핑 `{ref: "discovered-facts.json#/<pointer>", expected: <값>}` (expected 있으면 실제값과 == 검사).
   - 제약: 파일은 run-dir 안이어야 하고 `proposal/` 자기참조 금지. `no_change`는 반드시 `discovered-facts.json#`로 시작 + `expected` 필수.
   - 서브에이전트는 스펙이 없어 `customization-commits#sha=...`를 지어냈고 → 파일 없음으로 전부 실패.
2. **decision 필수 필드 미명시.** `_validate_required_decision_fields`가 각 decision에 다음 6개를 요구: `subject`, `decision`, `decision_source`, `evidence_refs`, `affected_customization_ids`, `required_follow_up`. 현재 proposal-format.md에 이 목록이 없다.

### 서브에이전트가 스스로 보고한 추가 모호점 (softer)

- 제안서 문서 shape의 완성 예시 부재(매니페스트 필드 + decisions/findings 결합 형태를 지어냄).
- `status`/`kind` 허용값 미정의(→ `proposed`/`core-patch` 추측).
- `title` 사실 출처 없음(→ 커밋 subject 재사용).
- `criticality` 참조엔 요구되나 사실 없음(→ 생략).
- `contracts` 초안을 요구하나 test/contract 사실이 비어 근거 불가.

### 조치 항목 (2026-08-20 개정 반영 완료)

- [완료][P0] `proposal-format.md`에 evidence_refs 정확 형식(`<파일>#<JSON포인터>` + `expected`) + decision 필수 6필드 + `decision_source` enum(proposed/human_input/observed) + no_change 요건 추가.
- [완료][P0] 완성 예시 `references/example-initial-proposal.md` 수록. 검증: YAML 파싱 OK, decision 6필드 충족, evidence_refs 2개가 실제 discovered-facts.json에서 expected와 일치(포인터 해석 성공).
- [완료][P1] `initial.md`에 `status=proposed`/`kind=core-patch` 판단·`title`(커밋 subject 초안)·`criticality`(미상/질문)·`contracts`(근거 없으면 질문) 규칙 추가.
- [완료] SKILL.md 5단계·참조 로딩에 예시·형식 규칙 연결.
- [완료][P0] 개정본 clean-room 재시험 end-to-end **통과**. 새 `plan start`(run `om-plan-initial-20260820T012840`) → 백지 서브에이전트가 개정 references만으로 BANK-OM-001 proposal 작성(추측 불필요 보고) → `plan check` 결과: **verdict=approval(종료코드 2), review_state=review_ready, evidence_ref_errors=0, reasons=0**. 지난 실패(analysis_error, evidence_ref_errors 5)와 대비해 근본원인 해소 확인.
- 근거 산출물 백업: `scratchpad/retest-evidence/`(통과본 BANK-OM-001.yaml, attempt-0001.json, validation-result.json), 이전 실패본은 `scratchpad/cleanroom-evidence/`, 정답지 `scratchpad/answer-key-backup/`.

### 재시험 해석

- `approval`(2)은 "사람이 계획을 검토할 준비 완료"이며 배포·반영 승인이 아니다. owner=null이라 proposal이 owner 질문 + `next_step_blocked`로 STOP했음에도 구조가 유효해 review_ready가 나온 것은 정상.
- 검증된 것: initial 모드의 형식·evidence·decision 스펙이 clean-room에서 자기완결적으로 통과.

## 10. change 모드 clean-room 검증 (2026-08-20)

- 시나리오: BANK-OM-007 post_change_reconcile. 요청 refs `custom_baseline=a2393b5f`(restore Tibero) → `candidate=8ac18ad`(complete Tibero coverage), 등록자료 present(7개), requirement 지정.
- 자동 탐지: 검사기가 두 커밋 사이 변경 2경로(`serviceConnection.ts`, `DatabaseServiceUtils.test.tsx`) 자동 추출. 공유경로 교집합 없음.
- 백지 서브에이전트: change.md + proposal-format.md + example만으로 proposal 작성. evidence_refs를 관찰경로에 바인딩, owner=null STOP, required_tests는 지어내지 않고 `required:false/not_run` + apply 단계 follow-up로 처리, shared_impact=[].
- **plan check 결과: verdict=approval(2), review_state=review_ready, evidence_ref_errors=0, reasons=0.** 통과.
- 근거 백업: `scratchpad/change-evidence/`(예정).

### change 재시험에서 나온 개선점 (P1, 통과했으나 개선 여지)

- [P1] plan 단계의 required_tests 표현이 under-spec. `required:true`+미실행은 자동 block인데 /om-plan은 테스트를 실행할 수 없고 결과 조작도 금지 → 유일한 무모순 표현이 `required:false`+`not_run`+follow-up. change.md에 "post_change_reconcile의 미실행 필수 테스트 표현법"을 명시하면 추측 제거.
- [P1] `example-change-proposal.md` 부재(initial 예시만 있음). change용 완성 예시 추가 권장.
- [P1] 등록 ID→contract/test 매핑 사실 부재. 서브에이전트가 requirement 텍스트+파일 범위로 추론 후 질문으로 hedge. 사실에 매핑이 있으면 근거화 가능(검사기 개선 후보 → Codex).

## 11. feature 모드 clean-room 검증 (2026-08-20)

- 시나리오: 신규 BANK-OM-008. 요청 refs `custom_baseline=8ac18ad`, `requirement`(감사로그 보존기간 신규 커스터마이징), 등록자료 present(7개). feature는 diff 자동수집 없음(requirement 기반 확인).
- 백지 서브에이전트: 신규 ID를 `registered-customizations` 사실로 근거화(expected=7개 목록, BANK-OM-008 부재 증명), 미구현이라 changed_paths=[]·kind=unknown, observed 결정 1개(ID 가용성)+proposed 결정 1개(계획, evidence_refs=[]), 미지 5건을 unresolved_questions+STOP.
- **plan check: verdict=approval(2), review_ready, errors=0, reasons=0.** 통과.
- 개선점(P1): feature 완성 예시 부재로 "proposed 결정(비관찰, evidence_refs 빈 리스트) 형태"를 추론해야 했음. `example-feature-proposal.md` 권장.

## 12. upgrade 모드 clean-room 검증 (2026-08-20) — 통과했으나 MD 자기완결성 미달

- 시나리오: 공식 1.13.1→1.13.2(인접). 공식 1.13.2-release(2763bf97)를 제품 저장소로 fetch, 로컬 공식문서(version_token 1.13.2, docker) 작성. 요청 refs=official_base/official_target/custom_baseline, hop_policy=adjacent_only, deployment_method=docker, official_documents 1건.
- plan start: `official-upgrade-paths` 1702경로 + doc snapshot 1건 수집.
- **plan check: verdict=approval(2), errors=0, reasons=0.** 통과.
- **[P0 핵심 발견] upgrade는 references만으로 자기완결이 안 된다.** 백지 서브에이전트가 유효한 upgrade proposal을 만들려고 **검사기 코드를 직접 읽어야 했다.** 참조에 없고 코드에만 있는 강제 규칙:
  1. `shared_impact`가 옵션이 아님. 검사기가 `official-upgrade-paths ∩ shared-path-owners`를 자동 계산(collectors.py ~L455)하고 **교집합 각 경로(이번엔 23개)마다 소유자 선언을 강제**. upgrade.md는 이를 아예 언급 안 함 → `[]`로 냈으면 23건 block.
  2. operations-checklist: `category ∈ {db_migration, reindex, configuration}` finding은 `operations[].finding_id`로 참조돼야 함(코드 전용).
  3. doc-code-crosscheck: `crosschecks[].relation`이 **정해진 5개 문자열 enum**(문서 어디에도 없음).
  4. pointer_movement은 `shared_code_definitions_delta` 동반 선언 필요(코드 전용).
- 결론: initial·change·feature는 references가 (거의) 자기완결. **upgrade.md는 floor-valid 문서까지만 가능하고, valid-upgrade 문서를 만들려면 위 4개 규칙을 references로 끌어올려야 함.**

### upgrade 조치 항목

- [완료][P0] upgrade.md에 검사기 강제 4규칙을 collectors.py(478~517) 코드에서 확인해 명시:
  1. shared_impact 자동교집합(official-upgrade-paths ∩ shared-path-owners) 경로별 필수선언.
  2. 운영 finding(category db_migration/reindex/configuration)의 id를 operations[].finding_id로 연결.
  3. crosschecks[].relation은 5개 enum(documented_and_observed/documented_not_located/code_change_not_documented/documented_contradicts_observed/not_applicable)만.
  4. pointer_movements 선언 시 shared_code_definitions_delta 동반 필수.
- [완료] 요청 필수필드(hop_policy=adjacent_only, versions.base/target, deployment_method, official_documents{source(로컬파일 가능)/version_token/deployment_methods}, refs.custom_baseline) 명시.
- [완료] 검증: 보강 규칙을 통과본 upgrade-1.13.2.yaml과 대조 — shared_impact 23, crosscheck relation=documented_and_observed, operations+db_migration finding 연결, pointer_movements 없음 모두 일치.
- [P1 미완] `example-upgrade-proposal.md` 완성본 수록(통과본 239줄 기반). 현재는 upgrade.md 말미에서 scratchpad 통과본을 참고용으로 지시.
- 근거 백업: `scratchpad/{feature,upgrade,change}-evidence/`.

### 근본원인(왜 처음에 부족했나) 회고

- 참고자료(01/02) 결함 아님. 01 A-05("링크만 던지고 핵심 규칙 안 씀")가 정확히 이 실수를 경고했음.
- 실행 실수: 코드 정독을 공통 검증 + 먼저 테스트할 initial에 집중하고, upgrade 전용 검증(collectors.py 478~517)은 안 읽고 설계문서 개념 목록으로 대체함. upgrade를 마지막에 테스트하려 해 근거를 가장 얕게 잡음.
- 이 유형 구멍은 문서검토로 안 보이고 실제 실행 시 발동 → clean-room 테스트가 안전망으로 작동해 4규칙 전부 검출.

## 13. 4모드 종합

| 모드 | plan check | references 자기완결성 |
|---|---|---|
| initial | approval ✅ | 자기완결(개정 후) |
| change | approval ✅ | 거의 자기완결(P1: 예시·미실행 테스트 표기) |
| feature | approval ✅ | 거의 자기완결(P1: 예시) |
| upgrade | approval ✅ | **미달(P0: 코드전용 규칙 4종 미수록)** |

4모드 모두 clean-room에서 통과하는 proposal 생성 가능. 단 upgrade는 references만으로는 부족하고 코드 참조가 필요했다 → upgrade.md 개정이 실질 남은 과제.

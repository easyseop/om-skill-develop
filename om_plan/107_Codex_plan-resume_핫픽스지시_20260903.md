# Codex 핫픽스 지시 — plan-resume 경로 복구 (#20 P0) + analysis_error resume (#21)

작성일: 2026-09-03. 근거: Mac 재현 밤작업(106)에서 실행자 세션이 근본원인까지 규명. Claude가 코드 위치 실측 완료(구현은 코덱스). 대상: clean-export 검사기 작업트리. **커밋·push·MR 금지**(Claude 검증 후 대행).

## #20 (P0) — plan-resume 경로 전체 사용 불가

**증상**: plan check가 block → plan-resume → plan check 재시도 시 항상 `TRUSTED_INPUT_LOCK_DIGEST_MISSING`("session marker does not contain a trusted input lock digest")로 실패. **resume 경로 전체가 죽어있음.**

**근본원인(실측)**:
- `record_trusted_input_lock_digest(marker, digest)`가 **plan start에서만** 호출됨 — `harness/om_workflow.py:253` (`result = run_preflight(...)` 직후).
- **plan-resume 핸들러**(`harness/om_workflow.py:459-477`)는 `resume_proposal_run(args.run_dir, marker, ...)`만 호출하고 마커에 digest를 기록하지 않음. → resume가 만든/재바인딩한 세션 마커엔 digest가 없어, 이후 plan check가 통과 불가.

**요구**: plan-resume이 성공적으로 run에 재바인딩한 뒤, 그 run의 `input-lock.yaml`의 `input_lock_digest`를 읽어 `record_trusted_input_lock_digest(marker, digest)`로 마커에 기록하게 하라. (resume.py는 이미 `input-lock.yaml`을 읽어 무결성 검증함 — `_verify_immutable_files`. 같은 값 재사용.)
- 위치 후보: `resume_proposal_run` 내부(재바인딩 성공 직후) 또는 `om_workflow.py`의 plan-resume 핸들러(resume 결과 확인 후). 마커·digest 결속이 start와 동일 의미가 되게.
- **완화 금지**: digest는 반드시 그 run의 input-lock에서 와야 함(외부 주입 금지). input-lock이 변조됐으면 기존 `RESUME_INPUT_LOCK_CHANGED`가 먼저 막아야 함(순서 유지).

**반례**: ① block run resume 후 plan check가 승인/정상 판정 도달(현재는 불가) ② resume 후에도 input-lock 변조 시 RESUME_INPUT_LOCK_CHANGED로 차단 ③ start 경로 회귀 없음.

## #21 (P1) — analysis_error 시 run 영구 소실

**증상**: proposal YAML 구문 오류 하나로 plan check가 analysis_error를 내면 `.plan-active`가 소비되는데, `resume.py`의 `resume_proposal_run`이 `result["verdict"] != "block"`이면 `RUN_NOT_PROPOSAL_REVISABLE`("only a proposal validation block can resume")로 거부(`resume.py`의 verdict 체크). → 구문 오류(사실 변경 아님)로 run 전체 폐기. 실제로 feature run2가 이렇게 폐기됨.

**요구**: 구문 오류처럼 **입력 사실 불변인데 proposal 파싱만 실패한 analysis_error**를 resume 허용 대상에 포함할지 검토·구현. 최소안: analysis_error 중 원인 코드가 "proposal 파싱/구문" 계열이면 block과 동일하게 resume 허용(사실 파일 불변 검증은 그대로 통과해야 함). 그 외 analysis_error(INPUT_UNREADABLE 등 입력 자체 손상)는 기존대로 거부.
- **완화 금지**: `_verify_immutable_files`(input-lock·facts 불변) 검증은 어떤 경우에도 유지. 입력이 바뀐 analysis_error는 resume 금지.

**반례**: ① proposal 구문오류 analysis_error → resume 허용 → 고친 proposal로 재검증 가능 ② 입력 손상 analysis_error → 여전히 거부 ③ 정상 block resume 회귀 없음.

## 결과서
`108_Codex_plan-resume_결과_20260903.md`: 코드 변경 위치(파일:줄)·반례별 결과·전체 테스트 집계. #20이 최우선. 커밋·push 금지.

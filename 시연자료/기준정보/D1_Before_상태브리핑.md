> 수집: 2026-08-26, 서버 세션 보고 원문. 검증자 교차확인 완료(README 참조).

업그레이드 직전 상태 브리핑 (1.13.1 커스텀 → 1.13.2)
1. 등록된 커스터마이징
쉬운 설명: 우리가 공식 OpenMetadata에 손댄 기능 7개를, 각각 "어느 커밋으로 만들었고 / 어느 파일을 건드렸고 / 무엇이 깨지면 안 되고 / 다른 기능과 파일을 겹쳐 쓰는지" 순으로 펼친 목록입니다.

BANK-OM-001 기준코드(InstanceCode) — 중요도 high, 담당 데이터시스템부

├─ 커밋: 23ec5bf00e "restore InstanceCode customization"
├─ 변경 경로: 48개 (단독 16 / 공유 32)
│    bootstrap/sql/migrations/native/1.13.1/mysql/schemaChanges.sql
│    openmetadata-service/src/main/java/org/openmetadata/service/Entity.java
│    openmetadata-spec/src/main/resources/json/schema/entity/data/instanceCode.json
├─ 계약: CONTRACT-INSTANCE-CODE — codeGroup/codeValue가 CRUD와 재색인 뒤에도 동일하게 조회된다
│    필수 테스트: tests/bank/contracts/test_instance_code.py::test_crud_search_roundtrip
└─ 공유 관계: 공유 32경로 (BANK-OM-002 / 003 / 004와)

BANK-OM-002 쿼리 리포트(QueryReport) — high

├─ 커밋: c899f5b014 "restore QueryReport customization"
├─ 변경 경로: 55개 (단독 23 / 공유 32)
│    bootstrap/sql/migrations/native/1.13.1/postgres/schemaChanges.sql
│    openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/CollectionDAO.java
│    openmetadata-spec/src/main/resources/json/schema/entity/data/queryReport.json
├─ 계약: CONTRACT-QUERY-REPORT — 연결된 Query 목록이 생성·수정·삭제 및 재색인 뒤에도 보존된다
│    필수 테스트: tests/bank/contracts/test_query_report.py::test_query_usage_roundtrip
└─ 공유 관계: 공유 32경로 (BANK-OM-001 / 003 / 004와)

BANK-OM-003 데이터 검증 결과(Data Assertions) — high

├─ 커밋: 8cb6f00abc "restore Data Assertions customization"
├─ 변경 경로: 25개 (단독 4 / 공유 21)
│    openmetadata-ui/.../components/AppRouter/AuthenticatedAppRouter.tsx
│    openmetadata-ui/.../components/MyData/MyDataFailedAssertions/MyDataFailedAssertions.component.tsx
│    openmetadata-ui/.../constants/constants.ts
├─ 계약: CONTRACT-DATA-ASSERTIONS — 실패한 test case가 상태·소유자·테이블·컬럼과 함께 일관되게 노출된다
│    필수 테스트: tests/bank/contracts/test_data_assertions.py::test_failed_assertion_projection
│                tests/bank/contracts/test_data_assertions.py::test_failed_assertion_rendered_row
└─ 공유 관계: 공유 21경로 (BANK-OM-001 / 002 / 004와) — 단독 소유가 4개뿐인 가장 의존적인 항목

BANK-OM-004 은행 컬럼 확장 표시 — high

├─ 커밋: 8a376508ed "restore bank column display customization"
├─ 변경 경로: 33개 (단독 14 / 공유 19)
│    openmetadata-ui/.../components/DataAssetSummaryPanelV1/DataAssetSummaryPanelV1.tsx
│    openmetadata-ui/.../components/Database/SchemaTable/SchemaTable.component.tsx
│    openmetadata-ui/.../components/Explore/ExploreTree/ExploreTree.tsx
├─ 계약: CONTRACT-BANK-COLUMN-VIEW — 컬럼 순번·키·제약조건 등 행내 확장 정보가 스키마와 검색 화면에서 동일하게 표시된다
│    필수 테스트: tests/bank/contracts/test_bank_columns.py::test_extended_column_projection
│                tests/bank/contracts/test_bank_columns.py::test_extended_column_rendered_row
└─ 공유 관계: 공유 19경로 (BANK-OM-001 / 002 / 003과)

BANK-OM-005 한글 입력 조합 보정 — medium (유일한 medium)

├─ 커밋: a5c8b9d098 "restore Korean IME customization"
│        1cf40ff403 "preserve Korean IME composition in controlled editor"
├─ 변경 경로: 2개 (단독 2 / 공유 0)
│    openmetadata-ui/.../components/Database/SchemaEditor/SchemaEditor.tsx
│    openmetadata-ui/.../components/Database/SchemaEditor/SchemaEditor.test.tsx
├─ 계약: CONTRACT-KOREAN-IME — CodeMirror 입력 중 조합 중인 한글 자모가 중복·역전·소실되지 않는다
│    필수 테스트: tests/bank/contracts/test_korean_ime.py::test_hangul_composition_roundtrip
└─ 공유 관계: 단독 소유 (shared-path-owners.yaml에 등재 0건)

BANK-OM-006 Sybase 연결 유형 — high

├─ 커밋: d430b33ae8 "restore Sybase customization"
├─ 변경 경로: 18개 (단독 11 / 공유 7)
│    openmetadata-spec/.../connections/database/sybaseConnection.json
│    openmetadata-spec/.../entity/services/databaseService.json
│    openmetadata-ui/.../assets/svg/service-icon-sybase.svg
├─ 계약: CONTRACT-SYBASE-CONNECTOR — Sybase 연결 스키마가 생성 UI·API·serviceConnection union에서 동일하게 유지된다
│    필수 테스트: tests/bank/contracts/test_sybase.py::test_connection_schema_roundtrip
└─ 공유 관계: 공유 7경로 (BANK-OM-007과만)

BANK-OM-007 Tibero 연결 유형 — high

├─ 커밋: a2393b5f07 "restore Tibero customization"
│        8ac18ad053 "complete Tibero service connection coverage"  ← 현재 product HEAD
├─ 변경 경로: 10개 (단독 3 / 공유 7)
│    openmetadata-spec/.../connections/database/tiberoConnection.json
│    openmetadata-spec/.../entity/services/databaseService.json
│    openmetadata-ui/.../assets/svg/service-icon-tibero.svg
├─ 계약: CONTRACT-TIBERO-CONNECTOR — Tibero 연결 스키마가 생성 UI·API·serviceConnection union에서 동일하게 유지된다
│    필수 테스트: tests/bank/contracts/test_tibero.py::test_connection_schema_roundtrip
└─ 공유 관계: 공유 7경로 (BANK-OM-006과만)
출처: customization-registry.yaml · commit-inventory.yaml · manifests/ · contracts.yaml · shared-path-owners.yaml

2. 장부 파일 지도
쉬운 설명: 이 폴더가 "무엇을 커스터마이징했는지"에 대한 유일한 공식 기록이고, 아래는 파일마다 어떤 역할을 맡는지입니다.

harness/registrations/om-temp-1.13.1/
├─ README.md                          (258B)  등록 워크스페이스 유래·미승인 상태 안내
├─ customization-registry.yaml        (2.5K)  ID 7개의 명단 — 제목·담당·중요도·매니페스트·계약 연결
├─ manifests/
│   ├─ BANK-OM-001.yaml               (6.8K)  ID별 변경 경로·업그레이드 감시 경로·보증 계약
│   ├─ BANK-OM-002.yaml               (7.8K)      〃
│   ├─ BANK-OM-003.yaml               (4.1K)      〃
│   ├─ BANK-OM-004.yaml               (5.6K)      〃
│   ├─ BANK-OM-005.yaml               (796B)      〃
│   ├─ BANK-OM-006.yaml               (3.5K)      〃
│   └─ BANK-OM-007.yaml               (1.9K)      〃
├─ contracts.yaml                     (2.5K)  계약 7개 — 불변식 문장과 필수 테스트 selector
├─ commit-inventory.yaml              (44K)   ID→커밋(sha·제목·변경경로) 원장, 기준 ref 2개 기록
├─ shared-path-owners.yaml            (4.6K)  현재 기준 "한 파일을 여러 ID가 함께 쓴다" 소유 지도
├─ source-snapshot-path-owners.yaml   (12K)   최초 스냅샷 시점의 경로→소유 ID 지도(전체 경로)
├─ shared-code-definitions.yaml       (121K)  공유 파일 안에서 각 ID가 책임지는 코드 조각 단언
├─ source-diff-paths.txt              (9.2K)  스냅샷 시점 공식 대비 변경 경로 목록 (111줄)
├─ current-diff-paths.txt             (9.3K)  현재 HEAD 기준 변경 경로 목록 (112줄)
├─ repository-layout.yaml             (740B)  경로 문법(NFC·대소문자 구분·심볼릭링크 거부)과 소유 루트 정의
└─ propose_shared_code_definitions.sh (999B)  shared-code-definitions 재생성 보조 스크립트
전체 집계: ID 7개 / 총 등록 경로 112개(매니페스트 합집합) / 공유 경로 39개 / 계약 7개 · 필수 테스트 9개(파일 7개) / 공유코드 단언 정의 114건·assertion 790개(37개 파일)

경로 수 두 값에 대한 주석: registry의 changed_path_count: 111과 source-diff-paths.txt(111줄)는 스냅샷 시점 값이고, 매니페스트 합집합·current-diff-paths.txt는 112입니다. 차이 1건은 SchemaEditor.test.tsx 하나로, BANK-OM-005의 두 번째 커밋 1cf40ff403에서 추가됐습니다. 불일치가 아니라 시점 차이입니다.

출처: harness/registrations/om-temp-1.13.1/ (위 전 파일 실측)

3. 저장소·기준점 상태
쉬운 설명: 계획을 시작하려면 두 저장소가 모두 "손댄 것 없이 깨끗"해야 하고, 비교 기준이 될 커밋 3개가 확정돼 있어야 합니다.

저장소 / 현재 브랜치 / HEAD / clean
제품 /home/jovyan/datacatalog-upgrade/product | upgrade/1.13.2-run-20260825 | 8ac18ad053 | ✅ (porcelain 0줄)
검사기 …/kb-datacatalog-upgrade-checker | upgrade/1.13.2-registration-20260825 | eec02540a3 | ✅ (porcelain 0줄)

기준점 / sha(10자) / 의미
custom_baseline | 8ac18ad053 | 현재 은행 커스텀 최신 — "complete Tibero service connection coverage"
official_base | afcb2d2cd7 | 공식 1.13.1 출발점 — "chore(release): Prepare Branch for 1.13.1"
official_target | 2763bf97ce | 공식 1.13.2 도착점 — "chore: update images and version to 1.13.2 in Dockerfiles and docker-compose files"
작업 브랜치 ① | — | 제품: upgrade/1.13.2-run-20260825
작업 브랜치 ② | — | 검사기: upgrade/1.13.2-registration-20260825
official_base는 장부 commit-inventory.yaml의 range.patch_sha 및 customization-registry.yaml의 source.upstream_sha와 같은 값입니다. custom_baseline은 range.custom_head_sha와 같은 값입니다.

출처: git rev-parse / git status --porcelain 실측 · commit-inventory.yaml · customization-registry.yaml

4. 이번 run 입력 요약
쉬운 설명: 계획 실행에 넘길 요청서에 실제로 뭐가 적혀 있는지, 값만 뽑은 것입니다.

mode | upgrade (hop_policy adjacent_only)
버전 | 1.13.1 → 1.13.2
배포방식 | docker-compose
공식 문서 | official-docs/openmetadata-1.13.2-release-notes.md (23,566B)
문서 sha256 앞 12자 | 35bff89e7d28 (64자 전체 기대값 일치 확인)
version_token | "1.13.2" — 원문 4회 출현, deployment_methods: [all]
registration_path | harness/registrations/om-temp-1.13.1
owner | seop
출처: /home/jovyan/datacatalog-upgrade/run-requests/upgrade-1.13.1-to-1.13.2-20260825.yaml · /home/jovyan/datacatalog-upgrade/official-docs/openmetadata-1.13.2-release-notes.md (sha256 실측)

5. 지금 상태 한 줄 결론
장부 기준 7개 기능이 등록돼 있고(경로 112·공유 39·계약 7·필수 테스트 9), 계획 단계 시작 조건(스키마 SCHEMA OK · 두 저장소 clean · 기준점 3개 확정 · 공식 문서 sha256·토큰 검증)이 모두 충족된 상태입니다.

한 가지만 덧붙입니다 — 차단 사유는 아니지만 기록해 둘 값입니다. customization-registry.yaml의 limitations에 "담당자는 최초 등록 승인 전에 실제 운영 담당자로 교체해야 합니다", "업무 정상 조건은 1.13.0 등록자료를 참고한 초안이므로 담당자 검토가 필요합니다" 두 항목이 남아 있고, 명단 7건의 owner는 모두 부서명 데이터시스템부(owner_status: assigned)입니다. 게이트에는 걸리지 않습니다.

출처: customization-registry.yaml · 3·4절의 실측값

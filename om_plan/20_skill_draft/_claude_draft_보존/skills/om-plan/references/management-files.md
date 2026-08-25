# 관리 파일 필드별 변경 조건

등록(관리) 파일 각 필드가 **언제·누구에 의해 바뀌는지**를 정한다. 목적: 후보(proposal) 단계에서 LLM이 **제안할 것과 손대지 말 것**을 구분한다.

> 이 문서는 2026-08-20 실구현(`registration_prep.py::apply_plan`, `initial_registration.py`) 대조로 정정됐다. 이전 판은 "status/provenance가 승인 시 자동 갱신"이라 적었으나 **그런 전이는 설계·구현에 없다.**

## 네 가지 주체 구분

- **[제안]** — 후보 단계에서 LLM이 관찰·근거로 제안한다(evidence_refs 필수). 지어내지 않는다.
- **[사람]** — 사람만 정한다. LLM이 채우면 검사기가 거부하거나 사람이 확정해야 한다.
- **[apply갱신]** — 승인 반영(apply) 시 시스템이 실제로 다시 쓴다. **후보·사람이 손대지 않는다.**
- **[전이없음]** — 최초 등록(bootstrap) 때 정해지고 **이후 아무 절차도 자동으로 바꾸지 않는다.** 바꾸려면 사람 편집 또는 (upgrade의 경우) 새 버전 bootstrap이 필요하다. 일부는 설계 미규정 갭이다.

## apply가 실제로 다시 쓰는 것 (실측)

`apply_plan`이 갱신하는 파일: 변경된 `manifests/*`(changed_paths·upgrade_watch.paths·schema_version·series.allowed), 신규 ID일 때만 `customization-registry.yaml`에 **entry append**, `commit-inventory.yaml`, `current-diff-paths.txt`. **그 외(특히 `source:` 블록과 기존 entry의 status/owner_status/provenance)는 건드리지 않는다.**

## customization-registry.yaml

### source 블록

| 필드 | 언제 바뀌나 | 주체 |
|---|---|---|
| `schema_version` | 스키마 구조 개정 시만 | [전이없음] |
| `source.repository`·`upstream_repository` | 저장소 이전 시(드묾) | [사람] |
| `source.snapshot_sha` | **bootstrap(최초/새버전)에서만** custom_sha로 설정. apply는 안 바꿈 | [전이없음] |
| `source.upstream_sha`·`upstream_tag` | **bootstrap에서만** official_sha로 설정. apply는 안 바꿈 → upgrade 반영 경로 미규정(갭 A4) | [전이없음] |
| `source.changed_path_count` | bootstrap이 계산. apply는 `current-diff-paths.txt`만 다시 쓰고 이 정수는 안 고침(어긋남 가능, 갭 A5) | [전이없음] |
| `source.ancestry_preserved` | bootstrap이 판정. apply 불변 | [전이없음] |
| `source.unregistered_findings`·`limitations` | 사람 검토 항목 | [사람] |

### entries[] (커스터마이징별)

| 필드 | 언제 바뀌나 | 주체 |
|---|---|---|
| `customization_id` | 최초 등록 1회. 이후 불변 | [제안]→확정 |
| `title` | 등록 시 커밋 기반 초안. 목적 재정의 시 | [제안](초안)/[사람](확정) |
| `owner` | 배정·재배정 시 | [사람] |
| `owner_status` | 사람이 배정 시 assigned. **apply 자동 전이 없음** | [사람] |
| `status` | 신규 entry는 apply가 곧바로 `active`로 기록. **"proposed→active" 전이 개념 없음**(제안은 registration 밖 proposal/에만 존재) | [apply갱신](신규)/[전이없음](기존) |
| `criticality` | 사람 평가·재평가 | [사람] |
| `manifest` | 등록 시 경로 설정. 불변 | [전이없음] |
| `contracts` | 계약 추가/삭제 시 | [제안]/[사람](확정) |
| `provenance` | 최초=`source-snapshot`, 신규 ID=`candidate-follow-up`. **이후 change/upgrade를 거쳐도 영구 불변**(upgrade 반영 표시 값·전이 설계에 없음) | [전이없음] |

## manifests/&lt;ID&gt;.yaml

| 필드 | 언제 바뀌나 | 주체 |
|---|---|---|
| `schema_version` | 스키마 개정 시만 | [전이없음] |
| `customization_id` | 불변 | — |
| `status` | 신규는 apply가 `active`로 기록. 기존은 보존 | [apply갱신](신규)/[전이없음](기존) |
| `kind` | 등록 시 설정. 구현 방식 바뀌면 | [제안](초안)/[사람] |
| `title` | registry title과 동기화 | (registry 따름) |
| `implementation.changed_paths` | change 반영 시 apply가 delta를 실제로 다시 씀 | [제안](관찰)→[apply갱신] |

## 기타 관리 파일

| 파일 / 항목 | 언제 바뀌나 | 주체 |
|---|---|---|
| `contracts.yaml` | 계약 추가/변경/삭제 | [제안]/[사람] |
| `shared-code-definitions.yaml` | 공유 코드 정의 변경(upgrade delta) | [제안]/[사람] |
| `shared-path-owners.yaml` | 최초 등록 시 사람 확정 입력. **apply 자동 갱신 없음**(change가 새로 도입한 공유 경로는 미반영) | [사람] |
| `source-snapshot-path-owners.yaml` | bootstrap(스냅샷) 시. 이후 불변 | [전이없음] |
| `sensitive-zones.yaml` | 민감영역 정의·기준 변경 | [사람] |
| `repository-layout.yaml` | 저장소 구조 규칙 변경(드묾) | [사람] |
| `commit-inventory.yaml`·diff 목록 | apply/재수집 시 다시 씀 | [apply갱신] |

## 후보 단계에서 LLM이 지켜야 할 것 (요약)

- **제안한다**: `changed_paths`(관찰), `title`·`kind`·`contracts` 초안, `decisions`/`unresolved_questions`.
- **손대지 않는다(사람 몫)**: `owner`, `owner_status`, `criticality`, `limitations`, `title` 확정, `shared-path-owners`.
- **손대지 않는다(apply/시스템)**: 신규 entry의 `status`, `manifest.changed_paths` 반영, `commit-inventory`, diff 목록.
- **바뀌지 않는다고 가정**: `source:` 블록 전체, `provenance`. change/upgrade를 거쳐도 자동으로 안 바뀐다.

## 미규정 갭 (Codex/사람 결정 대기 — 22 감사 참조)

- upgrade 반영 시 `source.upstream_sha/tag/snapshot_sha`를 누가 갱신하는가(apply? 새 버전 bootstrap?) — **미규정(갭 A4)**.
- `source.changed_path_count`가 `current-diff-paths.txt`와 어긋날 수 있음 — 교차검증 없음(갭 A5).
- change가 새로 도입한 공유 경로가 `shared-path-owners`에 반영 안 됨.

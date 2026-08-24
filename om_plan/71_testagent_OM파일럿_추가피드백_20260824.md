# [다른 세션용 추가 지시서] test-agent — OpenMetadata 파일럿 실측 발견 3건

작성일: 2026-08-24. 출처: 실제 OpenMetadata 1.13.2(lab, localhost:8585)에 붙여본 파일럿. 앞선 온보딩 피드백(70)과 별개의 **실전 통합 이슈**다.

## 발견 1 — [핵심·수정 필요] 인증 세션이 시나리오·크롤에 이어지지 않음

- **증상**: auth steps는 성공(로그인 폼 사라짐 단언 통과, "인증 완료" 표시)하는데, 이후 시나리오가 `/`에 가면 **다시 로그인 화면**. discover 크롤도 로그인 화면만 봄. README의 "저장된 세션을 모든 시나리오와 크롤링이 재사용"과 불일치.
- **원인(실측 확정)**: OpenMetadata는 인증 토큰을 **IndexedDB(`AppDataStore`)에 저장**한다(localStorage에는 없음 — 실측: localStorage 키 `__anon_id`,`loggedInUsers`뿐). Playwright의 기본 `storage_state()`는 **쿠키+localStorage만** 담고 IndexedDB를 안 담아서, 복원된 컨텍스트에 토큰이 없어 로그아웃 상태가 된다.
- **실증된 해결책**: Playwright 1.56(현재 사용 버전)의 `context.storage_state(indexed_db=True)`로 저장 → 그 상태로 `new_context(storage_state=...)` → **로그인 유지 성공**(`/my-data` 접근, signin 리다이렉트 없음)을 아래 재현으로 확인했다.
  ```python
  state = ctx.storage_state(indexed_db=True)   # ← 이 옵션이 핵심
  ctx2 = browser.new_context(storage_state=state)
  # ctx2에서 http://localhost:8585/my-data 접근 시 로그인 유지됨 (실측)
  ```
- **수정 제안**: auth 상태 저장·복원 경로에 `indexed_db=True`를 적용(기본 또는 `auth.indexed_db: true` 옵션). IndexedDB에는 토큰이 들어가므로 기존의 "인증 상태 파일 = 비밀번호 취급·자동 삭제" 정책을 그대로 적용할 것.

## 발견 2 — configs/openmetadata.yaml의 로그인 셀렉터 정정

- `fill [data-testid="email"]` → **실패**: `Element is not an <input>...` (그 testid는 입력이 아니라 래퍼).
- 실측 입력 셀렉터: **`#email` / `#password`** (discover 인벤토리로 확인). `wait_for [data-testid="login-form-container"]`·`click [data-testid="login"]`은 정상 동작.
- 수정: 초안 YAML의 fill 두 줄을 `#email`/`#password`로.

## 발견 3 — [마찰] discover만 돌려도 무관한 환경변수 요구

- `discover` 실행 시 data_checks의 `${OM_JWT}`가 미설정이면 **설정 오류로 전체 차단** — discover는 data_checks를 실행하지 않는데도.
- fail-closed 자체는 철학에 맞으나, 첫 발디딤(discover)이 여기 막히면 셀렉터를 얻을 수 없어 진행이 잠긴다. 검토 제안: discover는 미사용 값의 치환 실패를 경고로 완화하거나, 문서에 "discover도 모든 env 필요"를 명시.

## 잘 된 것 (참고 — 건드리지 말 것)

- **React SPA 크롤 인벤토리 정확**: 로그인 화면에서 `#email`/`#password`/버튼 6개를 정확히 수집.
- auth steps 실행·판정 자체는 정상 (assert_not_visible로 로그인 성공 감지).
- 정형 출력(report.json `schema_version`·`summary`·시나리오별 status/reasons, JUnit XML, 종료코드) 실측 확인 — 외부 파이프라인에서 통제 가능.

## 수정 후 검증법

OM lab(1.13.2, admin@open-metadata.org)에서:
1. auth 후 spec_check `{page: "/", assert_not_visible: login-form-container}` 가 **통과**해야 함 (현재는 실패).
2. discover가 로그인 후 화면들(예: /my-data)을 크롤하는지.

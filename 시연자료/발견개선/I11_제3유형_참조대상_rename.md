> 수집: 2026-09-01. 유형: 발견(P1 — 진짜 업그레이드 결함 첫 채집 + 탐지 공백 제3유형).
결과 한 줄: verify 선행 빌드에서 `Could not resolve "../utils/StringsUtils"` — 공식 1.13.2가 StringsUtils.ts→StringUtils.ts로 rename(R084)했는데 은행 전용 파일 2개(instanceCodeAPI·queryReportAPI)가 옛 이름을 import. 필요한 심볼(getEncodedFqn)은 새 파일에 실재 — 교정은 import 두 줄.
왜 제3유형인가: ①경로 삭제(존재 대조로 탐지)·②내용 이탈(심볼 스캔으로 탐지)과 달리, ③은 "은행이 안 건드린 순수 공식 파일의 rename"이라 층1 교집합(공식변경∩등록경로)에 안 들어오고, 내용 이탈 스캔은 은행 파일 내부만 보며, apply check는 import 해석을 안 봄. 정적 정합 완벽 + 빌드 불가 상태 — 발견 시점이 ①②(apply 중단)보다 늦은 빌드 단계까지 밀림.
규율 작동: 세션이 "두 줄만 고치고 진행"(B)을 planned_actions 경계 위반 + candidate 결속 파괴로 스스로 배제하고 정지·보고. 코드 무수정(source_clean 유지).
처방(수렴): 계획 단계에 "은행 등록 TS/TSX 전 경로의 상대 import가 최종 트리에서 해석되는가" 전수 스캔 추가 — 이 스캔이면 ①②③ 모두 계획 단계에서 잡힘. 이번 스캔 스크립트가 58경로 전수 검사로 2건을 정확히 특정(실증).

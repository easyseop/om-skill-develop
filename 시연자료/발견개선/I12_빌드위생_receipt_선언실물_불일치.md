> 수집: 2026-09-02. 유형: 발견(P1 — 빌드 단계 무결성 공백) + 세션 자기적발.
결과 한 줄: compose 기동 후 세션이 스스로 실측 — service jar에 은행 백엔드 클래스 0건(/v1/instanceCodes 404). 원인은 이식이 아니라 **빌드 위생**: 여러 리셋을 거치며 오래된 backend target/을 "존재하니 재사용 가능"으로 취급(-rf로 UI만 재빌드), 산출물이 어느 트리에서 나왔는지는 미확인. 결과적으로 build receipt의 dist_source_tree_sha를 **선언만 하고 실물 대조 없이** 기재 — receipt·이미지·compose 무효 판정(자진). apply-result·커밋 8건은 유효(소스·장부 정상).
의미: apply check pass = 소스·장부 정합이지, 빌드 산출물이 그 소스에서 나왔다는 보장이 아님. 검사기의 dist_source_tree_sha==candidate_tree 규칙이 정확히 이걸 막는 설계인데, local-issued receipt는 발행자의 정직에 의존 — 이번에 그 한계가 실증됨(om-verify 남은위험 "local-issued 한계"의 실물 사례).
조치: 전체 재빌드 + receipt 발행 전 실물 검증 단계 신설(jar 내 은행 클래스 실재 + UI dist 포함 대조, 기록 보존).

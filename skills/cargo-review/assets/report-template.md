# Code Review Report

## Summary
- 변경 파일: N개 (추가 N / 수정 N / 삭제 N)
- 테스트: PASS/FAIL (N개 통과, N개 실패)
- Clippy: PASS/WARN (N개 경고)
- 테스트 커버리지: N/M 파일
- 리뷰 모드: Single / Multi-Agent (N reviewers)
- 포커스: Full / Refactor

---

## 1. Correctness (정확성)

변경된 코드의 논리적 정확성을 검토한다.

| # | 파일:라인 | 심각도 | 이슈 | 제안 |
|---|----------|--------|------|------|
| 1 | path:L42 | HIGH/MED/LOW | 설명 | 수정안 |

검토 항목:
- 로직 오류, off-by-one, 경계 조건
- unwrap()/expect() 사용 — 패닉 가능성
- 에러 핸들링 누락 (Result/Option 미처리)
- unsafe 블록의 안전성
- 동시성 이슈 (데이터 레이스, 데드락)
- 리소스 누수 (미해제 핸들, 미닫힘 연결)

이슈 없으면: "정확성 이슈 없음"

---

## 2. Style (스타일)

Rust 컨벤션 및 프로젝트 일관성을 검토한다.

| # | 파일:라인 | 심각도 | 이슈 | 제안 |
|---|----------|--------|------|------|
| 1 | path:L42 | MED/LOW | 설명 | 수정안 |

검토 항목:
- 네이밍 컨벤션 (snake_case, CamelCase)
- clippy 경고
- 불필요한 clone/collect
- 과도한 중첩 (3단계 이상)
- 미사용 import/변수
- 문서 주석(doc comment) 누락 (pub 항목)

이슈 없으면: "스타일 이슈 없음"

---

## 3. Suggestions (제안)

코드 개선 기회를 제안한다. 필수 수정이 아닌 선택적 개선.

| # | 파일:라인 | 유형 | 제안 |
|---|----------|------|------|
| 1 | path:L42 | PERF/READABILITY/IDIOM/SECURITY | 설명 |

제안 유형:
- PERF: 성능 개선 (불필요한 할당, 더 나은 자료구조)
- READABILITY: 가독성 개선 (복잡한 표현식 분리, 변수명)
- IDIOM: Rust 관용구 적용 (if let, match, iterator chain)
- SECURITY: 보안 강화 (입력 검증, 권한 확인)

제안 없으면: "추가 제안 없음"

Refactor 포커스인 경우: "리팩토링 모드 — 생략"

---

## Verdict (판정)

| 항목 | 상태 |
|------|------|
| 테스트 통과 | YES/NO |
| Clippy 통과 | YES/NO |
| HIGH 이슈 | N개 |
| MED 이슈 | N개 |
| 커버리지 갭 | N개 파일 |
| **종합 판정** | APPROVE / REQUEST CHANGES / NEEDS DISCUSSION |

판정 기준:
- APPROVE: HIGH 0개, 테스트 통과, clippy 통과
- REQUEST CHANGES: HIGH 1개 이상 또는 테스트 실패
- NEEDS DISCUSSION: 아키텍처 변경 또는 트레이드오프 판단 필요

# Cargo Review — 사용 시나리오 예시

> SKILL.md에서 사용 예시가 필요할 때 이 파일을 참조한다.

---

## 시나리오 1: 커밋 전 셀프 리뷰 (가장 일반적)

### 상황
새 함수를 추가하고 커밋하기 전에 빠르게 검증하고 싶다.

### 입력
```
/cargo-review
```

### 동작
- diff 40줄 → **Single Mode, Full 포커스**
- `cargo test` 실행 → 32개 통과
- `cargo clippy` 실행 → 경고 없음
- 직접 3개 축 리뷰

### 출력 요약
```
## Summary
- 변경 파일: 1개 (수정 1)
- 테스트: PASS (32개 통과)
- Clippy: PASS
- 리뷰 모드: Single
- 포커스: Full

## Verdict
| 종합 판정 | APPROVE |
```

---

## 시나리오 2: 대규모 기능 PR 전 리뷰

### 상황
3일간 작업한 기능 브랜치. 파일 8개, diff 350줄. PR 올리기 전에 꼼꼼히 검토하고 싶다.

### 입력
```
/cargo-review
```

### 동작
- diff 350줄 → **Multi-Agent Mode, Full 포커스**
- `cargo test --workspace` 실행 → 128개 통과
- 서브에이전트 3개 병렬 디스패치:
  - Agent A (Correctness): unwrap() 2건, 에러 핸들링 누락 1건 발견
  - Agent B (Style): clippy 경고 1건, 미사용 import 1건 발견
  - Agent C (Suggestions): iterator chain 적용 가능 2건 제안
- Step 5에서 결과 병합, 중복 없음

### 출력 요약
```
## Summary
- 변경 파일: 8개 (추가 3 / 수정 5)
- 테스트: PASS (128개 통과)
- Clippy: WARN (1개 경고)
- 리뷰 모드: Multi-Agent (3 reviewers)
- 포커스: Full

## 1. Correctness
| # | 파일:라인          | 심각도 | 이슈                               | 제안                |
|---|--------------------|--------|------------------------------------|--------------------|
| 1 | handler.rs:L45     | HIGH   | unwrap() on network response       | ? 연산자로 전파    |
| 2 | handler.rs:L78     | HIGH   | unwrap() on JSON parse             | match로 에러 처리  |
| 3 | service.rs:L112    | MED    | Result 무시 (let _ = send())       | 로그 후 에러 전파  |

## 2. Style
| # | 파일:라인          | 심각도 | 이슈                    | 제안                       |
|---|--------------------|--------|------------------------|---------------------------|
| 1 | handler.rs:L12     | LOW    | clippy: needless_borrow | &val → val                |
| 2 | models.rs:L3       | LOW    | 미사용 import (serde)  | 제거                       |

## 3. Suggestions
| # | 파일:라인          | 유형        | 제안                                      |
|---|--------------------|-------------|------------------------------------------|
| 1 | service.rs:L88     | IDIOM       | for loop → .iter().filter().map() chain  |
| 2 | service.rs:L95     | READABILITY | 중첩 if 3단계 → early return 패턴        |

## Verdict
| 종합 판정 | REQUEST CHANGES |
| HIGH 이슈 2개 해결 필요 |
```

---

## 시나리오 3: 리팩토링 검증

### 상황
에러 핸들링을 `anyhow`에서 커스텀 에러 타입으로 전환. 동작은 안 바뀌어야 한다.

### 입력
```
/cargo-review --refactor
```

### 동작
- diff 180줄 → **Multi-Agent Mode, Refactor 포커스**
- Agent A (Correctness) + Agent B (Style) 만 디스패치, Agent C 생략
- Agent A가 Refactor 추가 검토 항목 적용:
  - public API 시그니처 변경 여부 확인
  - 기존 테스트 전체 통과 확인
  - 의미론적 동등성 검증

### 출력 요약
```
## Summary
- 변경 파일: 6개 (수정 6)
- 테스트: PASS (95개 통과)
- Clippy: PASS
- 리뷰 모드: Multi-Agent (2 reviewers)
- 포커스: Refactor

## 1. Correctness
| # | 파일:라인        | 심각도 | 이슈                                    | 제안                    |
|---|-----------------|--------|-----------------------------------------|------------------------|
| 1 | lib.rs:L22      | HIGH   | pub fn process() 반환 타입 변경 (breaking) | 기존 Result<T, anyhow::Error> 유지 또는 From impl 추가 |

## 2. Style
이슈 없음

## 3. Suggestions
리팩토링 모드 — 생략

## Verdict
| 종합 판정 | REQUEST CHANGES |
| breaking change 1건 — 반환 타입 호환성 확인 필요 |
```

---

## 시나리오 4: Cargo workspace 프로젝트

### 상황
workspace에 crate 5개. `crate-parser`만 수정했는데 전체 빌드는 느리다.

### 입력
```
/cargo-review
```

### 동작
- Step 0: `Cargo.toml`에 `[workspace]` 감지 → workspace 모드
- Step 0: `git diff --name-only`로 변경 파일 → `crate-parser` crate에만 해당
- Step 3: `cargo test -p crate-parser` + `cargo clippy -p crate-parser` 실행 (전체 workspace 아님)
- diff 60줄 → Single Mode

### 출력 요약
```
## Summary
- 변경 파일: 2개 (수정 2) [crate: crate-parser]
- 테스트: PASS (18개 통과)
- Clippy: PASS
- 리뷰 모드: Single
- 포커스: Full

## Verdict
| 종합 판정 | APPROVE |
```

---

## 시나리오 5: CI/headless 자동 리뷰

### 상황
GitHub Actions에서 PR이 올라오면 자동으로 코드 리뷰를 실행하고 싶다.

### 입력 (GitHub Actions workflow)
```yaml
- name: Cargo Review
  run: |
    claude -p "Read and follow .claude/skills/cargo-review/SKILL.md to review the current diff" \
      --allowedTools "Bash,Read,Grep,Glob,Agent" \
      > review-report.md

- name: Post review comment
  run: gh pr comment ${{ github.event.pull_request.number }} --body-file review-report.md
```

### 동작
- 비대화형 실행 — 전체 파이프라인 자동 완료
- stdout으로 리포트 출력
- PR 코멘트로 리포트 게시

### 결과
PR에 구조화된 리뷰 리포트가 코멘트로 자동 게시된다.

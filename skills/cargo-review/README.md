# Cargo Review

Rust 프로젝트에서 `cargo test`와 `cargo clippy`를 돌리고, 변경된 코드의 diff를 읽어서 **Correctness / Style / Suggestions** 3개 축으로 구조화된 리뷰 리포트를 자동 생성하는 Claude Code 스킬입니다.

## 이런 상황에서 쓰세요

- 커밋 전에 "내가 뭘 놓쳤는지" 빠르게 확인하고 싶을 때
- PR 올리기 전에 셀프 리뷰를 자동화하고 싶을 때
- 리팩토링 후 동작이 안 깨졌는지 검증하고 싶을 때

## 작동 방식

```
환경 감지 → git diff → 테스트 커버리지 확인 → cargo test + clippy → 리뷰 리포트
```

코드를 변경한 뒤 `/cargo-review`만 입력하면, 파이프라인이 순차 실행되고 최종 리포트에 **APPROVE / REQUEST CHANGES / NEEDS DISCUSSION** 판정이 나옵니다.

diff가 100줄을 넘으면 서브에이전트 3개가 병렬로 각 축을 깊게 리뷰합니다.

## 사용법

```bash
# 기본 리뷰 (Full 모드)
/cargo-review

# 리팩토링 검증 (Suggestions 생략, 동작 보존에 집중)
/cargo-review --refactor
```

## 리포트 예시

```
# Code Review Report

## Summary
- 변경 파일: 3개 (추가 1 / 수정 2)
- 테스트: PASS (47개 통과)
- Clippy: WARN (1개 경고)
- 테스트 커버리지: 2/3 파일
- 리뷰 모드: Single
- 포커스: Full

## 1. Correctness (정확성)
| # | 파일:라인      | 심각도 | 이슈                          | 제안              |
|---|---------------|--------|-------------------------------|-------------------|
| 1 | parser.rs:L87 | HIGH   | unwrap() on user input Result | ? 연산자로 전파   |

## 2. Style (스타일)
| # | 파일:라인       | 심각도 | 이슈                | 제안                    |
|---|----------------|--------|---------------------|-------------------------|
| 1 | config.rs:L142 | LOW    | 불필요한 .clone()   | 참조로 전달             |

## 3. Suggestions (제안)
| # | 파일:라인      | 유형   | 제안                                    |
|---|---------------|--------|-----------------------------------------|
| 1 | parser.rs:L91 | IDIOM  | match 대신 if let Some(v) 사용          |

## Verdict (판정)
| 종합 판정 | REQUEST CHANGES |
| HIGH 이슈 1개 해결 후 재실행 권장 |
```

## 주요 기능

| 기능 | 설명 |
|------|------|
| Multi-Agent 리뷰 | diff 100줄 초과 시 Correctness/Style/Suggestions 전담 에이전트 3개 병렬 실행 |
| Refactor 모드 | 동작 보존 검증에 집중 — breaking change, 의미론적 동등성 검토 |
| Workspace 지원 | `[workspace]` 감지 시 변경된 crate만 대상으로 테스트 |
| 커맨드 오버라이드 | `.cargo-review.toml`로 test/clippy 명령, base branch 커스텀 |
| Base branch 자동 감지 | `origin/HEAD` → `main` → `master` fallback |
| CI/Headless | 비대화형 환경에서 자동 실행 가능 |

## 설정 (선택사항)

프로젝트 루트에 `.cargo-review.toml`을 생성하면 기본값을 오버라이드할 수 있습니다.

```toml
base_branch = "develop"
test_command = "cargo test --workspace --no-fail-fast"
clippy_command = "cargo clippy --workspace -- -D warnings"
```

## 요구사항

- Rust toolchain (`cargo test`, `cargo clippy`)
- Git 저장소

## 상세 사용 예시

시나리오별 상세 입출력 예시는 [references/examples.md](references/examples.md)를 참조하세요.

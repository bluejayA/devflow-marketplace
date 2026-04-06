---
name: cargo-review
description: |
  Rust 프로젝트의 현재 변경사항을 Correctness, Style, Suggestions 3개 축으로 구조화된 코드 리뷰 리포트를 생성한다. 대규모 변경 시 서브에이전트 병렬 리뷰, 리팩토링 모드, Cargo workspace를 지원한다.
  Use when reviewing Rust code changes before commit or PR. Keywords: cargo review, cargo-review, 리뷰해줘, 코드 리뷰, diff 검토, 변경사항 점검, PR 전 검토, Rust 리뷰, 코드 검토, 커밋 전 점검, 리팩토링 검증, refactor check.
metadata:
  version: "2.0.0"
  author: bluejayA
  category: code-review
  invoke_mode: user-invocable
  return_behavior: report-to-user
---

# Cargo Review

Pipeline 실행 전 사용자에게 다음을 출력한다: **"Cargo Review 스킬을 사용합니다."**

## Purpose

현재 브랜치의 변경사항을 수집하고, 테스트 커버리지와 cargo test/clippy 결과를 확인한 뒤, Correctness/Style/Suggestions 3개 섹션의 구조화된 리뷰 리포트를 생성한다.

**자유도: 저(Low)** — 단계 순서와 게이트 조건은 고정. 리뷰 내용 작성은 중자유도.

---

## Review Focus

사용자의 호출 방식에 따라 리뷰 포커스를 결정한다.

| 호출 패턴 | 포커스 | 동작 |
|-----------|--------|------|
| `/cargo-review` (기본) | **Full** | Correctness + Style + Suggestions 전체 수행 |
| `/cargo-review --refactor` 또는 "리팩토링 검증" | **Refactor** | Correctness 강화, Style 유지, Suggestions 생략 |

### Refactor 포커스 규칙

- **Correctness 강화**: 기존 항목 + public API 시그니처 변경(breaking change), 기존 테스트 통과 여부, 의미론적 동등성
- **Style 유지**: 리팩토링이라도 스타일 위반은 잡아야 함
- **Suggestions 생략**: 리팩토링 자체가 개선이므로 추가 제안은 노이즈
- **리포트에 `포커스: Refactor` 표기**

---

## Configuration

프로젝트 루트의 `.cargo-review.toml` 또는 `Cargo.toml`의 `[workspace]` 존재 여부로 설정을 감지한다. 설정 파일이 없으면 기본값을 사용한다.

### `.cargo-review.toml` (선택사항)

```toml
# 기본 브랜치 (생략 시 자동 감지)
base_branch = "develop"

# 테스트 커맨드 오버라이드 (생략 시 기본값 사용)
test_command = "cargo test --workspace --no-fail-fast"

# clippy 커맨드 오버라이드 (생략 시 기본값 사용)
clippy_command = "cargo clippy --workspace -- -D warnings"
```

### 기본값

| 설정 | 기본값 | 감지 방법 |
|------|--------|-----------|
| base_branch | 자동 감지 | `git rev-parse --abbrev-ref origin/HEAD` → fallback `main` → `master` |
| test_command | `cargo test` | workspace이면 `cargo test --workspace` |
| clippy_command | `cargo clippy -- -D warnings` | workspace이면 `cargo clippy --workspace -- -D warnings` |

---

## Pipeline

> **DO NOT skip any step.** 각 단계의 출력이 다음 단계의 입력이다.

### Step 0: 환경 감지

1. `.cargo-review.toml` 존재 시 로드하여 설정 오버라이드 적용
2. `Cargo.toml`에 `[workspace]` 존재 여부 확인 → workspace 모드 결정
3. base branch 결정:
   - `.cargo-review.toml`의 `base_branch` 있으면 사용
   - 없으면 `git rev-parse --abbrev-ref origin/HEAD` 시도
   - 실패 시 `main` → `master` 순서로 fallback
4. workspace 모드이면 변경된 crate 목록을 미리 수집 (`git diff --name-only` + `Cargo.toml` 매핑)

### Step 1: Diff 수집

1. `git diff HEAD` 실행 (unstaged + staged 변경)
2. diff가 비어있으면 `git diff --cached` 시도
3. 여전히 비어있으면 `git diff {base_branch}...HEAD` 시도 (브랜치 전체 변경)
4. 변경사항이 없으면 **"변경사항 없음 — 리뷰할 내용이 없습니다"** 출력 후 종료

수집할 정보:
- 변경된 파일 목록 (`git diff --stat`)
- 전체 diff 내용
- 변경된 `.rs` 파일 목록 (리뷰 대상)
- Non-Rust 파일 분류: `Cargo.toml`(의존성 검토 대상), 설정 파일(구문 오류만), `.md`(제외)
- **diff 총 라인 수 기록** — Step 4에서 리뷰 모드 결정에 사용
- workspace 모드이면 **변경된 crate 목록** 기록

> **DO NOT proceed to Step 2 until diff is collected.**

### Step 2: 테스트 커버리지 확인

변경된 각 `.rs` 파일에 대해:

1. `src/` 파일이면 대응하는 테스트 존재 여부 확인:
   - 같은 파일 내 `#[cfg(test)]` 모듈
   - `tests/` 디렉토리의 통합 테스트
   - 파일명 기반 매칭 (`foo.rs` → `test_foo.rs`, `foo_test.rs`)
2. 새로 추가된 `pub fn`/`pub async fn`에 테스트가 있는지 확인
3. 커버리지 요약 생성

> **DO NOT proceed to Step 3 until coverage check is complete.**

### Step 3: cargo test + clippy 실행

1. 테스트 커맨드 실행 (설정 또는 기본값):
   - 기본: `cargo test 2>&1`
   - workspace: `cargo test --workspace 2>&1` 또는 변경된 crate만 `cargo test -p {crate} 2>&1`
   - 오버라이드: `.cargo-review.toml`의 `test_command`
2. 결과 기록: 통과/실패 수, 실패한 테스트 목록, 컴파일 에러
3. clippy 커맨드 실행 (설정 또는 기본값):
   - 기본: `cargo clippy -- -D warnings 2>&1`
   - workspace: `cargo clippy --workspace -- -D warnings 2>&1`
   - 오버라이드: `.cargo-review.toml`의 `clippy_command`
4. clippy 경고/에러 목록 기록

컴파일 실패 시: 에러 메시지를 리포트의 Correctness 섹션 최상단에 HIGH로 기록하고, Step 4로 진행한다.

> **DO NOT proceed to Step 4 until tests and clippy complete.**

### Step 4: 리뷰 모드 결정 + 리뷰 실행

두 가지 축으로 리뷰 모드를 결정한다.

**축 1 — 에이전트 모드** (diff 크기 기준):

| diff 라인 수 | 모드 | 동작 |
|---|---|---|
| ≤ 100줄 | Single | 직접 리뷰 |
| > 100줄 | Multi-Agent | 서브에이전트 병렬 리뷰 |

**축 2 — 리뷰 포커스** (Review Focus 섹션에서 결정):

| | Single Mode | Multi-Agent Mode |
|---|---|---|
| **Full** | 직접 3개 축 리뷰 | 에이전트 3개 (A+B+C) |
| **Refactor** | 직접 2개 축 리뷰 | 에이전트 2개 (A+B, C 생략) |

#### Single Mode (diff ≤ 100줄)

Step 1-3의 결과를 종합하여 직접 리포트를 생성한다. Report Format 섹션의 형식을 따른다.
Refactor 포커스인 경우 Suggestions 섹션을 `"리팩토링 모드 — 생략"` 한 줄로 대체한다.

#### Multi-Agent Mode (diff > 100줄)

Agent tool을 사용하여 서브에이전트를 **동시에** 디스패치한다. 반드시 하나의 메시지에서 모든 Agent tool call을 병렬로 실행한다.

각 에이전트 프롬프트는 `references/agent-prompts.md`에 정의되어 있다. 해당 파일을 읽고 프롬프트를 구성한다.

공통 컨텍스트로 전달할 것:
- Step 1의 전체 diff 내용
- Step 2의 테스트 커버리지 요약
- Step 3의 cargo test/clippy 결과
- Non-Rust 파일 분류 정보
- Refactor 포커스 여부 (Agent A에 추가 검토 항목 적용)

> **DO NOT proceed to Step 5 until all agents have returned.**

### Step 5: 결과 종합 + Verdict (Multi-Agent Mode만)

에이전트 결과를 Report Format에 맞게 병합한다.

병합 규칙:
1. **중복 제거**: 같은 파일:라인을 2개 이상 지적한 경우, 더 높은 심각도를 채택
2. **심각도 재조정**: 에이전트 간 심각도가 다른 경우 상향 채택
3. **Verdict 산출**: 병합된 결과로 판정 기준 적용
4. **리뷰 모드 표기**: 리포트 Summary에 모드 기록

---

## Report Format (리포트 형식)

`assets/report-template.md`에 정의된 템플릿을 따른다.

핵심 구조:
1. **Summary** — 변경 파일 수, 테스트/clippy 결과, 커버리지, 리뷰 모드, 포커스
2. **Correctness (정확성)** — 로직 오류, 패닉 가능성, 에러 핸들링 등
3. **Style (스타일)** — 네이밍 컨벤션, clippy 경고, 불필요한 코드 등
4. **Suggestions (제안)** — PERF, READABILITY, IDIOM, SECURITY (Refactor 모드 시 생략)
5. **Verdict (판정)** — APPROVE / REQUEST CHANGES / NEEDS DISCUSSION

판정 기준:
- APPROVE: HIGH 0개, 테스트 통과, clippy 통과
- REQUEST CHANGES: HIGH 1개 이상 또는 테스트 실패
- NEEDS DISCUSSION: 아키텍처 변경 또는 트레이드오프 판단 필요

---

## CI / Headless Mode

GitHub Actions 등 비대화형 환경에서 사용할 때의 가이드.

```bash
# headless 실행 예시
claude -p "Read and follow .claude/skills/cargo-review/SKILL.md to review the current diff" \
  --allowedTools "Bash,Read,Grep,Glob,Agent"
```

headless 모드에서는:
- 사용자 상호작용 없이 전체 파이프라인을 자동 실행한다
- 리포트를 stdout으로 출력한다 (PR 코멘트로 파이프 가능)
- Multi-Agent 임계값(100줄)은 동일하게 적용한다

---

## Examples

사용 시나리오별 상세 입출력 예시는 `references/examples.md`를 참조한다.

---

## Troubleshooting

| 증상 | 원인 | 해결 |
|------|------|------|
| "변경사항 없음" 출력 | diff가 비어있음 | `git status`로 상태 확인 후 재실행 |
| cargo test 타임아웃 | 테스트가 너무 오래 걸림 | `.cargo-review.toml`에 `test_command = "cargo test -- --test-threads=1"` 설정 |
| clippy 실행 실패 | clippy 미설치 | `rustup component add clippy` 실행 |
| diff가 너무 커서 리뷰 불완전 | 변경량 500줄 초과 | `.rs` 파일만 대상으로 축소, 나머지는 요약 |
| 컴파일 에러로 테스트 불가 | 코드가 빌드되지 않음 | 에러를 Correctness HIGH로 기록 후 리포트 진행 |
| base branch 감지 실패 | remote HEAD 미설정 | `.cargo-review.toml`에 `base_branch = "main"` 명시 |
| workspace에서 전체 빌드가 느림 | 변경 안 된 crate도 테스트 | 변경된 crate만 `-p` 플래그로 테스트 (기본 동작) |
| 서브에이전트 간 중복 발견 | 같은 라인을 여러 관점에서 지적 | Step 5 병합 규칙에 따라 상위 심각도 채택 |
| Refactor 모드에서 Suggestions 필요 | 기본적으로 생략됨 | `/cargo-review` (포커스 없이)로 재실행 |

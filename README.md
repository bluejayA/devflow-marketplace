# devflow-marketplace

Central plugin registry for the devflow toolchain — install Claude Code plugins with a single command.

## Install

```bash
claude plugins install https://github.com/bluejayA/devflow-marketplace.git
```

This gives you access to all registered plugins and their skills.

## Registered Plugins

| Plugin | Version | Description |
|--------|---------|-------------|
| [aidlc](https://github.com/bluejayA/aidlc-devflow) | 1.8.0 | AI-DLC methodology development workflow (28 skills, 5 agents, 4-stage code review) |
| [reverse-engineering](https://github.com/bluejayA/devflow-reverse-engineering) | 0.4.0 | Brownfield codebase analysis (4-phase pipeline, 3 modes) |
| [skill-security-audit](https://github.com/bluejayA/skill-security-audit) | 2.0.0 | Skill security gatekeeper (35 rules, OWASP AST10) |

## Direct Skills

Standalone skills shipped directly in this repository (`skills/` directory):

| Skill | Version | Description |
|-------|---------|-------------|
| [cargo-review](skills/cargo-review/) | 2.0.0 | Rust code review (Correctness/Style/Suggestions 3-axis report, parallel subagent, refactoring mode) |

## Submitting a Plugin

### Option A: Direct Skill Submission

Add your skill directly to the `skills/` directory via PR:

```bash
# Fork this repo, then:
mkdir -p skills/my-skill
# Create skills/my-skill/SKILL.md with your skill definition
git add skills/my-skill
git commit -m "feat: add my-skill"
# Push and create PR
```

### Option B: Remote Plugin Registration

Add your plugin URL to `marketplace.json` via PR:

```json
{
  "name": "my-plugin",
  "source": {
    "source": "url",
    "url": "https://github.com/your-org/your-plugin.git"
  },
  "revision": "<full commit SHA>",
  "description": "Your plugin description",
  "version": "1.0.0",
  "strict": false
}
```

**Requirements:**
- `url` must be `https://github.com/` (other protocols are blocked)
- `revision` must be a full 40-character commit SHA
- Your plugin repo must have `skills/*/SKILL.md` structure

## Automated Security Audit

Every PR is automatically audited by [skill-security-audit](https://github.com/bluejayA/skill-security-audit):

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| **Skill Audit Gate** | All PRs | Reports audit scope, always passes |
| **Skill Audit: Direct** | `skills/**` changes | Audits skill files in the PR |
| **Skill Audit: Remote** | `marketplace.json` changes | Clones plugin repo, audits all skills |

### Verdicts

- **PASSED** — No issues, safe to merge
- **PASSED with warnings** — HIGH/MEDIUM findings, review recommended
- **BLOCKED** — CRITICAL findings, must fix before merge (check shows red X)

### Security Design

- **2-job isolation**: Scan job (read-only) and report job (write permissions) are separated
- **Base branch audit tool**: Audit tool is loaded from main branch, not PR (prevents tampering)
- **URL allowlist**: Only `https://github.com/` URLs accepted
- **Revision pinning**: Audits are reproducible via immutable commit SHA
- **Fail-Closed**: If the audit tool fails to run, the result is BLOCKED (never silent PASSED)

### Before submitting

Run the audit locally to catch issues early:

```bash
claude plugins install https://github.com/bluejayA/skill-security-audit.git
claude "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘"
```

See the [Local Verification Guide](https://github.com/bluejayA/skill-security-audit/blob/main/docs/local-verification-guide.md) and [CI Integration Guide](https://github.com/bluejayA/skill-security-audit/blob/main/docs/ci-integration-guide.md) for details.

## CI Test Results

End-to-end verification performed on 2026-04-02:

| Test | Scenario | Result |
|------|----------|--------|
| Gate Only | PR with no audit targets | **PASS** — Gate success, Direct/Remote skipped |
| Direct Clean | Safe skill submission | **PASS** — PASSED verdict, PR comment posted |
| Direct Dangerous | Malicious skill (4 CRITICAL findings) | **PASS** — BLOCKED, check failure |
| Remote Plugin | marketplace.json revision change | **PASS** — External repo cloned and audited |
| URL Allowlist | `file:///etc/passwd` blocked | **PASS** — Rejected in 5 seconds |
| Fail-Closed | Missing API key | **PASS** — BLOCKED (not silent PASSED) |

---

## 한국어 요약

devflow 도구 모음의 중앙 플러그인 레지스트리입니다.

```bash
claude plugins install https://github.com/bluejayA/devflow-marketplace.git
```

### 플러그인 제출 방법
- **Direct**: `skills/` 디렉토리에 직접 스킬 추가 PR
- **Remote**: `marketplace.json`에 플러그인 URL 등록 PR

### 자동 보안 감사
모든 PR은 [skill-security-audit](https://github.com/bluejayA/skill-security-audit)으로 자동 감사됩니다. CRITICAL 발견 시 머지가 차단됩니다.

제출 전 로컬 검증을 권장합니다: [로컬 검증 가이드](https://github.com/bluejayA/skill-security-audit/blob/main/docs/local-verification-guide.md)

## License

MIT

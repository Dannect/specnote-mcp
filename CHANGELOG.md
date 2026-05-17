# Changelog

## 0.2.0 — 2026-05-18

### BREAKING
- env 박는 토큰 방식 (SPECNOTE_TOKEN) 제거. OAuth 2.1 자동 인증으로 전환.

### Added
- MCP 2025-06-18 spec 준수 (RFC 9728 + 8414 + 7591 + 7636)
- 첫 호출 시 browser 자동 OAuth (사용자 별도 토큰 발급 불필요)
- 토큰 자동 회전 (30일간 재로그인 불필요)
- `.codex-plugin/plugin.json` 신설 (Codex 호환)

### Removed
- `headers.Authorization` (OAuth 자동 처리로 불필요)
- `SPECNOTE_TOKEN` env 의존

### Migration (0.1.0 → 0.2.0)
1. `~/.zshrc` 또는 `~/.zshenv` 의 `SPECNOTE_TOKEN` env 라인 제거 (선택)
2. plugin 업데이트 후 첫 MCP 호출 시 browser 자동 로그인

## 0.1.0 — 2026-05-17
- 초기 release. PAT (Personal Access Token · SPECNOTE_TOKEN env) 인증 방식.

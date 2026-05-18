# Changelog

## 0.3.0 — 2026-05-18

### BREAKING
- `mcpServers.specnote.url` 의 env substitution `${SPECNOTE_URL:-...}` 제거 → prod URL hardcode.
  - 이전 사고: substitution 이 default 만 박혀서 "Failed to connect" 발생.
  - 영구 차단: plugin = prod URL 고정. dev 는 별도 `claude mcp add specnote-dev` 명령으로 등록.
- `SPECNOTE_URL` env 의존 제거. v0.3.0 부터 plugin 환경변수 0개.

### Changed
- README "Dev / Prod 분리" 섹션 갈아엎기 — env override → 수동 등록 명령 안내.
- Troubleshooting `Failed to connect` 섹션 갱신 (env 참조 제거 + prod URL hardcode 확인 명령).

### Migration (0.2.0 → 0.3.0)
1. `~/.zshrc` 등의 `SPECNOTE_URL` env 라인 제거 (있다면)
2. `/plugin update specnote@specnote-mcp` — Plugin 자동 갱신
3. Dev 사용자: `claude mcp add specnote-dev http://localhost:3000/api/mcp/mcp --transport http`

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

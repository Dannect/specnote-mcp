# Specnote MCP Plugin

[Specnote](https://specnote.io) 의 spec/정책 시스템을 Claude Code (또는 Codex) 에서 자연어로 다루는 MCP plugin.

**v0.2.0 — OAuth 2.1 자동 인증** (env 박는 토큰 방식 폐기). 첫 호출 시 brower 자동 로그인 → 토큰 자동 회전 → 30일간 재로그인 불필요.

---

## 무엇이 가능한가

| 자연어 요청 (예) | 자동 작동 |
|---|---|
| "Specnote 에 내 프로젝트 list 해줘" | `list_specs` — 사용자의 모든 프로젝트 목록 |
| "이 프로젝트에 환불 정책 추가해줘 — 14일 이내 전액 환불" | `add_policy` — PolicyHub 에 즉시 반영 |
| "프로젝트 X 의 정책들 보여줘" | `list_policies` — 약속·한계·나중 3 pillar |

---

## 설치 (2단계)

### Claude Code

Claude Code 안에서 다음 두 명령:

```
/plugin marketplace add Dannect/specnote-mcp
/plugin install specnote@specnote-mcp
```

설치 직후 첫 MCP 호출 (예: "specnote 에 내 프로젝트 list 해줘") 시 **자동으로 brower 가 열려 Specnote 로그인** → 권한 허용 → 끝.

토큰 발급·env 박기 불필요. Claude Code 가 OAuth 2.1 표준으로 모든 인증/회전을 자동 처리.

### Codex

같은 plugin 이 Codex 도 지원 (`.codex-plugin/` 디렉토리 박혀있음). 설치 방법은 Codex 표준 따름.

---

## Dev / Prod 자동 전환

기본 URL = `https://specnote.io/api/mcp/mcp` (production).

Specnote 본체 코드를 로컬에서 개발할 때 (`pnpm dev`) 는 환경변수로 dev URL override:

```bash
export SPECNOTE_URL="http://localhost:3000/api/mcp/mcp"
```

또는 specnote 레포 폴더의 `.envrc` + [direnv](https://direnv.net) 가 자동 처리:

```bash
brew install direnv
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
cd /path/to/specnote
direnv allow
```

이제 Specnote 폴더 진입 = dev URL 자동, 폴더 나가면 = prod URL 자동.

---

## 환경변수 사양

| 변수 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `SPECNOTE_URL` | ✗ | `https://specnote.io/api/mcp/mcp` | MCP endpoint URL. dev 시 override |

**v0.2.0 부터 `SPECNOTE_TOKEN` 불필요** — OAuth 2.1 가 자동 토큰 발급/저장/회전 처리.

---

## 보안

| 항목 | 정책 |
|---|---|
| **인증 방식** | OAuth 2.1 — Authorization Code + PKCE (S256) |
| **토큰 저장** | Claude Code 의 secure storage (사용자가 직접 다룸 X) |
| **DB 저장** | Specnote 서버는 SHA-256 hash 만 보관 — 토큰 원본 복원 불가 |
| **Access token TTL** | 1시간 (자동 회전) |
| **Refresh token TTL** | 30일 (회전 + reuse detection) |
| **권한 (scope)** | `mcp:read` (조회) + `mcp:push` (정책 추가) |
| **폐기** | Specnote 마이페이지 → MCP 연결 → "폐기" 버튼 (해당 client 의 모든 토큰 즉시 무효) |

> ⚠ 보안 사고 의심 시 즉시 마이페이지에서 해당 client 폐기.

---

## Tool 사양

| Tool | 입력 | 작동 |
|---|---|---|
| `list_specs` | (없음) | 사용자의 모든 프로젝트 목록 (id · title · status · updatedAt) |
| `add_policy` | specId · pillar (`promise`·`limit`·`future`) · area · path · text · sourceFile? | PolicyHub 에 1건 추가. 권한 검증 후 즉시 반영 |
| `list_policies` | specId | 해당 프로젝트의 모든 정책 |

자연어 요청만 하시면 됩니다 — tool 이름 직접 외울 필요 없음. Claude Code/Codex 가 자동 라우팅.

---

## 트러블슈팅

### `Failed to connect`
- URL 정상인지: `curl -i $SPECNOTE_URL` → 401 (auth 살아있음) 또는 200 기대
- dev 시 specnote 서버 띄워졌는지: `curl http://localhost:3000` → 200
- OAuth 자동 flow 진입 안 됨 → Claude Code 재시작 후 재시도

### `❌ 인증 실패` (419)
Refresh token 회전 chain 오류 (reuse detection). 마이페이지에서 해당 client 폐기 → 다시 호출 시 자동 재인증.

### `❌ 프로젝트(...)에 권한이 없거나 존재하지 않습니다`
다른 사용자의 프로젝트 ID 를 요청했거나, 삭제된 프로젝트. `list_specs` 로 확인.

---

## v0.1.0 → v0.2.0 Migration

v0.1.0 사용자라면:
1. `~/.zshrc` 또는 `~/.zshenv` 의 `SPECNOTE_TOKEN` env 라인 제거 (선택)
2. Specnote 마이페이지의 옛 PAT (`spnt_*`) 폐기 (선택)
3. Plugin 업데이트 — Claude Code 안에서 `/plugin update specnote@specnote-mcp`
4. 다음 MCP 호출 시 brower 자동 OAuth

기존 PAT 도 그대로 작동 (호환성 유지 — dev 직접 호출용).

---

## License

MIT — © Dannect

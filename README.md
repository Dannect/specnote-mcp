# Specnote MCP Plugin

[Specnote](https://specnote.io) 의 spec/정책 시스템을 Claude Code 에서 자연어로 다루는 MCP plugin.

개발 중 Claude Code 가 자동으로 정책을 누적하고, spec 을 조회할 수 있어요. 매번 회의록·README 정리할 필요 없이 코드 변경과 동시에 spec 이 살아 있는 상태로 유지됩니다.

---

## 무엇이 가능한가

| 자연어 요청 (예) | 자동 작동 |
|---|---|
| "Specnote 에 내 프로젝트 list 해줘" | `list_specs` — 사용자의 모든 프로젝트 목록 |
| "이 프로젝트에 환불 정책 추가해줘 — 14일 이내 전액 환불" | `add_policy` — PolicyHub 에 즉시 반영 |
| "프로젝트 X 의 정책들 보여줘" | `list_policies` — 약속·한계·나중 3 pillar |

---

## 설치 — 3단계

### 1. 사전 준비 — 토큰 발급

[Specnote](https://specnote.io) → 로그인 → **마이페이지** → **MCP 연결** 탭 → "토큰 발급" 버튼 클릭.

토큰 (`spnt_xxxxx...`) 이 한 번만 표시됩니다. 안전한 곳에 복사.

### 2. 환경변수 박기

본인 shell rc (`~/.zshrc` 또는 `~/.bashrc`) 에:

```bash
export SPECNOTE_TOKEN="spnt_xxxxx..."
```

저장 후 새 터미널 또는 `source ~/.zshrc`.

### 3. Plugin 설치

Claude Code 안에서:

```
/plugin marketplace add Dannect/specnote-mcp
/plugin install specnote@specnote-mcp
```

설치 후 `claude mcp list` 에서 `specnote ✓ Connected` 확인.

---

## Dev / Prod 자동 전환

Specnote 본체 레포 (`Dannect/specnote`) 에서 dev 서버 (`pnpm dev`) 띄워 작업할 때는 dev URL 로 자동 전환:

### 방법 1 — direnv (권장 · 자동)

Specnote 레포 폴더에 `.envrc` 박혀 있음. [direnv](https://direnv.net) 설치 후:

```bash
brew install direnv
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc   # bash 면 bash
source ~/.zshrc

cd /path/to/specnote
direnv allow
```

이제 Specnote 폴더 진입 = dev URL (`http://localhost:3000/api/mcp/mcp`) 자동 박힘.
폴더 나가면 = prod URL (`https://specnote.io/api/mcp/mcp`) 자동 복귀.

### 방법 2 — 수동 env

```bash
# dev 시
export SPECNOTE_URL="http://localhost:3000/api/mcp/mcp"

# prod 시 (또는 새 터미널)
unset SPECNOTE_URL
```

---

## 환경변수 사양

| 변수 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `SPECNOTE_TOKEN` | ✓ | (없음) | Personal Access Token (`spnt_` prefix · 마이페이지에서 발급) |
| `SPECNOTE_URL` | ✗ | `https://specnote.io/api/mcp/mcp` | MCP endpoint URL. dev 시 override |

---

## 보안

| 항목 | 정책 |
|---|---|
| **토큰 저장** | 사용자 shell rc 또는 secrets manager. **plugin 안 절대 박지 마세요** |
| **토큰 노출** | 발급 직후 한 번만 표시. 잃어버리면 마이페이지에서 재발급 (옛 토큰 자동 폐기) |
| **토큰 단일성** | 사용자 1인당 활성 토큰 1개. 새 발급 시 기존 토큰 즉시 무효 |
| **DB 저장** | 서버는 SHA-256 hash 만 보관 — 토큰 원본 복원 불가 |
| **scope** | `mcp:read` (조회) + `mcp:push` (정책 추가). 다른 권한 없음 |

> ⚠ 토큰이 유출됐다 싶으면 즉시 마이페이지에서 재발급. 옛 토큰 자동 폐기되어 무효화됩니다.

---

## Tool 사양

| Tool | 입력 | 작동 |
|---|---|---|
| `list_specs` | (없음) | 사용자의 모든 프로젝트 목록 (id · title · status · updatedAt) |
| `add_policy` | specId · pillar (`promise`·`limit`·`future`) · area · path · text · sourceFile? | PolicyHub 에 1건 추가. 권한 검증 후 즉시 반영 |
| `list_policies` | specId | 해당 프로젝트의 모든 정책 |

자연어 요청만 하시면 됩니다 — tool 이름 직접 외울 필요 없음. Claude Code 가 자동 라우팅.

---

## 트러블슈팅

### `Failed to connect`
- `SPECNOTE_TOKEN` 박혀있는지: `echo $SPECNOTE_TOKEN`
- URL 정상인지: `curl -i $SPECNOTE_URL` → 401 (auth 살아있음) 또는 200 기대
- dev 시 specnote 서버 띄워졌는지: `curl http://localhost:3000` → 200

### `❌ 인증 실패`
토큰 만료 또는 폐기됨. 마이페이지에서 재발급.

### `❌ 프로젝트(...)에 권한이 없거나 존재하지 않습니다`
다른 사용자의 프로젝트 ID 를 요청했거나, 삭제된 프로젝트. `list_specs` 로 확인.

---

## License

MIT — © Dannect

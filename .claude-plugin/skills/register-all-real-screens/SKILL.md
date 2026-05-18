---
name: register-all-real-screens
description: Specnote spec 에 사용자 프로젝트의 모든 실 화면(Next.js page.tsx · 비슷한 framework 의 화면 라우트)을 한 번에 자동 디스커버리·등록. hardcoded list 의존 X, stale 영구 차단. v37 wave 2 (2026-05-19) — DOM serialization 으로 갈아엎음. PNG 캡처 대신 outerHTML + computed CSS 추출 → 서버 측 DOMPurify sanitize → DB 박힘 → board iframe srcdoc 으로 Figma prototype 같이 렌더 (텍스트 select · 컴포넌트 hover · vector 줌 · CSS 애니메이션 OK). 사용자가 "실 화면 다 등록해줘" / "전체 화면 sync" / "page list 박아줘" 같은 자연어 요청 시 발동.
allowed-tools: Bash, Read, Glob, mcp__specnote__list_specs, mcp__specnote__add_realframe, mcp__specnote__list_policies, mcp__specnote__list_groups, mcp__specnote__add_group, mcp__specnote__set_group_folder_rules
---

# 모든 실 화면 일괄 등록 (Skill)

## 0. 핵심 원칙

> **사용자가 화면 신규 추가/삭제할 때마다 수동으로 list 갱신할 필요 0 — 자동 디스커버리.**
> hardcoded `PAGES` 배열 의존하면 stale 사고 (이전 v31 v5 e2e 스크립트 학습).

---

## 1. 발동 트리거

| 패턴 | 동작 |
|---|---|
| "실 화면 다 등록해줘" / "전체 화면 등록" / "page 일괄 등록" | 즉시 발동 |
| "내 프로젝트 화면 sync" / "코드 화면 다 박아줘" | 즉시 발동 |
| "spec 에 화면 등록" + "전체" / "다" / "일괄" | 즉시 발동 |
| 단일 화면 등록 1회 ("이 페이지만 등록") | 발동 X — `add_realframe` 직접 호출 |

---

## 2. 표준 흐름 (5 단계 강제)

### Step 1 — 대상 Spec 확정 (Spec ambiguity 차단)

```
1-A. `mcp__specnote__list_specs` 호출 → 사용자 spec 목록 fetch
1-B. spec 1개면 자동 선택
1-C. 2개+ 면 사용자에게 명시 질문:
     "어느 spec 에 등록할까요?
      1. <spec1 title> (cmp...)
      2. <spec2 title> (cmp...)"
1-D. 사용자가 spec id 명시 후 진행
```

**금지**: spec 못 찾으면 임의 spec 에 박지 말 것. 반드시 명시 확인.

### Step 2 — 화면 라우트 자동 디스커버리 (framework 별)

framework 자동 감지 + 해당 패턴으로 page 자동 list.

| framework | 감지 신호 | 화면 라우트 패턴 |
|---|---|---|
| **Next.js App Router** | `src/app/` 존재 + `next.config.*` | `src/app/**/page.{tsx,ts,jsx,js}` |
| **Next.js Pages Router** | `pages/` 또는 `src/pages/` 존재 | `(src/)?pages/**/*.{tsx,ts,jsx,js}` (단 `_app` · `_document` · `api/` 제외) |
| **SvelteKit** | `src/routes/` + `svelte.config.*` | `src/routes/**/+page.svelte` |
| **Remix** | `app/routes/` + `remix.config.*` | `app/routes/**/*.{tsx,ts,jsx,js}` |
| **기타** | 사용자에게 명시 질문 | 사용자가 패턴 알려줌 |

**Bash 명령 예시 (Next.js App Router)**:
```bash
find src/app -type f \( -name "page.tsx" -o -name "page.ts" -o -name "page.jsx" -o -name "page.js" \) | sort
```

### Step 3 — 라우트·라벨·group 자동 추론

각 page 파일에서 다음을 추론:

| 항목 | 추론 규칙 |
|---|---|
| **route** (URL 경로) | `src/app/[locale]/login/page.tsx` → `/[locale]/login` |
| **screenId** (식별자) | route 의 의미 segment → `"auth/login"` 형식 (`<group>/<name>` · lowercase · hyphen) |
| **label** (한국어 노출 이름) | 폴더 이름 한국어 사전 매핑 (login → "로그인", signup → "가입", dashboard → "대시보드", onboarding → "온보딩", board → "보드", mypage → "마이페이지", admin → "관리자") + 모르면 폴더 이름 그대로 |
| **group** (5 카테고리) | 폴더 이름 → 매핑 표 참조 (아래) |
| **componentName** | page.tsx 안 `export default function <Name>()` 패턴 grep |
| **visibleTexts** | page.tsx 안 한국어 문자열 (>2자) 최대 50개 추출 |

**group 매핑 표 (Specnote 기본)**:

```
auth: login, signup, verify-email, forgot-password, reset-password, oauth, signup/terms-consent
onb: onboarding/*
board: board/*
dash: dashboard, mypage, mypage/*, admin/*
marketing: / (root locale page), checkout (있을 경우)
```

**금지**:
- ❌ 코드에 없는 path 미리 박지 말 것 (예: 미래 checkout 가정)
- ❌ "marketing" 같은 group 을 폴더 없는데 박지 말 것 — 단 `[locale]/page.tsx` (root 랜딩) 는 `marketing/landing` OK (관례)
- ❌ group 분류 모호하면 사용자에게 명시 질문

### Step 3-A — 자동 그룹 생성 (v0.6.0 — 2026-05-19)

discovered pages 의 group 카테고리별로 Group row 자동 신설. 사용자가 board UI 에서 화면을 group 별로 정리해서 보게 됨.

```
3-A-1. mcp__specnote__list_groups({ specId }) → 기존 group list
3-A-2. discovered pages 의 사용된 group 카테고리 set 추출 (marketing/auth/onb/dash/board)
3-A-3. 사용되었지만 기존 group 없는 카테고리만 새로 생성:
       mcp__specnote__add_group({
         specId,
         name: "마케팅" | "인증" | "대시보드" | "온보딩" | "보드",
         icon: "megaphone" | "lock" | "layout-dashboard" | "sparkles" | "presentation",
         color: "#F59E0B" | "#9363DA" | "#06B6D4" | "#10B981" | "#FF6700",
         folderRules: ["login","signup","oauth",...]  // 카테고리별 folder pattern
       })
3-A-4. 카테고리 → groupId 매핑 보관 (Map<GroupKey, groupId>)
```

**금지**:
- ❌ 사용된 카테고리 외 group 미리 생성 (필요한 것만)
- ❌ 기존 group 이름 덮어쓰기 (있으면 skip)

### Step 4 — 각 page 마다 `add_realframe` 호출 (병렬)

#### 4-1. Playwright 로 화면 캡처 + DOM 추출 (사용자 PC dev server 띄운 상태)

v37 wave 2 (2026-05-19) — DOM serialization 표준. PNG 보다 풍부 (텍스트 select · 컴포넌트 hover · vector 줌 · CSS 애니메이션).

**v0.7.0 (2026-05-19) — 2 contexts 분기 (MANDATORY · 사용자 PS 진단 박힘)**:

비인증 페이지 (`/`, `/login`, `/signup`, `/forgot-password`, `/oauth/*` 등) 를 인증 cookies 박힌 채 진입하면 → 서버가 dashboard 으로 redirect → **잘못된 본문 캡처**. 반드시 group 기반 cookies 분기:

| group | cookies | 이유 |
|---|---|---|
| `marketing` (랜딩 · checkout) | **fresh** (cookies X) | 로그인 시 dashboard redirect |
| `auth` (login · signup · oauth · verify · reset) | **fresh** (cookies X) | 동일 — 로그인된 사용자에게 redirect |
| `onb` (온보딩) | **authed** | spec 등록 필요 → 로그인 필수 |
| `dash` (dashboard · mypage · admin) | **authed** | 인증 필수 |
| `board` | **authed** | 인증 + spec 권한 필수 |

```typescript
// v37 wave 2 + wave 7 + wave 8 풀 사이클
import { chromium } from "playwright";

// 인증 cookies 미리 확보 (login flow)
const authCookies = await loginAndGetCookies();

for (const page of discoveredPages) {
  // v0.7.0 — group 기반 cookies 분기 (ZERO TOLERANCE)
  const isPublicPage = page.group === "marketing" || page.group === "auth";
  const cookies = isPublicPage ? [] : authCookies;

  const browser = await chromium.launch();
  const ctx = await browser.newContext({ viewport: { width: 1920, height: 1080 }});
  if (cookies.length > 0) await ctx.addCookies(cookies);
  const pw = await ctx.newPage();
  await pw.goto(`http://localhost:3000${page.urlPath}`, { waitUntil: "domcontentloaded" });

  // v0.6.x — React hydration 완료 대기 (client-only 페이지 빈 본문 사고 방지)
  try { await pw.waitForLoadState("networkidle", { timeout: 5000 }); } catch {}
  await pw.waitForTimeout(1000);

  // DOM 추출 — html 통째 강제 (v0.8.0 — <html class="dark/light"> theme attr 보존 필수)
  //   document.documentElement.outerHTML = <html> 통째. body 만 뽑으면 X.
  //   서버 측 buildIframeSrcdoc 가 원본 html 그대로 보존 + head 안 base/CSP/css inject
  //   → light/dark theme · color-scheme 등 모든 root attribute 그대로 박힘
  const dom = await pw.evaluate(() => ({
    html: document.documentElement.outerHTML,
    css: Array.from(document.styleSheets).flatMap(s => {
      try { return Array.from(s.cssRules).map(r => r.cssText); }
      catch (e) { return []; }
    }).join("\n")
  }));
  if (dom.html.length > 2_000_000) throw new Error(`HTML too large: ${dom.html.length}`);
  if (dom.css.length > 1_000_000) throw new Error(`CSS too large: ${dom.css.length}`);

  await browser.close();
  // add_realframe 호출 (4-2 참조)
}
```

**금지** (ZERO TOLERANCE):
- ❌ 모든 페이지를 같은 cookies 컨텍스트로 캡처 → 비인증 페이지 본문 깨짐
- ❌ `waitUntil: "load"` 만 하고 hydration 대기 skip → client-only 페이지 빈 본문

#### 4-2. add_realframe MCP 호출 (병렬)

```
for each discovered page:
  mcp__specnote__add_realframe({
    specId: <Step 1 확정 id>,
    screenId: <Step 3 추론>,
    label: <Step 3 추론>,
    group: <Step 3 추론>,
    sourceUrl: <상대 경로 — src/app/.../page.tsx>,
    route: <Step 3 추론>,
    componentName: <Step 3 grep 결과>,
    visibleTexts: <Step 3 grep 결과>,
    hasAuth: <heuristic — Auth 관련 import 있나>,
    hasForm: <heuristic — form/input 있나>,
    hasDataFetch: <heuristic — fetch/prisma/use server 있나>,
    // v37 wave 2 — DOM serialization (선택 · 권장)
    domHtml: dom.html,  // ← 4-1 의 HTML
    domCss: dom.css,    // ← 4-1 의 CSS
    // v0.6.0 (2026-05-19) — 자동 group 매핑
    groupId: <Step 3-A 의 groupKeyToId.get(group)>  // ← 박힘 보장
  })
```

**서버 측 자동 sanitize**:
- DOMPurify (script · onclick · iframe · object · embed 제거)
- inline style 의 expression() · javascript: URL 후처리
- CSS @import · data:text/html 차단
- 사용자는 sanitize 신경 X — push 만 하면 안전 보장

**Board 안 렌더**: iframe srcdoc + sandbox="" + CSP meta 으로 3-layer 격리.

**병렬 권장** — 5-10개 동시 호출 (네트워크 latency 절감).

**용량**: 1 화면 평균 500KB~2MB (PNG 200KB 의 10x). 50 화면 ≈ 25MB · DB Postgres TEXT 박힘 OK.

### Step 5 — 결과 보고

마지막에 사용자에게 표 형식 보고:

```
✅ 실 화면 N개 등록 완료 ({specTitle})

| screenId | label | group | sourceUrl |
|---|---|---|---|
| auth/login | 로그인 | auth | src/app/[locale]/login/page.tsx |
| ...

⚠️ 실패 K건 (있을 시):
- <screenId>: <reason>
```

---

## 3. 절대 금지 (ZERO TOLERANCE)

- ❌ 사용자에게 묻지 말고 임의 spec 에 박지 말 것 (Step 1-D 명시 확인 필수)
- ❌ 코드에 없는 미래 가정 page (예: checkout) 미리 박지 말 것 — 실 file 존재 검증 후 등록
- ❌ hardcoded PAGES list 사용 절대 금지 (이번 작업의 본질 = 자동 디스커버리)
- ❌ group 분류 모호하면 임의로 박지 말고 사용자 확인
- ❌ visibleTexts 에 비밀번호·토큰·이메일 같은 PII 포함 시 마스킹

---

## 4. 사용자 옵션 (자연어로 받기)

사용자가 다음 옵션 명시할 수 있음:

| 옵션 | 의미 |
|---|---|
| "특정 폴더만" (예: "auth 폴더만") | 해당 group 만 등록 |
| "기존 거 다 지우고 다시" | 기존 등록 frame soft delete 후 재등록 (cleanup mode) |
| "신규만" (default) | 이미 등록된 sourceUrl 은 update, 신규만 create |
| "dryrun" | 실 등록 X, list 만 보여줌 (사용자 확인 후 본 실행) |

---

## 5. 결과 후 다음 단계 권장

등록 완료 후 사용자에게 안내:

- 🔗 board URL: `https://specnote.io/ko/board/{specId}` (또는 localhost dev)
- "다음 추천: `mcp__specnote__add_scenario` 로 사용자 시나리오도 박아보세요"
- "정책 누락분 확인: `mcp__specnote__list_policies({specId})` 후 add_policy"

---

## 6. 한계 (사용자에게 명시)

- MCP server 는 사용자 file system 접근 X → 디스커버리는 **Claude Code 가 사용자 PC 에서 직접 수행**
- group 자동 추론 = heuristic — 다넥트 Specnote 패턴 기준. 다른 회사 프로젝트면 약간 어색할 수 있음 (사용자 보고 후 수정 가능)
- chokidar 같은 file watch 자동화는 별도 phase (현재는 사용자가 호출 시점에만 sync)

---

## 7. 발생 이력

- **2026-05-19 v0.4.0** 신설 — 이전 v31 v5 e2e 스크립트 (`scripts/dev-e2e-register-all.ts`) 의 hardcoded PAGES 11개 stale 사고 학습. checkout 3건 (코드 없는 미래 가정) + label 11개 누락 사고 → 자동 디스커버리 영구 fix.
- **2026-05-19 v0.5.0** DOM serialization 도입 — PNG 캡처 → outerHTML + computed CSS 추출. board iframe srcdoc 으로 Figma prototype 같이 렌더 (텍스트 select · hover · 줌 · 애니메이션 OK). 서버 측 DOMPurify sanitize + iframe sandbox + CSP 3-layer 격리.
- **2026-05-19 v0.6.0** 자동 group 매핑 — register 시 groupId 박힘 → Wireframe.groupId 즉시 set. Step 3-A 에서 사용된 카테고리만 group 자동 신설 (마케팅/인증/대시보드/온보딩/보드). 사용자가 board UI 에서 화면을 카테고리별 column 으로 보게 됨. dev-e2e-register-all.ts e2e 검증 통과 (22 화면 + 5 그룹).
- **2026-05-19 v0.7.0** 2 contexts 분기 (PS 진단 박힘) — 사용자 사고: auth/login 캡처가 dashboard 본문으로 박힘. 원인: 모든 페이지를 인증 cookies 박힌 채 캡처 → 비인증 페이지가 서버 redirect. 해결: group 기반 분기 (marketing/auth = fresh / 그 외 = authed) + React hydration networkidle+1s wait. ScreenCard footer label 우선 표시 (영문 screenId 노출 차단).
- **2026-05-19 v0.8.0** light/dark theme 보존 — 사용자 사고: 카드 미리보기에 흰색 영역 노출. 원인: server buildIframeSrcdoc 가 새 `<html lang="ko">` 으로 wrap → 원본 `<html class="light">` class 잃음 → CSS theme vars 잘못 매핑. 해결: server 측 원본 html 그대로 보존 + head 안 base/CSP/css inject. MCP skill 측 `document.documentElement.outerHTML` (html 통째) 사용 명시 — body 만 뽑으면 X. 사용자 실 화면과 동일한 theme 표현.
- **2026-05-19 v0.9.0** iframe 안 click → board 카드 jump — 사용자 사고: 확대 modal 안 link 클릭 시 iframe 자체 navigation (카드 안 화면만 바뀜). 사용자 의도: 우리 board 의 해당 route 카드로 jump. 해결 (3 layer): (1) server buildIframeSrcdoc 가 nonce 박힌 inject script 추가 — anchor click + form submit 가로채서 postMessage. (2) iframe sandbox `""` → `"allow-scripts"` + CSP `script-src 'nonce-XXX'` (사용자 script 차단 · 우리 script 만 허용). (3) InfiniteCanvasView 의 message listener — href → pathname → locale 제거 → wireframe.route 매칭 → setDetailScreenId jump. MCP plugin 자체 코드 변경 X (server 측 + 클라이언트 측 변경만).
- **2026-05-19 v0.10.0** Next.js Server Action hidden input 노출 차단 — 사용자 사고: 화면 미리보기에 base64 string 가득 + 흰 textbox. 진단: `<input name="user-content-$ACTION_*">` 가 type="hidden" 누락된 채 박힘 → default type="text" → encrypted ref string 노출. 해결: server buildIframeSrcdoc 의 inject CSS 에 `input[name*="$ACTION"], input[name*="$STATE"], input[name^="user-content-$"] { display: none !important }` 강제 숨김. 사용자 실 입력 form (email/password) 영향 X.

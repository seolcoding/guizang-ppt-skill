# Microsoft 365 Copilot Integration Guide

> **English** | [한국어](#microsoft-365-copilot-통합-가이드)

---

## Overview

This guide explains how to reference and integrate **guizang-ppt-skill** in a Microsoft 365 Copilot environment (Copilot Studio, Declarative Agents, and Teams extensions).

The skill generates **single-file HTML horizontal-swipe presentations** using pure HTML/CSS/JavaScript — no build step, no server required. This makes it uniquely well-suited for Copilot integration because the entire output is text and can be returned as a message, saved to OneDrive, or piped into a conversion service.

---

## Pattern A — HTML-as-text response (simplest)

**How it works:** The agent generates the complete HTML deck as a text response. The user copies and saves it as `index.html`.

**When to use:** When you do not have a backend conversion service and want zero-infrastructure deployment.

**Copilot Studio configuration:**

1. Create a Declarative Agent in [Copilot Studio](https://copilotstudio.microsoft.com).
2. In the **Instructions** field, paste the contents of `SKILL.md` (or reference it via a SharePoint page or Knowledge source).
3. In **Conversation starters**, add prompts such as:
   - "Create a Swiss-style slide deck about our product launch"
   - "Make a magazine-style HTML presentation from this document"
4. The agent will output the full HTML. Tell users to save the response as `index.html` and open it in their browser.

**Limitations:** Response length limits in Copilot (currently ~4,000 tokens per turn) may truncate large decks. Use Pattern B or C for decks exceeding ~12 slides.

---

## Pattern B — Copilot Studio MCP connector to an external conversion server

**How it works:** You host the skill's generation logic as a microservice (e.g., on Fly.io, Cloudflare Workers, or Azure Container Apps). Copilot Studio connects to it via an OpenAPI action.

**When to use:** When you need full deck generation without token-length constraints, or when you want to serve the HTML directly as a URL.

**Architecture:**

```
User → M365 Copilot (Declarative Agent)
           │
           │ OpenAPI Action (HTTP POST /convert)
           ▼
  External conversion server
  (Node.js / Python + guizang-ppt-skill templates)
           │
           │ Returns: { "html": "...", "previewUrl": "..." }
           ▼
  Copilot relays the previewUrl to the user
```

**Steps:**

1. Deploy the conversion server using the template in `examples/copilot-openapi-action.yaml` as your API specification.
2. In Copilot Studio, go to **Actions > Add an action > From an OpenAPI description** and upload `examples/copilot-openapi-action.yaml` (with your real endpoint substituted for `your-converter.example.com`).
3. Add the action to your Declarative Agent manifest (see `examples/copilot-declarative-agent-manifest.json`).
4. The agent will call `/convert` with the user's topic/outline, receive the HTML, and return a preview link.

**Required server environment:**

- Node.js >= 18 or Python >= 3.11
- Read access to `assets/template.html` and `assets/template-swiss.html`
- No authentication required for the templates themselves; secure the `/convert` endpoint with an API key passed in the `X-Api-Key` header.

---

## Pattern C — CodeInterpreter capability (experimental)

**How it works:** Enable the `CodeInterpreter` capability in the Declarative Agent manifest. Copilot executes Python in a sandboxed environment to render the HTML deck directly.

**When to use:** Microsoft 365 E5 or Copilot for Microsoft 365 tenants with CodeInterpreter enabled (check with your tenant admin).

**Steps:**

1. Verify that `codeInterpreter` is listed under `capabilities` in your tenant's Copilot feature flags.
2. In your Declarative Agent manifest (`examples/copilot-declarative-agent-manifest.json`), ensure `"codeInterpreter": {}` is present in the `capabilities` array.
3. Instruct the agent to read the template file and generate the HTML using Python string manipulation (no `python-pptx` needed — the output is HTML, not PPTX).
4. The agent can then offer the file via `FileAttachmentCard` or write it to the user's OneDrive.

**Note:** CodeInterpreter availability varies by tenant configuration and Copilot SKU. Test in your environment before committing to this pattern.

---

## Knowledge source recommendation

For best results, add the following files as Knowledge sources in Copilot Studio (via SharePoint or uploaded documents):

| File | Purpose |
|------|---------|
| `SKILL.md` | Core workflow and design rules |
| `references/layouts.md` | Style A layout templates |
| `references/layouts-swiss.md` | Style B layout templates |
| `references/themes.md` | Style A color presets |
| `references/themes-swiss.md` | Style B color presets |
| `references/checklist.md` | Quality checklist |

Do not include `assets/template.html` or `assets/template-swiss.html` as Knowledge — they are large seed files best referenced by the backend service.

---

## Security notes

- Never commit real API keys. Use `SERVICE_KEY_PLACEHOLDER` in config files.
- If you host a conversion server, restrict CORS to your M365 tenant domain.
- HTML output is user-controlled content. Serve preview URLs from a sandboxed domain (not your primary corporate domain).

---

## Microsoft 365 Copilot 통합 가이드

### 개요

이 문서는 **guizang-ppt-skill**을 Microsoft 365 Copilot 환경(Copilot Studio, Declarative Agent, Teams 확장)에서 참조하고 연동하는 세 가지 방법을 설명합니다.

이 스킬은 순수 HTML/CSS/JS로 **단일 파일 HTML 횡스크롤 PPT**를 생성합니다. 빌드 스텝이나 서버가 필요 없고 출력이 텍스트이므로 Copilot 메시지로 반환하거나 OneDrive에 저장하거나 변환 서버로 전달하기 적합합니다.

---

### 패턴 A — HTML 텍스트 응답 (가장 단순)

에이전트가 완성된 HTML을 텍스트로 반환합니다. 사용자가 `index.html`로 저장해 브라우저에서 열면 됩니다.

**Copilot Studio 설정 순서:**

1. [Copilot Studio](https://copilotstudio.microsoft.com)에서 Declarative Agent를 생성합니다.
2. **Instructions** 필드에 `SKILL.md` 내용을 붙여넣거나, SharePoint 페이지를 Knowledge 소스로 추가합니다.
3. 대화 시작 문구 예시를 추가합니다:
   - "제품 출시 관련 스위스 스타일 슬라이드 덱을 만들어줘"
   - "이 문서를 기반으로 잡지풍 HTML PPT를 만들어줘"

**제한:** Copilot 응답 길이 제한(약 4,000 토큰)으로 큰 덱은 잘릴 수 있습니다. 12슬라이드 초과 시 패턴 B 또는 C를 사용하세요.

---

### 패턴 B — Copilot Studio MCP 커넥터 + 외부 변환 서버

스킬 생성 로직을 마이크로서비스(Fly.io, Cloudflare Workers, Azure Container Apps 등)로 호스팅하고 Copilot Studio에서 OpenAPI Action으로 연결합니다.

**아키텍처:**

```
사용자 → M365 Copilot (Declarative Agent)
              │
              │ OpenAPI Action (HTTP POST /convert)
              ▼
    외부 변환 서버
    (Node.js / Python + guizang-ppt-skill 템플릿)
              │
              │ 반환: { "html": "...", "previewUrl": "..." }
              ▼
    Copilot이 previewUrl을 사용자에게 전달
```

**설정 순서:**

1. `examples/copilot-openapi-action.yaml`을 API 명세로 사용해 변환 서버를 배포합니다.
2. Copilot Studio > **작업 > 작업 추가 > OpenAPI 설명에서** 경로로 `examples/copilot-openapi-action.yaml`을 업로드합니다 (`your-converter.example.com`을 실제 엔드포인트로 교체).
3. Declarative Agent 매니페스트에 작업을 추가합니다(`examples/copilot-declarative-agent-manifest.json` 참조).

---

### 패턴 C — CodeInterpreter 기능 (실험적)

Declarative Agent 매니페스트에서 `CodeInterpreter` 기능을 활성화합니다. Copilot이 샌드박스 환경에서 Python을 실행해 HTML 덱을 직접 렌더링합니다.

**사전 조건:** Microsoft 365 E5 또는 Copilot for Microsoft 365 테넌트에서 CodeInterpreter가 활성화되어 있어야 합니다(테넌트 관리자에게 확인).

---

### Knowledge 소스 권장

Copilot Studio에서 SharePoint나 업로드 문서로 다음 파일을 Knowledge 소스로 추가하면 품질이 향상됩니다:

| 파일 | 용도 |
|------|------|
| `SKILL.md` | 핵심 워크플로우 및 디자인 규칙 |
| `references/layouts.md` | 스타일 A 레이아웃 템플릿 |
| `references/layouts-swiss.md` | 스타일 B 레이아웃 템플릿 |
| `references/themes.md` | 스타일 A 색상 프리셋 |
| `references/themes-swiss.md` | 스타일 B 색상 프리셋 |
| `references/checklist.md` | 품질 체크리스트 |

---

### 보안 주의사항

- 실제 API 키를 커밋하지 마세요. 설정 파일에는 `SERVICE_KEY_PLACEHOLDER`를 사용하세요.
- 변환 서버를 호스팅할 경우 CORS를 M365 테넌트 도메인으로 제한하세요.
- HTML 출력은 사용자가 제어하는 콘텐츠입니다. 미리보기 URL은 샌드박스 도메인에서 제공하세요(회사 주요 도메인 사용 금지).


## https://github.com/steel-dev/steel-browser

Read [](file:///Users/danac/projects/web-automation/steel-browser/README.md#1-1), lines 1 to 100

Read [](file:///Users/danac/projects/web-automation/steel-browser/docs/ARCHITECTURE.md#1-1), lines 1 to 100

Read [](file:///Users/danac/projects/web-automation/steel-browser/README.md#100-100), lines 100 to 200

Read [](file:///Users/danac/projects/web-automation/steel-browser/api/src/routes.ts)

Read [](file:///Users/danac/projects/web-automation/steel-browser/README.md#200-200), lines 200 to 300

Here's a comprehensive breakdown of **Steel Browser**:

---

### What It Is

Steel is an open-source **browser API server** designed for AI agents and web automation. It wraps Chrome/Chromium with a REST API, so you can control a browser over HTTP instead of managing Puppeteer/Playwright infrastructure yourself.

---

### Architecture

```
REST API (Fastify, port 3000)
    └── Session Manager → CDP Service → Chrome (via Puppeteer)
                                     ↘ Plugin System (stealth, proxy, fingerprint, etc.)
React UI (port 3000/ui or 5173 in dev)
    └── Session viewer, DevTools integration, real-time debugging
```

Key source locations:
- api/src/modules/ — route handlers for sessions, actions, CDP, files, logs, selenium
- api/src/services/ — CDP service, session management
- ui/src/ — React frontend

---

### Running It

**Simplest (Docker):**
```bash
docker run -p 3000:3000 -p 9223:9223 ghcr.io/steel-dev/steel-browser
```

**Docker Compose:**
```bash
docker compose up
# Mac Silicon:
DOCKER_DEFAULT_PLATFORM=linux/arm64 docker compose up
```

**Node.js (requires Chrome installed):**
```bash
npm install
npm run dev   # API on :3000, UI on :5173
```

---

### Two Main Usage Patterns

#### 1. Sessions API — stateful, full-control browser sessions

Create a session, then connect Puppeteer/Playwright/Selenium to it:

```bash
# Create a session
curl -X POST http://localhost:3000/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{ "blockAds": true, "proxyUrl": "user:pass@host:port" }'
```

```typescript
// Node SDK
import Steel from 'steel-sdk';
const client = new Steel({ baseURL: "http://localhost:3000" });
const session = await client.sessions.create({ blockAds: true });
// then connect Playwright/Puppeteer using session.id
```

Selenium sessions are also supported via `isSelenium: true`.

#### 2. Quick Actions API — stateless, one-shot operations

No session needed — just fire and forget:

```bash
# Scrape a page to HTML/Markdown
curl -X POST http://localhost:3000/v1/scrape \
  -d '{ "url": "https://example.com" }'

# Take a screenshot
curl -X POST http://localhost:3000/v1/screenshot \
  -d '{ "url": "https://example.com" }'

# Export to PDF
curl -X POST http://localhost:3000/v1/pdf \
  -d '{ "url": "https://example.com" }'
```

---

### Key Features

| Feature | Details |
|---|---|
| **Multi-framework** | Works with Puppeteer, Playwright, Selenium |
| **Anti-detection** | Stealth plugins, fingerprint management |
| **Proxy support** | Per-session proxy chains |
| **Extensions** | Load custom Chrome extensions |
| **Session isolation** | Separate cookies/localStorage per session |
| **Debug UI** | Live session viewer at `http://localhost:3000/ui` |
| **OpenAPI docs** | Interactive Swagger at `http://localhost:3000/documentation` |

---

### SDKs

```bash
npm install steel-sdk       # Node
pip install steel-sdk       # Python
```

Both SDKs are fully typed and work with Steel Cloud or self-hosted instances via a `baseURL`/`base_url` option.

## PinchTab

PinchTab, a high-performance browser automation bridge designed specifically for AI agents.

You can find the repository here: pinchtab/pinchtab on GitHub.

Here are the key details that confirm it matches your description:

    Rapid Traction: The project has seen a surge in popularity, quickly accumulating over 8,800 GitHub stars, far exceeding the 6,000-star mark you mentioned. Its rise has been notably rapid.

    Focus on Infrastructure: PinchTab acts as a standalone HTTP server that gives AI agents direct, programmatic control over a Chrome browser through a simple API. It's designed as a lightweight "bridge" that handles all the underlying browser communication for the agent.

    Key Features: It differentiates itself by being a 12MB binary written in Go with zero dependencies, offering a REST API that can be used from any programming language. Its efficiency is a core feature, as it can dramatically reduce token usage (up to 13x fewer tokens) for AI agents by using the browser's accessibility tree

### https://www.hubwiz.com/blog/pinchtab-browser-control-via-http/

如果你尝试过给 AI 代理提供浏览器访问权限，你应该已经知道问题所在。Playwright MCP 把你绑在 Node 上。Browser Use 需要 Python。OpenClaw 的浏览器后端只在其自己的生态系统中工作。切换代理，或者尝试快速发出一个 curl 请求来检查页面，你就得从头开始重写集成。

Pinchtab 采取了不同的方法：它只是一个 HTTP 服务器。一个 12MB 的 Go 二进制文件，不需要 Node、Python，也不依赖任何东西。它会启动自己的 Chrome，并通过一个简单的 REST API 暴露所有功能——导航、点击、表单填充、截图、可访问性快照。无论你使用什么代理，只要能说 HTTP，这就是完整的集成故事。

1、为什么使用可访问性树而不是截图

大多数人首先选择截图，因为这很直观：拍张照片，发送给视觉模型，完成。问题在于成本。使用截图的 10 步任务大约花费 $0.06。而使用可访问性树的相同任务大约花费 $0.015。

真正的差异在规模化时显现出来。运行相同的任务 1000 次，截图需要花费 $60；可访问性树只需要 $15。运行一个 50 页的监控任务：

[token consumption](./token-consumption.png)

Pinchtab 的 /text 端点使用 Mozilla 的 Readability 库（Firefox 阅读器视图背后的技术）以每页约 800 个 Token 提取可读内容。这比完整的可访问性快照便宜 5 倍，比截图便宜 13 倍。对于以阅读为主的工作，差异会迅速累积。

还有一个可靠性的论据。视觉模型从像素猜测坐标。可访问性树为你提供了绑定到实际 DOM 元素的稳定节点引用（e0、e1、e2...）。点击 e5，你会点击到正确的按钮，无论页面如何渲染。

2、完整的 API

Pinchtab 超越了基本的导航和点击：

[more details](./Pinchtab-control-browser-by-http.md)

## Browser Use

Browser Use是指在原文语境中，一种需要Python环境、依赖LLM和CDP协议的浏览器自动化库，主要用于AI智能体通过自然语言描述任务并自主执行网页操作。它与Pinchtab的HTTP控制方案形成对比，强调了多语言模型支持、事件驱动架构和复杂任务处理能力，而Pinchtab则采用轻量Go语言实现HTTP服务器方案。
Browser Use是一个基于LLM和CDP的异步Python浏览器自动化库，核心是通过事件驱动架构和LLM驱动的决策循环，让AI智能体能够自主导航网页、与元素交互并完成复杂任务。它支持多种LLM提供商（OpenAI、Anthropic等），提供页面导航、元素交互、表单填写等完整浏览器自动化能力，特别适用于AI智能体的复杂Web交互场景。
Browser Use通过CDP协议直接控制浏览器，无需手动编写复杂脚本，通过LLM实现智能决策，无需显式的元素选择器。它采用Watchdog系统实现自动化监控，支持完整的DOM处理和可访问性树，确保智能体准确理解页面结构。此外，Browser Use还提供云端浏览器服务，支持隐身模式和代理轮换，适用于自动化测试、数据抓取、任务自动化等场景。与传统工具相比，Browser Use更注重自然语言交互，降低了编程门槛，提高了自动化任务的效率和准确性。



# https://www.hubwiz.com/blog/pinchtab-browser-control-via-http/

Software 2.0

  • 首页
  • AI应用
  • AI开发包
  • AI数据集
  • AI模型库
  • AI工具箱
  • 在线课程
  • 自学指南

Subscribe
TOOL

Pinchtab: 通过 HTTP 控制浏览器

Pinchtab 是一个 12MB 的 Go 二进制文件，它让任何 AI 代理都能通过纯 HTTP API 和可
访问性树来控制浏览器。零配置，不依赖特定框架，而且比截图方式便宜得多。

  •  

admin

Mar 9, 2026 • 12 min read

Pinchtab: 通过 HTTP 控制浏览器

    微信 ezpoda免费咨询：AI编程 | AI模型微调| AI私有化部署
    AI模型价格对比 | AI工具导航 | ONNX模型库 | Tripo 3D | Meshy AI | ElevenLabs
    | KlingAI | ArtSpace | Phot.AI | InVideo

如果你尝试过给 AI 代理提供浏览器访问权限，你应该已经知道问题所在。Playwright
MCP 把你绑在 Node 上。Browser Use 需要 Python。OpenClaw 的浏览器后端只在其自己
的生态系统中工作。切换代理，或者尝试快速发出一个 curl 请求来检查页面，你就得从
头开始重写集成。

Pinchtab 采取了不同的方法：它只是一个 HTTP 服务器。一个 12MB 的 Go 二进制文件，
不需要 Node、Python，也不依赖任何东西。它会启动自己的 Chrome，并通过一个简单的
REST API 暴露所有功能——导航、点击、表单填充、截图、可访问性快照。无论你使用什么
代理，只要能说 HTTP，这就是完整的集成故事。

1、为什么使用可访问性树而不是截图

大多数人首先选择截图，因为这很直观：拍张照片，发送给视觉模型，完成。问题在于成
本。使用截图的 10 步任务大约花费 $0.06。而使用可访问性树的相同任务大约花费
$0.015。

真正的差异在规模化时显现出来。运行相同的任务 1000 次，截图需要花费 $60；可访问
性树只需要 $15。运行一个 50 页的监控任务：

            方法             ~Token 数 估算成本
Screenshots (vision)         ~100,000  $0.30
Full a11y snapshot           ~525,000  $0.16
Pinchtab ?filter=interactive ~180,000  $0.05
Pinchtab /text               ~40,000   $0.01

Pinchtab 的 /text 端点使用 Mozilla 的 Readability 库（Firefox 阅读器视图背后的
技术）以每页约 800 个 Token 提取可读内容。这比完整的可访问性快照便宜 5 倍，比截
图便宜 13 倍。对于以阅读为主的工作，差异会迅速累积。

还有一个可靠性的论据。视觉模型从像素猜测坐标。可访问性树为你提供了绑定到实际
DOM 元素的稳定节点引用（e0、e1、e2...）。点击 e5，你会点击到正确的按钮，无论页
面如何渲染。

2、完整的 API

Pinchtab 超越了基本的导航和点击：

方法    端点                          描述
GET  /health     检查服务器和 Chrome 是否响应
GET  /tabs       列出所有打开的标签页及其 ID
GET  /snapshot   可访问性树，以 JSON 或文本格式
GET  /screenshot JPEG 截图，支持质量控制
GET  /text       可读页面文本（Readability 或原始 innerText）
POST /navigate   在标签页中导航到 URL
POST /action     点击、输入、填充、按下、聚焦、悬停、选择、滚动
POST /evaluate   运行任意 JavaScript
POST /tab        打开或关闭标签页
POST /tab/lock   锁定标签页以供代理独占访问
POST /tab/unlock 释放标签页锁定
POST /cookies    通过编程方式注入会话 Cookie

如果你同时运行多个代理，/tab/lock 端点值得注意。一个代理锁定一个标签页，完成工
作，然后解锁它。不会对同一个浏览器上下文产生竞争写入。

3、快照查询参数

快照端点有几个选项可以显著减少 token 使用量：

# 仅交互元素 - 节点减少约 75%
curl "localhost:9867/snapshot?filter=interactive"

# 紧凑的每节点一行格式 - 比 JSON 少 56-64% 的 token
curl "localhost:9867/snapshot?format=compact"

# 仅自上次快照以来的变化
curl "localhost:9867/snapshot?diff=true"

# 限制到页面的特定部分
curl "localhost:9867/snapshot?selector=main"

# 将输出限制在大约 N 个 token
curl "localhost:9867/snapshot?maxTokens=2000"

将 filter=interactive 与 format=compact 结合使用，可以为面向操作的任务提供尽可
能小的负载。对于增量更新的页面（例如，轮询实时仪表板），使用 diff=true，这样你
只发送实际发生变化的内容。

4、类似人类的操作

除了标准的点击和输入，Pinchtab 还有 humanClick 和 humanType 操作，可以添加逼真
的延迟和移动模式。在访问会监视交互时序的行为机器人检测的网站上时很有用。

curl -X POST localhost:9867/action \
  -d '{"kind":"humanType","ref":"e12","text":"hello world"}'

5、配置

所有配置都通过环境变量进行：

        变量               默认值                        描述
BRIDGE_PORT          9867               HTTP 端口
BRIDGE_TOKEN         (无)               用于认证的 Bearer 令牌
BRIDGE_HEADLESS      false              在没有窗口的情况下运行 Chrome
BRIDGE_STEALTH       light              light（webdriver 补丁）或 full（canvas/
                                        WebGL/字体伪装）
BRIDGE_PROFILE       ~/.pinchtab/       Chrome 配置文件目录
                     chrome-profile
BRIDGE_STATE_DIR     ~/.pinchtab        状态和会话存储
BRIDGE_BLOCK_IMAGES  false              跳过图像下载
BRIDGE_BLOCK_MEDIA   false              阻止图像、字体、CSS、视频
BRIDGE_NO_ANIMATIONS false              全局冻结 CSS 动画
BRIDGE_TIMEOUT       15                 操作超时（秒）
BRIDGE_NAV_TIMEOUT   30                 导航超时（秒）
BRIDGE_TIMEZONE      (系统)             Chrome 时区（例如 America/New_York）
CDP_URL              (无)               连接到现有的 Chrome 而不是启动新的
CHROME_BINARY        (自动)             Chrome 或 Chromium 的路径
CHROME_FLAGS         (无)               额外的 Chrome 启动标志

BRIDGE_BLOCK_MEDIA 是更激进的版本——它跳过除 HTML 和 JavaScript 之外的所有内容。
对于页面保真度不重要的大规模爬取很有用。如果你在动画中间抓取页面快照并获得不一
致的结果，BRIDGE_NO_ANIMATIONS 会有所帮助。

如果你更喜欢 JSON 而不是环境变量，也可以生成配置文件：

pinchtab config init   # 创建 ~/.pinchtab/config.json
pinchtab config show   # 显示当前有效配置

环境变量会覆盖配置文件，因此它可以与 Docker 密钥或 .env 文件很好地配合使用。

6、Docker 部署

在服务器上运行 Pinchtab 的最简单方法。Chrome 在容器中需要 seccomp=unconfined，
这是你想将其与堆栈的其余部分隔离的主要原因。

6.1 基本的 docker-compose 设置

# docker-compose.yml
services:
  pinchtab:
    image: pinchtab/pinchtab:latest
    container_name: pinchtab
    restart: unless-stopped
    security_opt:
      - seccomp:unconfined
    mem_limit: 2g
    cpus: "2.0"
    environment:
      - BRIDGE_PORT=9867
      - BRIDGE_HEADLESS=true
      - BRIDGE_STEALTH=full
      - BRIDGE_TOKEN=${PINCHTAB_TOKEN}
      - BRIDGE_BLOCK_IMAGES=true
      - BRIDGE_NO_ANIMATIONS=true
    volumes:
      - pinchtab-data:/data
    ports:
      - "127.0.0.1:9867:9867"

volumes:
  pinchtab-data:

这里有几件事需要注意。端口绑定 127.0.0.1:9867:9867 仅在 localhost 上公开服务，
而不是向网络公开。内存限制很重要：打开几个标签页的 Chrome 很容易使用 1-1.5GB。
如果你只做内容相关的任务，设置 BRIDGE_BLOCK_IMAGES=true 有助于保持较低的内存使
用量。

6.2 在 Caddy 反向代理后面

如果你想通过域名和 HTTPS 公开 Pinchtab：

# docker-compose.yml
services:
  pinchtab:
    image: pinchtab/pinchtab:latest
    container_name: pinchtab
    restart: unless-stopped
    security_opt:
      - seccomp:unconfined
    mem_limit: 2g
    cpus: "2.0"
    environment:
      - BRIDGE_PORT=9867
      - BRIDGE_HEADLESS=true
      - BRIDGE_STEALTH=full
      - BRIDGE_TOKEN=${PINCHTAB_TOKEN}
      - BRIDGE_BLOCK_IMAGES=true
    volumes:
      - pinchtab-data:/data
    networks:
      - proxy

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
      - caddy-config:/config
    networks:
      - proxy

networks:
  proxy:
    driver: bridge

volumes:
  pinchtab-data:
  caddy-data:
  caddy-config:

# Caddyfile
pinchtab.yourdomain.com {
    reverse_proxy pinchtab:9867
}

Caddy 会自动处理 TLS。设置了 BRIDGE_TOKEN 后，请求需要 Authorization: Bearer
<token> 标头，因此即使有 HTTPS，API 也不会向公众开放。

curl -H "Authorization: Bearer $PINCHTAB_TOKEN" \
  https://pinchtab.yourdomain.com/health

6.3 从源代码构建 vs 预构建镜像

仓库中的官方 docker-compose.yml 使用 build: .，它从源代码编译。如果你想要预构建
的镜像，请使用 image: pinchtab/pinchtab:latest。查看发布页面以获取可用标签。

7、安全顾虑和缓解措施

Pinchtab 给 AI 代理对真实 Chrome 浏览器的完全控制权，包括你通过该浏览器登录的任
何帐户。README 对此很直接："把 Pinchtab 当作把你的未锁定笔记本电脑给别人。" 这
在实践中意味着什么以及如何处理它。

7.1 默认情况下没有认证

开箱即用，Pinchtab 接受来自任何可以访问端口 9867 的人的请求。在共享网络或具有公
共 IP 的服务器上，这意味着任何人。

缓解措施：始终设置 BRIDGE_TOKEN。设置后，每个请求都需要 Authorization: Bearer
<token>，否则会收到 401。将此令牌视为密码——生成一些长而随机的内容，将其存储在环
境变量或密钥管理器中，永远不要硬编码它。

# 生成令牌
openssl rand -hex 32

7.2 Pinchtab 绑定到所有接口

默认情况下，服务器监听 0.0.0.0，而不仅仅是 localhost。在云服务器上，这意味着如
果防火墙允许，端口 9867 可以从任何地方访问。

缓解措施：要么仅将端口绑定到 localhost（如上面的 docker-compose 中所示，使用
127.0.0.1:9867:9867），要么设置防火墙规则阻止对端口 9867 的外部访问。使用像
Caddy 或 Nginx 这样的反向代理增加了另一层，并让你可以正确处理 TLS。

7.3 Chrome 配置文件保存实时会话

当你通过 Pinchtab 的 Chrome 窗口登录到站点时，该会话会保存在 ~/.pinchtab/
chrome-profile/ 中。Cookie、保存的密码、身份验证令牌。具有 API 访问权限的代理可
以使用这些会话在你登录的任何网站上作为你行动。

缓解措施：

  • 使用专用的 Chrome 配置文件，其中仅包含代理实际需要的帐户
  • 不要将个人帐户（电子邮件、银行、社交）登录到 Pinchtab 配置文件，除非你特别
    需要它们自动化
  • 将 ~/.pinchtab/ 视为敏感并限制文件权限：chmod 700 ~/.pinchtab
  • 在 Docker 中，使用命名卷，并避免将其挂载为只读（Pinchtab 需要写入状态），但
    不要将其挂载到其他容器可访问的地方

seccomp=unconfined 要求

Chrome 需要一个宽松的 seccomp 配置文件才能在容器中运行。这是一个真正的特权：它
删除了一层内核系统调用过滤。

缓解措施：如果不为 Chromium 构建自定义 seccomp 配置文件，很难完全缓解这一点。实
际方法是隔离 Pinchtab 容器——不要将其与具有数据库访问权限或其他敏感服务的容器运
行在同一网络上。将其保留在自己的网络段中，只有你的代理服务才能访问该段。

7.4 代理信任模型

具有 Pinchtab 访问权限的代理可以执行使用该浏览器的任何人可以做的任何事情。如果
你的代理接受来自用户或外部输入的任意指令，提示注入是一个真正的担忧：恶意网站可
能在其内容中包含指令，欺骗代理采取非预期的行动。

缓解措施：

  • 限制代理被允许做的事情。如果它只需要读取页面，不要给它操作能力
  • 记录所有 /action 和 /navigate 调用，以便你可以审计发生了什么
  • 考虑在沙盒化配置文件中运行代理，该配置文件没有用于不受信任输入的真实帐户
  • 限制你的 Pinchtab 端点的速率——如果你将其暴露给外部代理，请在 Caddy 或 Nginx
    中添加速率限制

7.5 不要跳过令牌

在任何连接到 Internet 的服务器上不使用 BRIDGE_TOKEN 运行 Pinchtab 是一个严重的
风险。任何找到端口的人都有完全的浏览器控制权。在设置其他任何内容之前先设置令牌
。

8、典型的代理工作流

import httpx

BASE = "http://localhost:9867"
HEADERS = {"Authorization": "Bearer your-token"}

# 导航到页面
httpx.post(f"{BASE}/navigate", json={"url": "https://example.com"}, headers=HEADERS)

# 仅获取交互元素以保持低 token
snapshot = httpx.get(f"{BASE}/snapshot?filter=interactive&format=compact", headers=HEADERS)
refs = snapshot.json()

# 通过 ref 点击按钮
httpx.post(f"{BASE}/action", json={"kind": "click", "ref": "e5"}, headers=HEADERS)

# 读取结果
text = httpx.get(f"{BASE}/text", headers=HEADERS)
print(text.text)

节点引用在快照内是稳定的。在加载新页面的点击后，获取一个新的快照——新页面上的引
用是独立的。对于在没有完全加载的情况下更新的页面（React、Vue SPA），使用 ?diff=
true 来查看发生了什么变化，而不是重新获取整个树。

对于多个代理共享一个浏览器实例的多代理设置，使用标签页锁定：

# 锁定标签页 1 以供独占使用
curl -X POST localhost:9867/tab/lock -d '{"tabId": 1}'

# ... 做你的工作 ...

# 释放它
curl -X POST localhost:9867/tab/unlock -d '{"tabId": 1}'

9、适用场景

Pinchtab 适合需要浏览 Web 但不需要与浏览器测试框架紧密耦合的代理。具体来说：

  • 任何规模的 Web 抓取和内容监控
  • 表单自动化（注册、数据输入、多步工作流）
  • 一次性登录设置后的身份验证抓取
  • 在 bash 脚本、Go 程序或任何支持 HTTP 的语言中运行的代理
  • 你想要在不同的代理框架之间切换而不更改浏览器集成的设置

它不是像素级精确视觉测试（Playwright 在这方面更好）或可访问性树稀疏或不可靠的站
点的正确工具。一些使用自定义组件构建的单页应用不会暴露太多有用的 ARIA 数据，完
整的截图成为更实用的选择。

10、开始使用

# Docker（最简单，不需要安装 Chrome）
docker run -d \
  -p 127.0.0.1:9867:9867 \
  --security-opt seccomp:unconfined \
  -e BRIDGE_TOKEN=your-secret-token \
  -e BRIDGE_HEADLESS=true \
  pinchtab/pinchtab:latest

curl -H "Authorization: Bearer your-secret-token" http://localhost:9867/health

# 从源代码构建（需要 Go 1.25+ 和已安装 Chrome）
git clone https://github.com/pinchtab/pinchtab.git
cd pinchtab
go build -o pinchtab .
BRIDGE_HEADLESS=true BRIDGE_TOKEN=your-secret-token ./pinchtab

GitHub 仓库有一个 OpenClaw 技能，如果你使用支持技能的代理，它可以自动安装和配置
Pinchtab。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

原文链接: Pinchtab: Browser Control via HTTP for AI Agents

汇智网翻译整理，转载请标明出处

[wechat-gro]
我的无聊AI生意，每月赚25万美元

我的无聊AI生意，每月赚25万美元

没人谈论的无聊模式——以及为什么它每个月都在默默印钞

  •  

admin Apr 22, 2026 • 7 min read
AI 用户体验：通过探索发现意图

AI 用户体验：通过探索发现意图

AI 不仅仅是一个更好的聊天框。它将用户的角色从操作者转变为监督者，这迫使 UX 从基
于命令的交互转向基于意图的委托、新的可用性指标、编排层、校准后的摩擦阻力，最终
转向基于探索的交互以澄清用户需求。

  •  

admin Apr 21, 2026 • 34 min read
JEPA 解读

JEPA 解读

了解 JEPA（Joint Embedding Predictive Architecture），这是 Yann LeCun 提出的框
架，用于在 latent space 中实现稳定的 AI 预测，无需进行生成式解码。

  •  

admin Apr 21, 2026 • 8 min read
Software 2.0 © 2026
Powered by Ghost

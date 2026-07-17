# 资讯速览小助手

> 粘贴一个 URL，AI Agent 自动判断页面类型、提取内容、生成摘要，30 秒告诉你今天发生了什么。

## 简介

每天要跟踪 AI / 科技行业动态的人，都会面对同一个麻烦：资讯聚合站首页动辄 20+ 篇文章，从微信群收到的分享又常常分不清是列表页还是单篇文章。逐篇点开、加载、扫读是一件高频且重复的体力活。

这个项目想做的，是把"打开 → 扫读 → 总结"这件事交给一个 AI Agent：用户只粘贴一个链接，系统自动判断它是列表页还是单篇文章，逐篇抓取正文、调用大模型生成 300 字中文摘要，并实时展示处理进度与 Agent 的思考过程。

## 功能特性

- **Agent 自主决策**：基于 ReAct 模式，大模型通过 function calling 自主选择调用哪个工具，无需硬编码判断 URL 类型
- **页面类型自动识别**：列表页 → 提取全部文章；单篇文章 → 直接抓取；识别失败时自动降级
- **多级抓取降级**：`curl` → `HttpClient` → `Playwright` → 静态 HTML 兜底，覆盖静态 / 动态 / JS 渲染页面
- **多级链接提取降级**：Nuxt 内嵌 JSON → 配置选择器 → 15 个通用降级选择器 → 扫描全部链接
- **实时进度推送**：进度条 + 状态日志滚动 + 结果卡片逐个出现，单篇失败不影响整体
- **配置化扩展**：新增新闻源只需在 `appsettings.json` 中加一条选择器配置

## 技术栈

| 层级 | 技术 |
|---|---|
| 框架 | .NET 8 Blazor Server (InteractiveServer) |
| 数据库 | SQLite + EF Core 8 |
| 大模型 | DeepSeek API (`deepseek-chat`) |
| 网页抓取 | curl.exe / HttpClient / Playwright / HtmlAgilityPack |
| 实时通信 | System.Threading.Channels |

## 工作原理

```mermaid
flowchart TD
    subgraph F1["流程一：Agent 决策与执行"]
        direction TB
        A["用户输入 URL"] --> B["Agent 每轮发给大模型<br/>提示词(常驻) + 工具清单(常驻) + 对话历史"]
        B --> C["DeepSeek 大模型<br/>决定调哪个工具、填什么参数"]
        C --> D["Agent 执行被点名的工具<br/>提取列表 / 抓取正文 / 保存摘要"]
        D --> E["页面解析 & 网页抓取<br/>4层降级"]
        E -.->|"结果写入历史，进入下一轮"| B
    end

    subgraph F2["流程二：结果实时展示"]
        direction TB
        G["大模型写好摘要<br/>作为 save_summary 参数"] --> H["写入 Channel 事件管道"]
        H --> I["UI 实时消费"]
        I --> J["进度条 · 状态日志 · 结果卡片"]
    end

    D -->|"save_summary"| G

    classDef blue fill:#6366f1,stroke:#4f46e5,color:#fff;
    classDef amber fill:#f59e0b,stroke:#d97706,color:#fff;
    classDef sky fill:#0ea5e9,stroke:#0284c7,color:#fff;
    classDef green fill:#10b981,stroke:#059669,color:#fff;
    classDef gray fill:#64748b,stroke:#475569,color:#fff;
    class A,J blue;
    class B,G gray;
    class C sky;
    class D,I green;
    class E gray;
    class H amber;
```

**流程一**是 Agent 的决策闭环：每轮把写死的提示词（岗位说明书）+ 工具清单 + 对话历史一起发给 DeepSeek，大模型只负责决定下一步调哪个工具、填什么参数，Agent 执行后把结果写入历史进入下一轮，如此循环直到 `finish`。

**流程二**是结果展示：大模型在 `save_summary` 调用里直接写好摘要（作为参数填入），Agent 执行时取出 → 经 Channel 管道实时推给 UI → 进度条/日志/卡片同步更新。

**列表页完整执行顺序**：
1. 大模型根据提示词决策：先调"提取文章列表"
2. 引擎解析页面返回 8 篇文章链接
3. 大模型：「列表提取成功，逐篇处理」→ 本就需要调"抓取正文"、读完后写好摘要调"保存"
4. 循环：抓取正文(#1) → 大模型写摘要 → 保存(#1) → UI 出现卡片 #1
5. 重复直到最后一篇
6. 大模型：「全部完成」→ 调"结束任务" → Channel 关闭 → 进度 100%

### Agent 工具链

| 工具 | 功能 |
|---|---|
| `get_article_list` | 从列表页提取文章链接 |
| `get_article_content` | 抓取单篇文章标题 + 正文 |
| `save_summary` | 保存大模型摘要并推送进度到 UI |
| `finish` | 标记任务完成 |

## 快速开始

### 环境要求

- .NET 8 SDK
- curl（Windows 10+ 自带）
- DeepSeek API Key

### 配置

在 `appsettings.json` 中填入 API Key：

```json
{
  "DeepSeek": {
    "ApiKey": "your-api-key-here",
    "Model": "deepseek-chat"
  }
}
```

### 运行

```bash
cd 资讯速览小助手
dotnet run --urls http://localhost:5000
```

打开浏览器访问 `http://localhost:5000`，输入文章链接即可开始速览。

## 项目结构

```
Views (Blazor 页面)
  └── Managers (业务编排)
        └── Engines (核心引擎: Agent + NewsSource)
              └── Accessors (数据访问: DeepSeek + WebPage)
                    └── Database / Logic Contracts (数据模型)
```

| 模块 | 职责 |
|---|---|
| `AgentEngine` | ReAct 循环：调用大模型决策 → 执行工具 → 推送进度 |
| `NewsSourceEngine` | 网页解析：链接提取、正文抓取、内容清洗 |
| `WebPageAccessor` | 多通道抓取：curl → HttpClient → Playwright |
| `DeepSeekAccessor` | 大模型 API 调用封装 |
| `NewsBriefingManager` | 业务流程编排，管理 Channel 管道 |

## 核心设计取舍

**Agent 模式 vs 规则引擎**
URL 类型不确定，用规则引擎硬编码判断分支会让核心能力没用上 AI，且每加一种场景就加 if-else。把决策权交给大模型，虽然多 1-2 秒延迟，但换来自动降级与可扩展性，且思考过程对用户可见本身就是体验的一部分。

**Channel 而非 Event / 回调**
Agent 后台多轮执行，每步都要推状态到 UI。Channel 是 .NET 原生生产者-消费者管道，天然解耦、线程安全、自带背压，契合 Blazor Server 的 SignalR 长连接模型。

**抓取做 4 层降级**
实测发现猫目（maomu.com）的 TLS 配置不兼容 Windows 原生 HTTP 栈（HttpClient / Playwright 均超时），但 curl（基于 OpenSSL）可以稳定访问。因此 curl 排第一不是因为最快，而是它是唯一能稳定访问目标站点的通道。

**链接提取做 3 层降级**
核心场景（猫目等 Nuxt 站点）直接从内嵌 JSON 取数据最准；已知站点用配置选择器精确提取；未知站点用 15 个通用选择器 + 全链接扫描兜底，保证"几乎总能返回点结果"。

**逐篇处理而非批处理**
批处理快但等待期间零反馈；逐篇处理让进度条、日志、卡片实时推进，首屏约 5 秒出第一篇。进度可见性是这类产品的体验基石，30 秒的"可见进度"比 15 秒的"黑盒等待"体感更好。

**SQLite 但结果不持久化**
数据模型（任务表、结果表）已完整定义，但 MVP 阶段速览结果走内存经 Channel 推送。持久化需要配套历史列表页等 UI，先跑通核心流程，留作后续版本。

## 实测效果

| 场景 | 结果 |
|---|---|
| 输入猫目首页 | 提取 8-10 篇并生成摘要 ✅ |
| 输入猫目单篇文章 | 自动识别为单篇模式 ✅ |
| 输入不支持的站点 | 提示"暂不支持该新闻源" ✅ |
| 单篇抓取失败 | 展示标题 + 错误 + 原文链接 ✅ |
| 处理中途取消 | 已完成保留，未完成停止 ✅ |
| curl 不可用时 | 自动降级到 HttpClient / Playwright ✅ |

- 端到端耗时：8 篇约 25-35 秒
- 首屏结果：第 1 篇摘要约 5 秒内出现
- 猫目文章提取成功率 100%（Nuxt JSON 路径）
- 摘要质量：300 字以内，客观准确，无幻觉编造

## 演进路线

| 版本 | 范围 |
|---|---|
| **v1（当前）** | 单源速览（猫目）、Agent 自主决策、实时进度、4 层抓取降级 |
| **v2** | 速览历史持久化 + 历史列表页 + 多源配置扩展 |
| **v3** | 定时自动速览 + 推送通知 + 用户系统 + 个性化订阅 |

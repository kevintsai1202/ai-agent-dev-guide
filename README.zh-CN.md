[English](README.md) | [繁體中文](README.zh-TW.md) | **简体中文** | [日本語](README.ja.md)

# AI Agent Dev Guide — Claude Code 技能套件

打造 **AI agent 应用** 的 Claude Code 技能集合，涵盖从 agent 后端（可按框架替换；目前为 **Embabel + Spring AI**：GOAP / `@Agent` / JVM）到后端无关（backend-agnostic）的前端——由 LLM 生成 UI（后端产生 spec → SSE 流式传输 → 渐进渲染仪表盘）的完整全栈开发。

实证来源：`learn-embabel` 课程（后端）＋ `embabel-json-render` POC（前端）。版本基准：**Spring Boot 3.5.x + Embabel 0.4.0 + Spring AI 1.1.x**（Boot 4 尚不支持，等 Embabel 2.0）。

## 技能一览

| 技能 | 层级 | 职责 |
| ---- | ---- | ---- |
| [`ai-agent-dev-guide`](skills/ai-agent-dev-guide/SKILL.md) | **入口 / 路由** | 不确定该用哪个技能时先看这里：路由表、「该不该用 agent 框架」判断、后端框架清单、全栈开发顺序 |
| [`embabel-agent-backend`](skills/embabel-agent-backend/SKILL.md) | 后端（Embabel） | `@Agent`/`@Action`/`@AchievesGoal`、GOAP 规划、Blackboard、领域 `@Tool`、MCP/RAG、`@Condition` 守门、`@State` 循环、成本护栏、多 agent 编排、测试与可观测性 |
| [`json-render-ui`](skills/json-render-ui/SKILL.md) | 前端（后端无关） | json-render 扁平元素树 spec 契约、组件目录 ↔ registry 对账、SSE 流式传输（`fetch`+`ReadableStream`）、容错渐进渲染、后端进度/花费面板、点击下钻、目录维护页 |

三者互相交叉引用；需求模糊或跨前后端时，从 `ai-agent-dev-guide` 分派最快。前端（`json-render-ui`）也可单独搭配任何符合契约的后端使用。

## 安装

### 方式一：Claude Code Plugin（推荐）

把本 repo 作为 plugin marketplace 加入，再安装整套技能：

```text
/plugin marketplace add kevintsai1202/ai-agent-dev-guide
/plugin install ai-agent-dev-guide@ai-agent-dev-guide
```

> 安装后三个技能会自动加载，Claude 会依 `description` 在相关任务时自动触发。

### 方式二：`skills` CLI

用 [`skills`](https://github.com/vercel-labs/skills) CLI 直接从 repo 安装整套技能——它会自动发现 `skills/` 下的三个技能并接入你的 agent：

```bash
# 交互式：自行选择要装到哪些 agent 与安装范围
npx skills add kevintsai1202/ai-agent-dev-guide

# 非交互式：所有技能、所有检测到的 agent
npx skills add kevintsai1202/ai-agent-dev-guide --all

# 全局安装（用户级别，所有项目均可用）
npx skills add kevintsai1202/ai-agent-dev-guide --global
```

> 跨平台通用（macOS / Linux / Windows），无需手动复制文件。

## 目录结构

```text
ai-agent-dev-guide/
├─ .claude-plugin/
│  ├─ plugin.json          # plugin 定义
│  └─ marketplace.json     # marketplace 定义（供 /plugin marketplace add）
├─ skills/
│  ├─ ai-agent-dev-guide/       # 入口 / 路由技能
│  ├─ embabel-agent-backend/   # 后端：agent / GOAP / JVM
│  └─ json-render-ui/   # 前端：动态生成 UI / 流式渲染
└─ README.md
```

## 版本兼容性

- Embabel 最新发布版 **0.4.0**（Maven Central），构建于 **Spring Boot 3.5.x + Spring AI 1.1.x**。
- **Spring Boot 4 尚不支持**：compile 能过但 context 启动失败（`HttpHeaders.addAll` 签名变更）。Boot 4 支持排在 Embabel 2.0（等 Spring AI 2.0 GA）。
- 开工前请从目标项目的 build 文件确认实际版本，勿凭空假设版号。详见 `embabel-agent-backend` 的 Version Compatibility 段落。

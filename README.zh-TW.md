[English](README.md) | **繁體中文** | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

# AI Agent Dev Guide — Claude Code 技能套件

打造 **AI agent 應用** 的 Claude Code 技能集合，涵蓋從 agent 後端（可依框架替換；目前為 **Embabel + Spring AI**：GOAP / `@Agent` / JVM）到後端無關（backend-agnostic）的前端——由 LLM 生成 UI（後端產生 spec → SSE 串流 → 漸進渲染儀表板）的完整全棧開發。

實證來源：`learn-embabel` 課程（後端）＋ `embabel-json-render` POC（前端）。版本基準：**Spring Boot 4.1.x + Embabel 1.5.1 + Spring AI 2.0.x**（Embabel 1.5.0 起已支援 Boot 4；仍在 Boot 3.5.x 的專案請留在 Embabel 1.0.x）。

## 技能一覽

| 技能 | 層級 | 職責 |
| ---- | ---- | ---- |
| [`ai-agent-dev-guide`](skills/ai-agent-dev-guide/SKILL.md) | **入口 / 路由** | 不確定該用哪個技能時先看這裡：路由表、「該不該用 agent 框架」判斷、後端框架清單、全棧開發順序 |
| [`embabel-agent-backend`](skills/embabel-agent-backend/SKILL.md) | 後端（Embabel） | `@Agent`/`@Action`/`@AchievesGoal`、GOAP 規劃、Blackboard、領域 `@Tool`、MCP/RAG、`@Condition` 守門、`@State` 迴圈、成本護欄、多 agent 編排、測試與可觀測性 |
| [`json-render-ui`](skills/json-render-ui/SKILL.md) | 前端（後端無關） | json-render 扁平元素樹 spec 契約、元件型錄 ↔ registry 對帳、SSE 串流（`fetch`+`ReadableStream`）、容錯漸進渲染、後端進度/花費面板、點選下鑽、型錄維護頁 |
| [`webmcp-development-guide`](skills/webmcp-development-guide/SKILL.md) | Agent 對外（選用） | 把頁面能力發佈成 WebMCP 工具，讓**外部** Agent（ChatGPT Site tools / Chrome 149+）能發現並呼叫：`document.modelContext`、Imperative/Declarative、工具 schema 與生命週期、安全與 Inspector/Evals 驗證；另含 [LLM 生成 UI 的接法](skills/webmcp-development-guide/references/generative-ui-integration.md) |

四者互相交叉連結；需求模糊或跨前後端時，從 `ai-agent-dev-guide` 分派最快。前端（`json-render-ui`）也可單獨搭配任何符合契約的後端使用；`webmcp-development-guide` 為選用層，方向相反——前三者是「你蓋一個 agent」，它是「讓你的網站成為別人 agent 的工具提供者」，可疊加在任何前端上。

> [開啟 GitHub Pages 圖形化說明網站](https://kevintsai1202.github.io/ai-agent-dev-guide/)（本地入口：[index.html](docs/ai-agent-visual-guide/index.html)）：集中閱讀四個技能的 17 張概念圖解。圖解網站與技能本體分離，技能目錄只保留可安裝的 Markdown 說明。

## 安裝

### 方式一：Claude Code Plugin（推薦）

把本 repo 當作 plugin marketplace 加入，再安裝整套技能：

```text
/plugin marketplace add kevintsai1202/ai-agent-dev-guide
/plugin install ai-agent-dev-guide@ai-agent-dev-guide
```

> 安裝後四個技能會自動載入，Claude 會依 `description` 在相關任務時自動觸發。

### 方式二：`skills` CLI

用 [`skills`](https://github.com/vercel-labs/skills) CLI 直接從 repo 安裝整套技能——它會自動探索 `skills/` 底下的四個技能並接到你的 agent：

```bash
# 互動式：自行選擇要裝到哪些 agent 與安裝範圍
npx skills add kevintsai1202/ai-agent-dev-guide

# 非互動式：所有技能、所有偵測到的 agent
npx skills add kevintsai1202/ai-agent-dev-guide --all

# 全域安裝（使用者層級，所有專案皆可用）
npx skills add kevintsai1202/ai-agent-dev-guide --global
```

> 跨平台通用（macOS / Linux / Windows），不需手動複製檔案。

## 目錄結構

```text
ai-agent-dev-guide/
├─ .claude-plugin/
│  ├─ plugin.json          # plugin 定義
│  └─ marketplace.json     # marketplace 定義（供 /plugin marketplace add）
├─ skills/
│  ├─ ai-agent-dev-guide/       # 入口 / 路由技能
│  ├─ embabel-agent-backend/   # 後端：agent / GOAP / JVM
│  ├─ json-render-ui/   # 前端：動態生成 UI / 串流渲染
│  └─ webmcp-development-guide/  # Agent 對外：WebMCP 工具發佈（選用）
├─ docs/
│  └─ ai-agent-visual-guide/    # 獨立的圖形化說明網站與圖解資產
└─ README.md
```

## 版本相容性

- Embabel 最新釋出版 **1.5.1**（Maven Central，2026-08-24），建構於 **Spring Boot 4.1.0 + Spring AI 2.0.x + Jackson 3**。
- **Boot 4 自 Embabel 1.5.0 起已支援**。仍在 Spring Boot 3.5.x 的專案請留在 **Embabel 1.0.0** 線（Spring AI 1.1.7）。兩線混用仍會在 context 啟動時失敗（`HttpHeaders.addAll` 簽章變更）。
- 升 Boot 4 同時要處理：`spring-boot-starter-web` → `spring-boot-starter-webmvc`、Jackson 3（`tools.jackson.*`、`ObjectMapper` 不可變）、Spring AI 2.0 model builder 改名。
- 開工前請從目標專案的 build 檔確認實際版本，勿憑空假設版號。詳見 `embabel-agent-backend` 的 Version Compatibility 段落。

[English](README.md) | [繁體中文](README.zh-TW.md) | [简体中文](README.zh-CN.md) | **日本語**

# AI Agent Dev Guide — Claude Code スキルスイート

**AI エージェントアプリケーション** を構築するための Claude Code スキル集です。エージェントバックエンド（フレームワークごとに差し替え可能。現在は **Embabel + Spring AI**：GOAP / `@Agent` / JVM）から、バックエンド非依存（backend-agnostic）のフロントエンド——LLM が生成した UI をレンダリングするフロントエンド（バックエンドが spec を生成 → SSE ストリーミング → ダッシュボードを段階的にレンダリング）まで、フルスタックの開発をカバーします。

実証ソース：`learn-embabel` コース（バックエンド）と `embabel-json-render` POC（フロントエンド）。基準バージョン：**Spring Boot 4.1.x + Embabel 1.5.1 + Spring AI 2.0.x**（Embabel 1.5.0 以降 Boot 4 に対応。Spring Boot 3.5.x のプロジェクトは Embabel 1.0.x 系に留めること）。

## スキル一覧

| スキル | レイヤー | 役割 |
| ---- | ---- | ---- |
| [`ai-agent-dev-guide`](skills/ai-agent-dev-guide/SKILL.md) | **入口 / ルーター** | どのスキルを使うか迷ったらここから：ルーティング表、「エージェントフレームワークを使うべきか」の判断、バックエンドフレームワーク一覧、フルスタックの開発順序 |
| [`embabel-agent-backend`](skills/embabel-agent-backend/SKILL.md) | バックエンド（Embabel） | `@Agent`/`@Action`/`@AchievesGoal`、GOAP プランニング、Blackboard、ドメイン `@Tool`、MCP/RAG、`@Condition` ゲート、`@State` ループ、コストガードレール、マルチエージェント編成、テストと可観測性 |
| [`json-render-ui`](skills/json-render-ui/SKILL.md) | フロントエンド（バックエンド非依存） | json-render のフラットな要素ツリー spec 契約、コンポーネントカタログ ↔ registry の照合、SSE ストリーミング（`fetch`+`ReadableStream`）、寛容な段階的レンダリング、バックエンド進捗/コストパネル、クリックによるドリルダウン、カタログ管理ページ |
| [`webmcp-development-guide`](skills/webmcp-development-guide/SKILL.md) | エージェント公開（任意） | ページの機能を WebMCP ツールとして公開し、**外部**のエージェント（ChatGPT Site tools / Chrome 149+）が発見・呼び出せるようにします：`document.modelContext`、Imperative/Declarative、ツールスキーマとライフサイクル、セキュリティと Inspector/Evals 検証。[LLM 生成 UI との接続](skills/webmcp-development-guide/references/generative-ui-integration.md) も収録 |

4 つは相互に参照し合います。要件が曖昧な場合やフロント・バックの両方にまたがる場合は、`ai-agent-dev-guide` から振り分けるのが最短です。フロントエンド（`json-render-ui`）は、契約に準拠した任意のバックエンドと組み合わせて単独でも利用できます。`webmcp-development-guide` は任意のレイヤーで、方向が逆です——前の 3 つは「自分でエージェントを作る」もの、これは「自分のサイトを他者のエージェントのツール提供者にする」ものです。

## インストール

### 方法 1：Claude Code Plugin（推奨）

この repo を plugin marketplace として追加し、スイート全体をインストールします：

```text
/plugin marketplace add kevintsai1202/ai-agent-dev-guide
/plugin install ai-agent-dev-guide@ai-agent-dev-guide
```

> インストール後、4 つのスキルが自動的にロードされ、Claude は各スキルの `description` に基づいて関連タスクで自動的にトリガーします。

### 方法 2：`skills` CLI

[`skills`](https://github.com/vercel-labs/skills) CLI を使って repo から直接スイート全体をインストールします。`skills/` 配下の 4 つのスキルを自動検出し、エージェントに組み込みます：

```bash
# 対話形式：インストール先のエージェントとスコープを選択
npx skills add kevintsai1202/ai-agent-dev-guide

# 非対話形式：すべてのスキルを検出したすべてのエージェントへ
npx skills add kevintsai1202/ai-agent-dev-guide --all

# グローバルインストール（ユーザーレベル、すべてのプロジェクトで利用可能）
npx skills add kevintsai1202/ai-agent-dev-guide --global
```

> クロスプラットフォーム対応（macOS / Linux / Windows）で、手動でのファイルコピーは不要です。

## ディレクトリ構成

```text
ai-agent-dev-guide/
├─ .claude-plugin/
│  ├─ plugin.json          # plugin 定義
│  └─ marketplace.json     # marketplace 定義（/plugin marketplace add 用）
├─ skills/
│  ├─ ai-agent-dev-guide/       # 入口 / ルータースキル
│  ├─ embabel-agent-backend/   # バックエンド：agent / GOAP / JVM
│  ├─ json-render-ui/   # フロントエンド：動的 UI 生成 / ストリーミングレンダリング
│  └─ webmcp-development-guide/  # エージェント公開：WebMCP ツール（任意）
└─ README.md
```

## バージョン互換性

- Embabel の最新リリース版は **1.5.1**（Maven Central、2026-08-24）で、**Spring Boot 4.1.0 + Spring AI 2.0.x + Jackson 3** 上に構築されています。
- **Spring Boot 4 は Embabel 1.5.0 以降サポート済み**。Spring Boot 3.5.x のままのプロジェクトは **Embabel 1.0.0** 系（Spring AI 1.1.7）に留めてください。系統を混在させると従来どおり context 起動時に失敗します（`HttpHeaders.addAll` のシグネチャ変更）。
- Boot 4 への移行では併せて対応が必要です：`spring-boot-starter-web` → `spring-boot-starter-webmvc`、Jackson 3（`tools.jackson.*`、`ObjectMapper` の不変化）、Spring AI 2.0 のモデルビルダー改名。
- 着手前に対象プロジェクトの build ファイルから実際のバージョンを確認してください。バージョン番号を推測しないこと。詳細は `embabel-agent-backend` の Version Compatibility セクションを参照。

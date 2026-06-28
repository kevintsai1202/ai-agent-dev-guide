[English](README.md) | [繁體中文](README.zh-TW.md) | [简体中文](README.zh-CN.md) | **日本語**

# AI Agent Dev Guide — Claude Code スキルスイート

**AI エージェントアプリケーション** を構築するための Claude Code スキル集です。エージェントバックエンド（フレームワークごとに差し替え可能。現在は **Embabel + Spring AI**：GOAP / `@Agent` / JVM）から、バックエンド非依存（backend-agnostic）のフロントエンド——LLM が生成した UI をレンダリングするフロントエンド（バックエンドが spec を生成 → SSE ストリーミング → ダッシュボードを段階的にレンダリング）まで、フルスタックの開発をカバーします。

実証ソース：`learn-embabel` コース（バックエンド）と `embabel-json-render` POC（フロントエンド）。基準バージョン：**Spring Boot 3.5.x + Embabel 0.4.0 + Spring AI 1.1.x**（Spring Boot 4 はまだ未対応 — Embabel 2.0 を待つこと）。

## スキル一覧

| スキル | レイヤー | 役割 |
| ---- | ---- | ---- |
| [`ai-agent-dev-guide`](skills/ai-agent-dev-guide/SKILL.md) | **入口 / ルーター** | どのスキルを使うか迷ったらここから：ルーティング表、「エージェントフレームワークを使うべきか」の判断、バックエンドフレームワーク一覧、フルスタックの開発順序 |
| [`embabel-agent-backend`](skills/embabel-agent-backend/SKILL.md) | バックエンド（Embabel） | `@Agent`/`@Action`/`@AchievesGoal`、GOAP プランニング、Blackboard、ドメイン `@Tool`、MCP/RAG、`@Condition` ゲート、`@State` ループ、コストガードレール、マルチエージェント編成、テストと可観測性 |
| [`json-render-ui`](skills/json-render-ui/SKILL.md) | フロントエンド（バックエンド非依存） | json-render のフラットな要素ツリー spec 契約、コンポーネントカタログ ↔ registry の照合、SSE ストリーミング（`fetch`+`ReadableStream`）、寛容な段階的レンダリング、バックエンド進捗/コストパネル、クリックによるドリルダウン、カタログ管理ページ |

3 つは相互に参照し合います。要件が曖昧な場合やフロント・バックの両方にまたがる場合は、`ai-agent-dev-guide` から振り分けるのが最短です。フロントエンド（`json-render-ui`）は、契約に準拠した任意のバックエンドと組み合わせて単独でも利用できます。

## インストール

### 方法 1：Claude Code Plugin（推奨）

この repo を plugin marketplace として追加し、スイート全体をインストールします：

```text
/plugin marketplace add kevintsai1202/ai-agent-dev-guide
/plugin install ai-agent-dev-guide@ai-agent-dev-guide
```

> インストール後、3 つのスキルが自動的にロードされ、Claude は各スキルの `description` に基づいて関連タスクで自動的にトリガーします。

### 方法 2：`skills` CLI

[`skills`](https://github.com/vercel-labs/skills) CLI を使って repo から直接スイート全体をインストールします。`skills/` 配下の 3 つのスキルを自動検出し、エージェントに組み込みます：

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
│  └─ json-render-ui/   # フロントエンド：動的 UI 生成 / ストリーミングレンダリング
└─ README.md
```

## バージョン互換性

- Embabel の最新リリース版は **0.4.0**（Maven Central）で、**Spring Boot 3.5.x + Spring AI 1.1.x** 上に構築されています。
- **Spring Boot 4 は未対応**：コンパイルは通りますが context の起動に失敗します（`HttpHeaders.addAll` のシグネチャ変更）。Boot 4 対応は Embabel 2.0 に予定されています（Spring AI 2.0 GA 待ち）。
- 着手前に対象プロジェクトの build ファイルから実際のバージョンを確認してください。バージョン番号を推測しないこと。詳細は `embabel-agent-backend` の Version Compatibility セクションを参照。

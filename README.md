# 概要

このボイラープレートは Codex の性能を最大限活かして、AI エージェントが下記を意識して自走できるように作ったものです。

- 複雑なタスクは長時間コーディングができる
- アクセシビリティを意識したデザイン・コーディングを行う
- 自分好みのフレームワークやライブラリを選択させる

# ファイル説明

## ディレクトリ構造

```
codex-coding-boilerplate/
├── .agents/              # AIエージェント向けの設定・ドキュメント
│   ├── docs/            # 開発ガイドライン・ルール
│   │   ├── Libraries.md          # 利用ライブラリの記録と追加ガイド
│   │   ├── NextjsMCPDebug.md     # Next.js デバッグ方法のガイド
│   │   ├── UICoding.md           # UIコーディングルール（アクセシビリティ・UX）
│   │   ├── UIDesign.md           # UIデザインルール（Vercelデザインガイドライン準拠）
│   │   └── VercelWorkflow.md     # Vercel Workflow を使用したバックエンド実装ガイド
│   ├── ExecPlans/       # ExecPlanファイルを格納するディレクトリ
│   └── PLANS.md         # ExecPlanの作成・実行方法の詳細ガイド
├── AGENTS.md            # AIエージェント向けの基本設定とルール
├── long-task-prompts-example.md  # 長期的タスク実行時のプロンプト例
├── MCP-example.md       # MCPサーバー設定例（config.toml / mcp.json）
├── ARCHITECTURE-example.md       # プロダクト要件定義（PRD）の例とワークフロー設計
└── README.md            # 本ファイル（プロジェクト概要と使い方）
```

# 使い方

## ARCHITECTURE の準備

ARCHITECTURE-example.md を参考に、ARCHITECTURE.md を作成してください。
※ Claude Code を利用する場合は、AGENTS.md をコピーして、CLAUDE.md を作成してください。

```
cp AGENTS.md CLAUDE.md
```

## 環境構築 の開始

```
ARCHITECTURE.mdを確認して、0からの環境構築するための ExecPlan を作成してください。
```

## MCP の準備

MCP-example.md を確認して、エージェントに合わせた MCP 設定を行ってください。

# Vibe Coding の仕方

## 小さなタスク

特に意識することはないです。
通常通り、ファイルのメンション等を用いて、エージェントに指示を出してください。

## 長期的のタスク実行

長時間のタスクを long-task-prompts-example.md を参考にして、ExecPlan を明示的に実行してください。
※ 指示がなくても、複雑な設計や大規模な作業であれば、エージェントは自動的に ExecPlan を実行します。

# おすすめの開発フロー

1.実装
小さなタスクは Cursor の Agents で、長期的なタスクは Codex, Claude Code で実装します。

2.ローカルレビュー
Codex CLI を利用して、`/review`コマンドを実行します。

3.PR レビュー
Codex で Codex review を走らせます。

4.マージ

# TIps

## どの粒度で Exec プランを作るか迷ったとき

まず環境構築・ライブラリ設定をお願いする

```
@ARCHITECTURE.md を読んで、最小限プロダクトを作るための環境構築をするExacPlanを作成してください。
まずは各フレームワークのインストールとライブラリのインストール、設定まででよいです。プロダクトを作り始める必要はありません。
```

ExacPlan の粒度と順番のアドバイスをもらう

```
@ARCHITECTURE.md を確認して、最小版ゴールを達成するためにどの粒度、順番でExacPlanを作成していくべきかアドバイスをください。
```

# 参考

- https://vercel.com/design/guidelines
- https://www.youtube.com/watch?v=Gr41tYOzE20

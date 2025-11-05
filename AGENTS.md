<!--
codex: true
cursor: false
mcp: true
-->

ユーザーへの説明は日本語で行ってください。

# Docs

- 新規にフレームワーク・ライブラリを追加する際は、.agent/Libraries.md を確認してください。追加したあとは、利用中のライブラリに追記してください
- 新規の UI を作成する際は、 .agent/UICoding.md と .agent/UIDesign.md のルールを確認してください
- Next.js に関するデバッグを行う際は、.agent/NextjsMCPDebug.md を確認し、適切な Nextjs のデバッグ方法を確認してください
- キュー、マイクロサービス、Kafka、SQS、複雑な AI 処理など特殊なバックエンドが必要な場合は、.agent/VercelWorkflow.md を確認し、適切な実装ができるか検証してください

# ExecPlans

複雑な機能や大規模なリファクタリングを行う際は、設計から実装まで .agent/PLANS.md に記載されている ExecPlan を使用してください。

# MCP Servers

- 利用シーンの目安:
  - `next-devtools`: Next.js 実装をコードベースから直接調査・補助したいときに使用します。`next dev` 実行中のアプリに接続し、ルーティングやサーバーアクションの解析を行う際に有効です。
  - `shadcn`: UI コンポーネントを素早く生成したいときや、shadcn/ui の設計思想に沿ったパターンを確認したいときに呼び出します。生成したコードは `.agent/UICoding.md` と `.agent/UIDesign.md` のルールに沿って整えてください。
  - `ai-elements`: 既存の Vercel の AI Elements に登録されているコンポーネントの情報を知りたい時や、レイアウト構築、スタイル実装に関する情報を得たい時に利用します。
  - `chrome-devtools`: ブラウザ上で再現するバグやパフォーマンス問題を調査するときに使います。また、UI を視覚的に評価する場合は、スクリーンショットを撮って現在の UI/UX を確認できます。

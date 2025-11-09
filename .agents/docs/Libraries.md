# 利用するライブラリ・フレームワーク

下記は必要になった場合に順次用意する。セットアップ時は最小限の構成を心がける。
Vercel 製の product は進化が早いので、最新ドキュメントを参照するように心がける

- 基本となる構成
  - node 管理: nodenv
  - 言語: Typescript
  - フレームワーク: Nextjs 16 (with "next-devtools-mcp")
  - バリデーション: Zod
  - CSS: tailwind css
  - component: shadcn(use mcp), Vercel AI Elements(use mcp)
  - icon: lucide-react
  - ホスティング: Vercel
- 必要があれば順次追加(ユーザーの許可が必要)
  - AI・LLM: Vercel AI SDK
  - LLM model: with Vercel AI Gateway
    - gemini-2.5-flash-lite(汎用タスク)
    - grok-4-fast-non-reasoning(少し高度なタスク)
  - DB: Turso
  - ORM: Drizzle
  - Auth: Better Auth(Auth.js に近いシンプルなもの)
  - メール配信: Resend
  - 状態管理: Zustand（useContext で十分な場合は必要なし）
  - デバッグ: Playwright MCP + next-devtools-mcp
  - テスト: Vitest 4 ユニットテスト/単体テスト
  - ブラウザテスト: Vitest 4 + Playwright
  - Backend(キュー、マイクロサービス、Kafka、SQS、重い AI 処理): vercel workflow
  - cron: Vercel cron jobs

# 利用中のライブラリ・フレームワーク

-

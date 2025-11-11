ExecPlan（自動ワークフロー前半）の md ファイルを作成してください。
ExecPlan の作り方は @.agents/PLANS.md に準拠してください。
全体のプランは @ARCHITECTURE.md で確認してください。

## 概要

app/api/image-recommendation/route.ts  と  app/api/generate-plan/route.ts  を 実装し、LLM 呼び出しや Wikimedia 連携を行う。RecommendedImageSchema  などを  src/schemas/workflows.ts  にまとめ、レスポンスをチャットから呼べるようにする。

## BackEnd について

- Vercel AI Gateway は新しい技術なので、Web 検索してシンプルな実装ができるよう調査してください。
- AI_GATEWAY_API_KEY のみで、様々なプロバイダーに接続できます。
- Vercel AI Gateway の gateway プロバイダーを利用してください。
- モデルは google/gemini-2.5-flash-lite を利用してください。

## UI について

- MCP で AI elements を調べ、利用できるものは最大限活用してください。
- ユーザーが Config を設定し、プロンプトを入力し、サブミットを押した時にワークフローを開始するようにしてください。
- ワークフローが開始したら、作業中であることをユーザーに示してください。
- その過程を AI elements を利用して、codex のような作業履歴を表示してください。 プロンプトを入力するときは、blueprint を毎回リセットしてください。
- 一度プロンプトを入力したら、入力は disable にしてください。
- plan と image のレスポンスが返ってきたら、AI elements の plan コンポーネントを表示してユーザーに示してください。

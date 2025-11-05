# 概要

これは、Next.js と Vercel AI SDK を利用した。自動動画生成サービスです。

# ゴール

このサービスのゴールは ユーザーと AI エージェントが協力して、Generate Video API に適切な JSON を送信することが目的です。
各ワークフローのステップ毎にエージェントが連動して、ChatUI 側が管理するシングルトンの動画生成用 JSON（React state + localStorage を単一の真実の源とする）を完成させます。ステップ毎に最小限のコンテキストを渡して構造化アイテムを生成し、クライアントの merge 関数で統合します。

## 最小版のゴールについて

まず最小版を作る段階では、認証やデータベースは必要ありません。データを保存する場合は、基本的にはローカルストレージ等を利用するべきです。
また、ローカルストレージを利用する際には、のちほどデータベースへの保存に付け替えることを考慮して抽象化するべきです。

今後ユーザーが会話を編集したり、画像を手動で検索、選択したりする UI(ChatGPT の Canvas のようなもの)が必要になると思いますが、現段階では必要ないです。
ChatUI ベースでエージェントが自走して、json オブジェクトを完成させることをまずゴールとしてください。

# 譲れない条件

LLM が全体のオーケストレーションすると、毎回 json 全体のレビューが必要になり、コンテキストを食うので、利用しません。
それぞれのワークフローのエージェントは全体のコンテキストは気にせず、最低限のコンテキストを受け取り、最小限の構造化オブジェクトを返して終了すべきです。
そして、生成された構造化オブジェクトはクライアント（ChatUI）が持つシンプルな差し込み関数によって、ローカルのシングルトン JSON にマージされるべきです。Edge Runtime API は純粋関数として振る舞い、サーバー側では状態を保持しません。

# UI/UX について

基本的には高品質な AI サービスとして実装できるよう、AI に関する部分は Vercel AI SDK を最大限活用します。他のパーツについても shadcn/ui を用いて高品質なコンポーネント作りを意識します。

- input や approve,workflow の進捗表示灯があるので、基本的な json 生成フローは Codex のようなエージェントサービスのような UX にすべきです。

# 各ワークフローの紹介

- ユーザー入力
  - 1.setting-config
  - 2.input-prompt
- エージェント自動
  - 3-1.image-recommendation
  - 3-2.generate-plan
- ユーザー承認
  - 4.plan-decision(approval | reject)
- エージェント自動
  - 5.generate-titles
  - 6.generate-overviews
  - 7.generate-conversations
    - 7-1.generate-intro
    - 7-2.generate-conversations(overview 毎に生成)
    - 7-3.generate-outro
- ユーザー承認/実行
  - 8.generate-video-decision（動画生成の実行/中止）

# 各ワークフローについての説明

## 1.setting-config

### 概要

ユーザーが各オプションを選択する。AI Elements の chat input の tool select でこれらの要素を設定し、動画生成リクエストの土台を作る

### schema

```
export const ConfigSchema = z.object({
  background: z.enum(["room", "park", "sky", "winter", "office", "sahel"]),
  character: z.enum(["girl", "boy"]),
  voice: z.enum(["annie", "kaon", "tsukuyomi", "zundamon"]),
});
```

### API

処理なし。
ChatUI 側の差し込み関数でシングルトン JSON の `config` ノードを更新します。

## 2.input-prompt

### 概要

ユーザーが AI Elements の chat input で作りたい動画のイメージを送ります。「大谷翔平」「コアラ」などの固有名詞、「今週の芸能ゴシップ」などの抽象的なお題、「youtube の文字起こし」などの動画制作に役立つ雑多なコンテキストなどが送られる可能性があります。

### schema

string

### API

処理なし（3-1.image-recommendation と 3-2.generate-plan に情報を渡す）。
ChatUI 側の差し込み関数でシングルトン JSON の `prompt` を更新します。

## 3-1.image-recommendation

### 概要

ユーザーが `input-prompt` で提供した内容を起点に、最大 `MAX_CANDIDATES` (=40) 件まで Wikimedia Commons から候補画像を収集し、Gemini で 0-100 点のスコアリングを行います。`RECOMMENDATION_SCORE_THRESHOLD` (=80) を超える画像が `MIN_RECOMMENDED_COUNT` (=10) 枚以上あれば実行可能 (`feasible=true`) とみなし、UI に進行可否を返します。候補とスコアを安定して突き合わせるため、すべての後続ステップは `candidateId` をキーに利用します。メタ情報（title / author / license など）は `ImageCandidateSchema` に集約し、推奨結果からは `candidateId` を介して再利用します。

### schema

```
export const RECOMMENDATION_SCORE_THRESHOLD = 80;
export const MIN_RECOMMENDED_COUNT = 10;
export const MAX_CANDIDATES = 40;

export const ImageCandidateSchema = z.object({
  id: z.string(),
  title: z.string(),
  url: z.string(),
  description: z.string().optional(),
  author: z.string().optional(),
  license: z.string().optional(),
  thumb: z.string(),
});

export const RankedImageSchema = z.object({
  candidateId: z.string(),
  url: z.string(),
  score: z.number(),
  reasoning: z.string().optional(),
});

export const ImageRecommendationResponseSchema = z.object({
  feasible: z.boolean(),
  recommendedCount: z.number().int().nonnegative(),
  recommended: z.array(RankedImageSchema).optional(),
  all: z.array(RankedImageSchema).optional(),
  message: z.string().optional(),
});
```

### API

- `POST /api/image-recommendation` (Edge Runtime)
- リクエストボディは `userPrompt` を受け取り、LLM で検索キーワードを決定して Wikimedia Commons API を叩く。収集件数は常に `MAX_CANDIDATES` 以下に制限する。
- Edge 側ではメタデータの正規化とスコアリングのみを行い、画像のサムネイル変換は行わない。フロントは `<img>` の `sizes/srcset` もしくは `/_next/image` などの外部最適化エンドポイントを使う。
- Gemini で算出したスコアから `recommended` を構築し、降順に整列させる。同時に `recommendedCount` と `feasible` を計算する。
- レスポンスは純粋関数として返し、ChatUI の差し込み関数がシングルトン JSON の `images` ノードにマージする。UI は `feasible` / `recommendedCount` のみを参照し、その他の状態はログ用途に留める。

### プロンプト例

```
あなたは検索キーワードに最適な画像を選定するクリエイティブディレクターです。

## 検索キーワード
{{searchQuery}}

## 候補画像一覧
{{#each images}}
- 画像{{@indexPlusOne}}: {{title}}
  - 説明: {{description || "説明なし"}}
  - 作者: {{author || "不明"}}
  - ライセンス: {{license || "不明"}}
  - ID: {{id}}
{{/each}}

## タスク
各画像について検索キーワードとの適合度を 0-100 点で評価し、配列で返してください。
スコア 80 点以上の画像はユーザーに提示する推奨候補として扱われます。

## 出力形式
{
  "ranked_images": [
    { "candidateId": "img-0-0", "score": 95, "reasoning": "評価理由（日本語）" }
  ]
}
```

## 3-2.generate-plan

### 概要

ユーザーが input prompt に対して、どのような動画を制作すべきかの実行プランを立てます。「プラン内容」「再生回数を稼げるポイント」「ユーザーを惹きつけるポイント」について明確に提示します。
エージェントは生成した plan を ChatUI のシングルトン JSON に差し込むための純粋データとして返します。

### schema

```
export const PlanSectionSchema = z.object({
  id: z.string(),
  title: z.string(),
  synopsis: z.string().min(60, "Must be at least 60 characters"),
  viewerBenefit: z.string().min(30, "Must be at least 30 characters"),
  keyMoments: z.array(z.string()).min(1).max(4),
});

export const PlanSchema = z.object({
  summary: z.string().min(80, "Must be at least 80 characters"),
  viralAngles: z
    .array(z.string().min(20, "Must be at least 20 characters"))
    .min(2),
  engagementIdeas: z
    .array(z.string().min(20, "Must be at least 20 characters"))
    .min(2),
  sections: z.array(PlanSectionSchema).min(3),
});

export const PlanAIResponseSchema = z.object({
  plan: PlanSchema,
  caveats: z.string().optional(),
});
```

### API

- `POST /api/generate-plan` (Edge Runtime)
- リクエストボディは `userPrompt`（input prompt の原文）、`config`（ConfigSchema から必要なフィールドを抽出したオブジェクト）、`imageStatus`（image-recommendation の結果サマリー。例: `{ feasible: true, recommendedCount: 12 }`）、`trendHints`（任意の市場データやキーワード補足）、`constraints`（運営上の制約など任意）を含む想定
- LLM には `PlanAIResponseSchema` を指定し、`plan.summary` を後続ワークフローで共有する `planSummary` として扱う。`plan.sections` の `id` は generate-titles → generate-overviews → generate-conversations が利用できるよう固定キーとして必ず発行する。
- 生成結果は純粋データとして返し、ChatUI の差し込み関数がシングルトン JSON の `plan` ノードを更新する。`viralAngles` や `engagementIdeas` は plan-decision UI に表示し、`sections` 情報は titles 生成の初期テンプレートに渡す。
- `caveats` が存在する場合は警告メッセージとして保存し、plan-decision で注意喚起できるようにする。

### プロンプト例

```
あなたはYouTube動画の企画プロデューサーです。
以下の情報をもとに、実行可能性の高い動画プランを作成してください。

## ユーザー要望
{{userPrompt}}

## 動画設定
{{config}}

{{#if imageStatus}}
## 画像調達状況
- 実行可能フラグ: {{imageStatus.feasible}}
- 推奨画像枚数: {{imageStatus.recommendedCount}}
- メモ: {{imageStatus.note}}
{{/if}}

{{#if trendHints}}
## トレンド・補足情報
{{trendHints}}
{{/if}}

{{#if constraints}}
## 制約条件
{{constraints}}
{{/if}}

## 出力要件
- `summary` では動画の狙いと構成方針を80文字以上で説明する
- `viralAngles` はバズを狙える切り口を最低2つ提示する
- `engagementIdeas` は視聴維持やCTAにつながる工夫を最低2つ提示する
- `sections` は3〜5件。`synopsis` に詳細、`viewerBenefit` に視聴者メリット、`keyMoments` に演出や映像のヒントを列挙する
- セクションの順序はストーリー性や学習効果が最大化される並びにする

## 出力形式
{
  "plan": {
    "summary": "...",
    "viralAngles": ["..."],
    "engagementIdeas": ["..."],
    "sections": [
      {
        "id": "section-1",
        "title": "セクションタイトル",
        "synopsis": "60文字以上の詳細説明",
        "viewerBenefit": "視聴者のメリット",
        "keyMoments": ["演出案1", "演出案2"]
      }
    ]
  },
  "caveats": "注意事項があればここに記載"
}
```

## 4.plan-decision

### 概要

ユーザーは出力された「プラン」と、推奨画像一覧のハイライト（最低 3 件）に対して「進める」「やめる」を選択できます。`feasible === false` や `recommendedCount < MIN_RECOMMENDED_COUNT` の場合は、選択ボタンの上に警告を表示しつつも強行実行できるようにします。

### API

処理なし。フロントエンド（ChatUI）がシングルトン JSON の `plan`・`images` を参照し、UI イベントに応じて次のワークフローに進むかを決定します。

## 5.generate-titles

### 概要

全体の Plan を参考にして、ユーザーを惹きつける動画タイトルと、各セクションのタイトルを作成します。生成結果は ChatUI の差し込み関数でシングルトン JSON の `titles` ノードにマージします。

### schema

```
export const TitleSchema = z.string().max(22);

export const SectionTitleItemSchema = z.object({
  id: z.string(),
  title: z
    .string()
    .min(5, "Section title must be at least 5 characters")
    .max(30, "Section title must be at most 30 characters"),
});

export const TitlesAIResponseSchema = z.object({
  title: TitleSchema,
  sectionTitles: z
    .array(SectionTitleItemSchema)
    .min(3, "Provide at least 3 section titles"),
});
```

### API

- `POST /api/generate-titles` (Edge Runtime)
- リクエストボディは `planSummary`（generate-plan の出力全文）、`userPrompt`（input prompt そのまま）、`searchContext`（webSearch が on の場合のみ追加）、`previousTitles`（再生成時に既存候補を避けるための配列、任意）などを含むオブジェクトを想定
- Gemini に `TitlesAIResponseSchema` を指定し、`sectionTitles` の各要素に plan.sections の `id` をそのまま引き継ぐ。id が欠落しないようプロンプト内で明示する。
- 生成結果は純粋データとして返し、ChatUI がシングルトン JSON の `titles` スロットに保存する。後続の generate-overviews が `id` ベースでセクションを特定できるようにする。

### プロンプト例

```
あなたは動画チャンネルのリードエディターです。以下の情報をもとに、
視聴者が思わずクリックしたくなるメインタイトルと、ストーリーを分解する
セクションタイトル（最低3個、最大6個）を作成してください。`sectionTitles[i].id`
には plan.sections[i].id をそのまま使用してください。

## ユーザー要望
{{userPrompt}}

## プラン要約
{{planSummary}}

{{#if searchContext}}
## 追加リサーチ結果
{{searchContext}}
{{/if}}

{{#if previousTitles}}
## 回避したいタイトル候補
{{previousTitles}}
{{/if}}

## 制約
- メインタイトルは22文字以内かつ強いフックを入れる
- セクションタイトルは視聴者が流れを理解できる順番に並べる
- セクションタイトルはそれぞれ 5 文字以上、30 文字以内
- dramatic の場合はナショジオ風にドラマチック / normal は分かりやすく / webSearch は最新情報を反映
- 同じ語尾・語彙の連続を避ける

## 出力形式
{
  "title": "視聴者を惹きつけるメインタイトル",
  "sectionTitles": [
    { "id": "section-1", "title": "第一章となるセクションタイトル" },
    { "id": "section-2", "title": "第二章となるセクションタイトル" },
    { "id": "section-3", "title": "第三章となるセクションタイトル" }
  ]
}
```

## 6.generate-overviews

### 概要

全体の Plan とエージェントが作成した titles を参考にして、各 sectionTitle ごとに Overview を作成します。各 Overview はセクション ID を保持し、generate-conversations が id で対象を解決できるようにします。

### schema

```
export const OverviewSchema = z.object({
  id: z.string(),
  title: z.string(),
  summaries: z.array(z.string().min(50, "Must be at least 50 characters")),
});
export const OverviewsSchema = z.array(OverviewSchema);

export const OverviewsAIResponseSchema = z.object({
  items: z.array(OverviewSchema),
  changes: z.string().optional(),
});
```

### API

- `POST /api/generate-overviews` (Edge Runtime)
- リクエストボディは `title`（generate-titles のメインタイトル）、`sectionTitles`（同 API の配列。各要素は `{ id, title }`）、`planSummary`（generate-plan の出力全文）、`videoType`・`mode`（Config から継承）、`referenceImages`（image-recommendation の上位候補の URL 群、任意）、`searchContext`（webSearch 時の追加情報、任意）などを含むオブジェクトを想定
- `sectionTitles` の順序と `id` を保持しながら Overview を Gemini で生成し、`OverviewsAIResponseSchema.items` に格納する。
- `summaries` にはセクションの骨子となる 2-3 段落を配列で格納し、後続の会話生成エージェントが引用しやすいよう 50 文字以上の要約を要求する。
- レスポンスは純粋データとして返し、ChatUI が `overviews` ブロックに保存する。generate-conversations の入力で id をキーにマッピングする。

### プロンプト例

```
あなたは動画構成の専門編集者です。以下の情報をもとに、
各セクションタイトルに対応した詳細な概要を作成してください。

## メインタイトル
{{title}}

## セクション一覧
{{#each sectionTitles}}
- {{this.id}}: {{this.title}}
{{/each}}

## プラン要約
{{planSummary}}

{{#if searchContext}}
## 追加リサーチ結果
{{searchContext}}
{{/if}}

## 制約
- 各セクションにつき 1 つのオブジェクトを生成
- `summaries` には 50〜120 文字程度の要約文を 2 件まで格納（1件でも可）
- 事実関係はプランおよび追加リサーチに基づき、誇張しない
- 視聴者の好奇心を刺激する語彙を織り交ぜるが、煽りすぎない
- sectionTitles の順番と `id` を必ず維持

## 出力形式
{
  "items": [
    {
      "id": "section-1",
      "title": "セクションタイトル 1",
      "summaries": [
        "要約文その1（50文字以上）",
        "必要なら要約文その2"
      ]
    }
  ]
}
```

## 7.generate-conversations

### 概要

全体の Plan とエージェントが作成した overview を参考にして、intro / 各セクション / outro の台本を並列生成します。並列生成であっても `partId` と `order` を必須化し、最終的に intro → section 順 → outro の並びを安定して再構築します。生成結果は ChatUI のシングルトン JSON の `conversations` にマージされます。

### schema

```
export const ConversationSchema = z.object({
  id: z.string().optional(),
  text: z.string(),
  type: z.enum(["intro", "text", "title", "outro"]),
  emotion: z.enum(["happy", "normal", "sad", "question"]),
  speakerId: z.number(),
  imageUrl: z.string().optional(),
  partId: z.string(), // "intro" | "outro" | sectionId
  order: z.number().int(),
});
export const ConversationsAIResponseSchema = z.object({
  items: z.array(ConversationSchema),
});

export const VoiceSchema = z.enum(["annie", "kaon", "tsukuyomi", "zundamon"]);
export const VOICE_MAP: Record<z.infer<typeof VoiceSchema>, number> = {
  annie: 1,
  kaon: 2,
  tsukuyomi: 3,
  zundamon: 4,
}; // 実際の ID は実装側で確定させる
```

### API

- `POST /api/generate-conversations` (Edge Runtime)
- リクエストボディは `planSummary`、`titles`（generate-titles の出力）、`overviews`（generate-overviews の配列）、`voice`（Config の選択）、`part`（"intro" / "outro" / セクションの `id`）、`styleOverrides`（任意の口調調整）を想定
- `speakerId` は必ず `VOICE_MAP[voice]` を使用する。voice が未選択の場合は Config ステップに戻ってもらう。
- intro / outro は全体を俯瞰した導入・締めを作成し、セクションごとのリクエストでは該当 overview の情報だけを参照する。各レスポンスには `partId` と `order` を必ず含め、UI は `partId` → `order` のキーで整列する。
- レスポンスは純粋データとして返し、ChatUI が `conversations.items` をマージする。空配列が返った場合は再生成を促す。

### プロンプト例

```
あなたはYouTubeナレーション作成のスペシャリストです。
以下の情報をもとに、指定されたパートの台本を作成してください。

## メインタイトル
{{titles.title}}

## セクションタイトル一覧
{{#each titles.sectionTitles}}
- {{this.id}}: {{this.title}}
{{/each}}

## 対象パート
{{part}}  // "intro" | "outro" | sectionId

{{#if isSectionPart}}
## 対象セクション概要
タイトル: {{targetOverview.title}}
要約:
{{#each targetOverview.summaries}}
- {{this}}
{{/each}}
{{/if}}

## 全体プラン
{{planSummary}}

## ボイス設定
選択音声: {{voice}}  // annie | kaon | tsukuyomi | zundamon
必ず `speakerId = VOICE_MAP[voice]` を使用すること。

{{#if styleOverrides}}
## スタイル調整
{{styleOverrides}}
{{/if}}

## 制約
- 各 `items` は 1 文ごとに分割
- `type` は intro, title, text, outro のいずれか
- `partId` に対象パートの ID を設定し、`order` は 0 からの連番とする
- `emotion` はシーンに応じて happy / normal / sad / question から選択
- 不要なフィールドを追加しない

## 出力形式
{
  "items": [
    {
      "text": "視聴者を惹きつける導入文",
      "type": "intro",
      "emotion": "happy",
      "speakerId": 1,
      "partId": "intro",
      "order": 0
    }
  ]
}
```

## 8.generate-video-decision（動画生成の実行/中止）

### 概要

ユーザーはシングルトンの JSON プレビューを確認し、「動画生成」「やめる」を選択します。進めるを選択した場合のみ generate-video API にリクエストを送ります。やめるを選択した場合は、その時点でワークフローを終了します。

### 実行前チェック

- `config` が存在し、必要なフィールドが埋まっていること
- `title` が存在し、22 文字以内の制約を満たしていること
- `conversations.items.length > 0` であり、すべての `partId` / `order` が揃っていること
- `images.recommendedCount >= MIN_RECOMMENDED_COUNT`。未満の場合は警告表示と二重確認の上でのみ送信すること

### API

- `POST /api/generate-video`
- ChatUI はシングルトン JSON から `config`, `title`, `conversations`, `images` を抽出し、合わせて `attributions: Array<{ title: string; author?: string; license?: string; url: string }>` を組み立てて送信する。`attributions` は推奨画像のメタ情報から `candidateId` で join して生成し、動画のクレジット / 概要欄に再利用できるようにする。
- `voice` の情報は `config.voice` に含まれている前提だが、`speakerId` が `VOICE_MAP` に基づいているかを最終チェックする。
- API は純粋データを返すのみで、状態は保持しない。送信後の結果（ジョブ ID 等）は ChatUI が受け取り、必要に応じてローカル状態に追記する。

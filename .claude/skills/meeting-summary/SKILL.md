---
name: meeting-summary
description: 指定チャネルの会話を分析し、商談サマリーをLINEに配信する。「商談サマリー」「ミーティングまとめ」等のキーワードで発動。
allowed-tools: mcp__claude_ai_DRM_AGENT__list-channels mcp__claude_ai_DRM_AGENT__list-messages mcp__claude_ai_DRM_AGENT__search-messages mcp__claude_ai_DRM_AGENT__get-channel mcp__claude_ai_DRM_AGENT__list-connect-users mcp__claude_ai_DRM_AGENT__send-line-message
---

# 商談サマリー配信

指定されたチャネルの会話履歴を分析し、構造化された商談サマリーをLINEユーザーに配信する。

## 引数

`$ARGUMENTS` は以下の形式で受け取る:

- `[チャネル名またはキーワード]` — 対象チャネルを特定するための検索ワード
- `[チャネル名] [期間]` — 期間指定あり（例: `ABC商事 直近1週間`、`田中さん 3日`）
- 引数なしの場合 — ユーザーにチャネルと送信先を確認する

## 実行手順

### Step 1: 対象チャネルの特定

1. `list-channels` でチャネル一覧を取得（type: "LINE" でフィルタ）
2. `$ARGUMENTS` のキーワードでチャネルを絞り込む
3. 候補が複数ある場合はユーザーに選択を求める

### Step 2: 会話履歴の取得

1. `list-messages` で対象チャネルのメッセージを取得
   - 期間指定がある場合: `from` / `to` パラメータで絞り込む
   - 期間指定がない場合: 直近7日間をデフォルトとする
2. メッセージが多い場合はカーソルページネーションで追加取得

### Step 3: 商談内容の分析

取得した会話から以下を抽出・整理する:

1. **商談概要**: 誰と何について話していたか
2. **決定事項**: 合意された内容、確定した事項
3. **未解決事項**: 返答待ち、検討中、保留になっている事項
4. **ネクストアクション**: 次にやるべきこと（抽象禁止 — 具体的な行動・文面・期限まで落とし込む）
5. **相手の温度感**: 関心度・懸念点・好反応だったポイント

### Step 4: 送信先の特定

1. `list-connect-users` でLINEユーザー一覧を取得（type: "LINE"）
2. 送信先をユーザーに確認する（**必ず確認してから送信すること**）

### Step 5: LINEへ配信

`send-line-message` でFlexメッセージとして送信する。

#### メッセージフォーマット

```json
{
  "type": "flex",
  "altText": "商談サマリー: [チャネル名]",
  "contents": {
    "type": "bubble",
    "size": "giga",
    "header": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        { "type": "text", "text": "商談サマリー", "weight": "bold", "size": "lg", "color": "#1a1a1a" },
        { "type": "text", "text": "[チャネル名] | [期間]", "size": "xs", "color": "#999999" }
      ]
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "spacing": "md",
      "contents": [
        { "type": "text", "text": "決定事項", "weight": "bold", "size": "sm", "color": "#1a1a1a" },
        { "type": "text", "text": "・...", "size": "sm", "wrap": true, "color": "#333333" },
        { "type": "separator", "margin": "md" },
        { "type": "text", "text": "未解決事項", "weight": "bold", "size": "sm", "color": "#1a1a1a" },
        { "type": "text", "text": "・...", "size": "sm", "wrap": true, "color": "#333333" },
        { "type": "separator", "margin": "md" },
        { "type": "text", "text": "ネクストアクション", "weight": "bold", "size": "sm", "color": "#1a1a1a" },
        { "type": "text", "text": "・...", "size": "sm", "wrap": true, "color": "#333333" }
      ]
    }
  }
}
```

## LLM出力ポリシー（厳守）

- **TODOは抽象禁止**: 「フォローアップする」ではなく「4/8（火）までに〇〇の件でメールを送る。件名案: 〜」レベルまで具体化
- **フィードバックは抽象禁止**: 「もっと深掘りすべき」ではなく「次回は『御社の〇〇についてもう少し詳しく教えていただけますか？』と聞く」
- **LINE表示前提**: 箇条書き中心、1項目2行以内、情報密度を高く
- **温度感の言語化**: 「好反応」「慎重」「要注意」など、次回のアプローチ判断に使える表現で

## 注意事項

- 送信前に必ずユーザーに内容と送信先を確認すること
- 長文になる場合は複数バブル（carousel）に分割する
- メッセージは最大5件まで。超える場合は要約して収める

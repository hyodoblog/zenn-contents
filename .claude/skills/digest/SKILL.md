---
name: digest
description: 日次または週次のダイジェストを生成し、LINEに配信する。「ダイジェスト」「日報」「週報」「まとめ」等のキーワードで発動。
allowed-tools: mcp__claude_ai_DRM_AGENT__list-channels mcp__claude_ai_DRM_AGENT__list-messages mcp__claude_ai_DRM_AGENT__search-messages mcp__claude_ai_DRM_AGENT__get-channel mcp__claude_ai_DRM_AGENT__list-connect-users mcp__claude_ai_DRM_AGENT__send-line-message
---

# 日次/週次ダイジェスト配信

複数チャネルの活動を横断的にまとめ、ハイライトと要対応事項をLINEに配信する。

## 引数

`$ARGUMENTS` は以下の形式で受け取る:

- `daily` または `日次` — 直近24時間のダイジェスト
- `weekly` または `週次` — 直近7日間のダイジェスト
- `[daily|weekly] [LINE|DISCORD|SLACK|CHATWORK]` — プラットフォーム指定
- 引数なしの場合 — `daily` をデフォルトとする

## 実行手順

### Step 1: 対象チャネルの取得

1. `list-channels` でチャネル一覧を取得
2. プラットフォーム指定がある場合は `type` でフィルタ
3. アクティブなチャネル（期間内にメッセージがあるもの）を特定

### Step 2: 各チャネルのメッセージ取得

各チャネルに対して `list-messages` を実行:
- daily: `from` を24時間前に設定
- weekly: `from` を7日前に設定
- メッセージがないチャネルはスキップ

### Step 3: ダイジェストの生成

チャネル横断で以下を整理する:

#### A. 全体サマリー
- 期間中のアクティブチャネル数、総メッセージ数
- 最も活発だったチャネルTOP3

#### B. チャネル別ハイライト（アクティブなもののみ）
- チャネル名
- メッセージ数
- 主な話題（1〜2行で要約）
- 重要な決定事項があれば記載

#### C. 要対応事項（全チャネル横断）
- 未返信の質問
- 期限が近いタスク
- エスカレーションが必要な事項
- 優先度: 高/中/低 でラベリング

#### D. トレンド（weeklyのみ）
- 前週との比較（活動量の増減）
- 新規チャネル・離脱ユーザー
- 注目すべき変化

### Step 4: 送信先の特定

1. `list-connect-users` でLINEユーザー一覧を取得（type: "LINE"）
2. 送信先をユーザーに確認する（**必ず確認してから送信すること**）

### Step 5: LINEへ配信

`send-line-message` でFlexメッセージとして送信する。

#### メッセージフォーマット

**メッセージ1: 全体サマリー**
```json
{
  "type": "flex",
  "altText": "[日次|週次]ダイジェスト [日付]",
  "contents": {
    "type": "bubble",
    "size": "giga",
    "header": {
      "type": "box", "layout": "vertical",
      "contents": [
        { "type": "text", "text": "[日次|週次]ダイジェスト", "weight": "bold", "size": "lg" },
        { "type": "text", "text": "[期間]", "size": "xs", "color": "#999999" }
      ]
    },
    "body": {
      "type": "box", "layout": "vertical", "spacing": "md",
      "contents": [
        { "type": "text", "text": "アクティブ: Xチャネル / メッセージ: Y件", "size": "sm", "color": "#666666" },
        { "type": "separator", "margin": "md" },
        { "type": "text", "text": "要対応", "weight": "bold", "size": "sm", "color": "#e74c3c" },
        { "type": "text", "text": "・[高] ...", "size": "sm", "wrap": true },
        { "type": "text", "text": "・[中] ...", "size": "sm", "wrap": true }
      ]
    }
  }
}
```

**メッセージ2: チャネル別ハイライト（テキストメッセージ）**
- チャネルが多い場合はテキストで簡潔にまとめる
- 各チャネル2行以内（チャネル名 + 要約）

## LLM出力ポリシー（厳守）

- **要対応事項は具体的に**: 「確認が必要」→「田中さんから4/3に届いた見積もりの件、返信待ち」
- **優先度の根拠を明示**: なぜ「高」なのか（期限切れ、金額大、クレームリスク等）
- **LINE表示前提**: 全体サマリーは1画面に収まるよう情報を厳選
- **ゼロアクティビティの報告は不要**: メッセージがないチャネルは無視

## 注意事項

- 送信前に必ずユーザーに内容と送信先を確認すること
- チャネル数が多い場合、全チャネルの詳細は不要。要対応事項を優先
- 最大5メッセージに収めること

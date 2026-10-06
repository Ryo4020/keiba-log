# 実装タスク: REQ-TICKET-006 直近記録リスト表示

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-TICKET-006` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-TICKET-006-recent-records.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`/records/recent` の `get`）, `specs/keiba-log/designs/architecture.md`（3.3・5節「Lambda の権限」）, `specs/keiba-log/designs/database.md`（2節 AP-02）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-02）
- **概要**:
  - API Gateway（HTTP API）の `GET /records/recent` ルート（JWT オーソライザー付き、API 本体の Lambda 統合に向ける）
  - API 本体の Lambda 実行ロールの権限: `keiba-log` テーブルに対する AP-02（`Query`）の操作
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-006-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-006-recent-records.md`, `specs/keiba-log/designs/api.yml`（`/records/recent` の `get`、`RecordSummaryList`・`RecordSummary`・`RecordType`・`ConfirmationStatus`・`Venue`、`RecordKey`、`BadRequest` 応答）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/architecture.md`（5節「データ分離」）, `specs/keiba-log/designs/database.md`（1.5・1.6・2節 AP-02・4節「読み取り整合性」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・共通属性・「関連するアクセスパターン」の AP-02・「AP-02 の射影属性」）
- **概要**:
  - `GET /records/recent`（`listRecentRecords`）のエンドポイント
    - 対象ユーザーは JWT オーソライザーが検証したクレームの `sub` のみで特定する
    - クエリパラメータ `limit`（1〜20の整数、省略時は 5）の受け取り。範囲外・整数でない場合は丸めずに `400`（`VALIDATION_ERROR`、不正な項目を `details` に列挙）を返す
    - AP-02 による取得（日付の新しい順・同一日付は登録日時の新しい順で、先頭から `limit` 件。ページングなし。結果整合の読み取り）
      - `ProjectionExpression` は `db-tables/keiba-log.md`「AP-02 の射影属性」の 12 属性とし、`bets` / `betGroups` は取得しない
    - 取得したアイテムから `RecordSummaryList` 形式のレスポンスを組み立てる
      - `recordKey` の発行、`balanceTargetPurchaseAmount`・`balanceTargetPayoutAmount` への集計寄与額の対応付け
      - 簡易記録では `raceNumber`・`confirmationStatus` の項目自体を含めない
      - 0 件のときは `items` を空配列で返す
    - 応答: 成功は `200`、`limit` 不正は `400`、想定外エラーは `500`（`api.md` 3節の共通エラーレスポンス）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-006-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-006-recent-records.md`, `specs/keiba-log/designs/ui-designs/home.md`（1節・2節の直近記録リストと馬券記録リスト行・確定状態による表記の分岐、3.1「遷移・開閉」、3.3「データの状態による表示の分岐」、3.4「読み込み中とエラー」、3.5「共通の操作性」、4節の共通パーツ、5節の Figma ノード `17-3`・`18-4`・`19-2`・`21-2`・`23-5`〜`23-9`）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・2節・5節・6節 共通パーツ一覧）, `specs/keiba-log/designs/api.yml`（`/records/recent` の `get`、`RecordSummaryList`・`RecordSummary`）, `specs/keiba-log/designs/architecture.md`（3.5 編集画面のクエリパラメータ方式）
- **概要**:
  - ホーム画面（`/`）の直近記録リスト領域
    - セクション見出し「直近の記録」と、馬券記録一覧画面（`/records`）への導線「すべて見る」
    - `GET /records/recent?limit=5` を呼び、`items` を返ってきた順のまま最大 5 件表示する（クライアント側で並べ替えない。ページング・もっと見るは置かない）
    - 0 件のときは共通パーツ「空状態表示」を配置する（見出しと「すべて見る」は残す）
    - 読み込み中・`500`（再読み込みボタンで当該 API のみ再取得）・`401` の扱いを収支サマリー領域とは独立して行う
  - 共通パーツ「馬券記録リスト行」（オーナー: ホーム画面）
    - 日付・競馬場とレース番号（簡易記録は競馬場のみ）・記録種別バッジ・購入額・払戻額・収支の表示。払戻額・収支は共通パーツ「収支表示」を配置する
    - 確定状態による表記の分岐（確定前のレース記録は払戻額・収支を「確定前」と表記し、「0円」と区別する）
    - 行全体の押下で馬券記録編集画面（`/records/edit?key=<recordKey>`、`recordKey` は URL エンコード）へ遷移する。ホバーで選択できる行であることを示す
    - 行内に削除の操作（スワイプ削除・長押しメニューを含む）を置かない
  - 共通パーツ「記録種別バッジ」（オーナー: ホーム画面）: `RACE` は「レース記録」、`SIMPLE` は「簡易記録」を示す表示専用のラベル
  - 共通パーツ「新規登録の導線」（オーナー: ホーム画面）: 馬券記録登録画面（`/records/new`）へ遷移するプライマリボタン。ホーム画面の枠に配置する
  - ホーム画面の枠・グローバルサイドバー・収支表示・空状態表示を配置する
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-006-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-006-recent-records.md`, `specs/keiba-log/designs/api.yml`（`/records/recent` の `get`、`RecordSummaryList`・`RecordSummary`、`BadRequest` 応答）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（1.5・2節 AP-02）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-02・「AP-02 の射影属性」）
- **概要**:
  - `GET /records/recent` のバックエンド処理の Go ユニットテスト
    - 取得対象がクレームの `sub` のユーザーの馬券記録に限られること
    - `limit` の指定・省略時（5 件）の取得件数、記録が `limit` 未満・0 件のときのレスポンス（0 件は空配列）
    - `limit` の境界値の扱い
      - `400`（`VALIDATION_ERROR`、丸めずに拒否）になるもの: 0、21、負数（例: -1）、数値でない文字列（例: `abc`）、小数（例: `1.5`）
      - `400` にならないもの: 1、20（範囲内としてそのまま処理）、省略時（5 件）
      - `400` のとき DynamoDB への問い合わせを行わないこと
    - 日付の新しい順・同一日付は登録日時の新しい順で返ること
    - 射影が「AP-02 の射影属性」どおりで、`bets` / `betGroups` を取得しないこと
    - レース記録・簡易記録それぞれのレスポンスの項目（簡易記録で `raceNumber`・`confirmationStatus` を含めないこと、`recordKey`・収支計上対象額の対応付け）
    - 成功時の `200`、`limit` 不正時の `400`、DynamoDB の失敗時の `500` の応答
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-006-*.md` の受け入れ基準をすべて満たしていること

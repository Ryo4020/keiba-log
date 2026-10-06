# 実装タスク: REQ-TICKET-002 簡易記録の登録

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-TICKET-002` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-TICKET-002-simple-record-create.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`/records` の `post`）, `specs/keiba-log/designs/architecture.md`（3.3・5節「Lambda の権限」）, `specs/keiba-log/designs/database.md`（2節 AP-05）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-05）
- **概要**:
  - API Gateway（HTTP API）の `POST /records` ルート（JWT オーソライザー付き、API 本体の Lambda 統合に向ける）
  - API 本体の Lambda 実行ロールの権限
    - `keiba-log` テーブルに対する AP-05 の操作（`TransactWriteItems` に含める馬券記録アイテムの `Put` と、生涯収支集計アイテムの更新）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-002-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-002-simple-record-create.md`, `specs/keiba-log/designs/api.yml`（`/records` の `post`、`RecordInput`・`SimpleRecordInput`・`Venue`、`Record`・`RecordCommon`・`SimpleRecord`、`components.responses`）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/architecture.md`（5節「データ分離」）, `specs/keiba-log/designs/database.md`（1.4・1.6・2節 AP-05）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・ソートキーの構成要素・生涯収支集計アイテムの属性・馬券記録アイテムの共通属性・簡易記録固有の属性・「関連するアクセスパターン」の AP-05）, `specs/keiba-log/requirements/business-rules.md`（マスタデータ「対象とする競馬場（JRA・10場）」「（NAR・14場）」・エンティティ項目定義「簡易記録」・「購入金額のルール」の簡易記録の行・「収支額・回収率の算出ルール」）
- **概要**:
  - `POST /records`（`createRecord`）のエンドポイントのうち、`recordType` が `SIMPLE`（`SimpleRecordInput`）の登録
    - 対象ユーザーは JWT オーソライザーが検証したクレームの `sub` のみで特定する（ボディにユーザー識別子を含めない）
    - `recordType` による記録種別の判別（discriminator）
    - 簡易記録の入力値の検証（`business-rules.md` のエンティティ項目定義「簡易記録」と「購入金額のルール」に従う）
      - 日付: `YYYY-MM-DD` 形式、`1900-01-01` 以上かつ JST（日本標準時、UTC+9）の当日以下（未来日は不可）
        - 上限の「当日」は、サーバーの現在時刻（Lambda の実行環境は通常 UTC）から JST の暦日を求めて比較する
        - 入力の日付（`YYYY-MM-DD`）は暦日そのものとして比較し、タイムゾーン変換はしない。登録日時（`createdAt`）は UTC のまま
      - 競馬場: マスタデータ（JRA・NAR）の値のみ
      - 購入額: 0より大きく `99,999,999` 以下、100円の整数倍
      - 払戻額: 0以上 `99,999,999` 以下
      - 簡易記録が持たない項目（買い目・買い目の集合・レース番号・結果の確定状態・結果状態）は保存しない
      - 不正な項目は `400`（`VALIDATION_ERROR`）として `details` に項目ごとの `field`・`message` を列挙する
    - AP-05: `createdAt`（現在時刻、ミリ秒精度）と `recordId`（UUID）を採番した `SK` で馬券記録アイテムを `Put` し、生涯収支集計アイテムへの寄与額（購入額・払戻額）と件数の加算を1つのトランザクション書き込みで行う
      - `recordType` は `SIMPLE` で保存する
      - 集計対象額（`summaryContributionPurchase`・`summaryContributionPayout`）は入力の購入額・払戻額をそのまま用いる
    - 登録した記録から `SimpleRecord` 形式のレスポンスを組み立てる（`recordKey` の発行、`balanceTargetPurchaseAmount`・`balanceTargetPayoutAmount` への集計対象額の対応付け）
    - 応答: 成功は `201`、入力値不正は `400`、書き込みの競合は `409`、想定外エラーは `500`（`api.md` 3節の共通エラーレスポンス）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-002-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-002-simple-record-create.md`, `specs/keiba-log/designs/ui-designs/record-create.md`（2節「タブの仕様」、4節 簡易記録タブ、5.1「登録の実行」、5.2「入力値の検証と不正時の表示」、5.3「共通の操作性」、6.1 記録共通入力ブロック、7節 状態 #4、9節 Figma の簡易記録タブ `151-465`・`158-934`）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・4節・5節・6節 共通パーツ一覧）, `specs/keiba-log/designs/api.yml`（`/records` の `post`、`SimpleRecordInput`・`Venue`、`components.responses`）, `specs/keiba-log/designs/api.md`（3節）, `specs/keiba-log/requirements/business-rules.md`（マスタデータの競馬場・エンティティ項目定義「簡易記録」・「購入金額のルール」）
- **概要**:
  - 馬券記録登録画面（`/records/new`）の簡易記録タブのフォーム
    - 共通パーツ「記録共通入力ブロック」（日付・競馬場）を配置する
      - 日付の補助テキストを簡易記録用に差し替え、未来日を入力できないこととレース記録タブとは範囲が異なることを示す
        - 補助テキストと入力欄の下の注記は、上限が JST（日本標準時）の当日であると読み取れる表記にする（`record-create.md` 4.1）
    - 購入額・払戻額の数値入力（いずれも必須、払戻額は0を許容）
    - 買い目・結果の確定状態・レース番号の入力欄を置かない
    - フォームの見出しと登録ボタンのラベルで簡易記録を登録していることを示す
    - 入力値はレース記録タブと独立して保持する
  - 登録操作と応答ごとの動作
    - 登録ボタン押下でクライアント側の検証（`business-rules.md` の簡易記録の項目定義）を行う。日付が未来日かどうかは、端末のタイムゾーンによらず JST の当日を基準に判定する（日付の値は暦日そのものとして比較し、タイムゾーン変換はしない）。不正がなければ送信中状態にして `POST /records` を `SimpleRecordInput`（`recordType: SIMPLE`）で1回だけ呼ぶ（送信中は再押下させない）
    - `201` を受け取ったら馬券記録一覧画面（`/records`）へ遷移する
    - `400` を受け取ったら `details` の項目ごとに不正な項目を示す
    - `401` を受け取ったら認証状態を終了し、ログイン画面（`/login`）へ遷移する
    - `409` を受け取ったら競合したことを示し、やり直せる状態にする
    - `500` を受け取ったら登録されていないことを示し、ボタンを押下可能に戻してやり直せる状態にする
    - キャンセル押下で入力を破棄して前の画面へ戻る
  - 入力値不正時の表示
    - 不正な項目の入力枠をエラー表示にし、直下に項目固有の注釈を出す（フォーム全体のエラーバナーは出さない）
    - 再送信時は直前のエラー表示をクリアし、新しい検証結果で出し直す
  - タブ切り替え時に簡易記録タブのフォームの先頭へフォーカスを移す。押下できる要素は最小タップ領域を確保する
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-002-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-002-simple-record-create.md`, `specs/keiba-log/designs/api.yml`（`/records` の `post`、`SimpleRecordInput`・`SimpleRecord`）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（1.4・2節 AP-05）, `specs/keiba-log/designs/db-tables/keiba-log.md`（馬券記録アイテムの共通属性・簡易記録固有の属性・「関連するアクセスパターン」の AP-05）, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義「簡易記録」・「購入金額のルール」）
- **概要**:
  - `POST /records`（簡易記録）のバックエンド処理の Go ユニットテスト
    - 記録アイテムの `Put` と生涯収支集計アイテムへの加算（購入額・払戻額・件数）が1つのトランザクション書き込みで行われること（払戻額0の記録を含む）
    - 保存するアイテムのキーがクレームの `sub` から組み立てられ、`recordType` が `SIMPLE`、集計対象額が入力の購入額・払戻額と一致すること
    - 保存するアイテムが買い目・買い目の集合・レース番号・結果の確定状態を持たないこと
    - 入力値の検証の境界値（日付の下限・当日・未来日・形式不正、マスタ外の競馬場、購入額の0・100円単位でない値・上限超過、払戻額の負値・上限超過、必須項目の欠落）で `400` と `details` の項目が返ること
      - 日付の当日・未来日は JST の当日を基準にする。現在時刻を固定して、UTC と JST で日付がずれる時間帯（UTC 15:00〜24:00 = JST 翌日 0:00〜9:00）の境界（例: UTC 14:59:59 と 15:00:00 の前後、JST の当日は受理・翌日は `400`）を含める
    - 成功時の `201` とレスポンスの項目（`recordKey`・収支計上対象額の対応付け）、書き込みの競合時の `409`、DynamoDB の想定外の失敗時の `500`
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-002-*.md` の受け入れ基準をすべて満たしていること

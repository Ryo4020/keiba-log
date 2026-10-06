# 実装タスク: REQ-TICKET-004 馬券記録の削除

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-TICKET-004` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-TICKET-004-record-delete.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`/records/{recordKey}` の `delete`）, `specs/keiba-log/designs/architecture.md`（3.3・5節「Lambda の権限」）, `specs/keiba-log/designs/database.md`（2節 AP-07）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-07）
- **概要**:
  - API Gateway（HTTP API）の `DELETE /records/{recordKey}` ルート（JWT オーソライザー付き、API 本体の Lambda 統合に向ける）
  - API 本体の Lambda 実行ロールの権限
    - `keiba-log` テーブルに対する AP-07 の操作（`TransactWriteItems` に含める馬券記録アイテムの条件付き削除と、生涯収支集計アイテムの更新）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-004-record-delete.md`, `specs/keiba-log/designs/api.yml`（`/records/{recordKey}` の `delete`・`components.parameters.RecordKey`・`components.responses`）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/architecture.md`（5節「データ分離」）, `specs/keiba-log/designs/database.md`（1.3・1.4・1.6・2節 AP-07・4節「TTL」「バックアップ・復旧」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・「馬券記録1件への到達」・「関連するアクセスパターン」の AP-07）, `specs/keiba-log/requirements/business-rules.md`（収支額・回収率の算出ルール）
- **概要**:
  - `DELETE /records/{recordKey}`（`deleteRecord`）のエンドポイント
    - 対象ユーザーは JWT オーソライザーが検証したクレームの `sub` のみで特定し、`recordKey` から対象の馬券記録アイテムのキーを組み立てる
    - `recordKey` の形式が不正な場合は `400`
    - AP-07: 馬券記録アイテムの削除（存在を条件とする条件付き削除）と、生涯収支集計アイテムからの寄与額（購入額・払戻額）の減算・件数の減算を1つのトランザクション書き込みで行う。減算する寄与額は削除対象の記録アイテムが保持する集計対象額とする
    - レース記録にネストした買い目・買い目の集合は記録アイテムの削除で同時に削除する
    - 論理削除・復元の仕組みは持たず、物理削除とする
    - 応答: 成功は `204`（ボディなし）、存在しない・他ユーザーの記録は区別せず `404`、書き込みの競合は `409`、想定外エラーは `500`（`api.md` 3節の共通エラーレスポンス）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-004-record-delete.md`, `specs/keiba-log/designs/ui-designs/home.md`（2節「確認ダイアログ」・3.5「共通の操作性」・4節 共通パーツ「確認ダイアログ」・5節 Figma の確認ダイアログのコンポーネント `23-11`）, `specs/keiba-log/designs/ui-designs/record-list.md`（2節「削除ボタン」「確認ダイアログ」・3.3「削除の確認フロー」・3.4「共通の操作性」・4.2・5節 削除確認ダイアログのフレーム `99-101`・`102-223`）, `specs/keiba-log/designs/ui-designs/record-edit.md`（2.1「記録の削除」・2.4・2.5「確認ダイアログ」の記録の削除・3.5「記録の削除」・3.6・5節「記録の削除」の要素と確認ダイアログ `208:1253`）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・2節・4節・6節 共通パーツ一覧）, `specs/keiba-log/designs/api.yml`（`/records/{recordKey}` の `delete`）, `specs/keiba-log/designs/api.md`（3節）
- **概要**:
  - 共通パーツ「確認ダイアログ」（オーナー: ホーム画面）
    - 見出し・本文・注意ブロック・取りやめ／破壊的操作のボタン行の構成。オーバーレイを敷いて表示する
    - 本文・注意ブロック・ボタンのラベルを呼び出し側で差し替えられる
    - 取りやめ手段（取りやめボタン・オーバーレイ押下・Esc）
    - 実行中は両ボタンを押下不可にし、破壊的操作のボタンのラベルを実行中の表示に変える
    - 初期フォーカスを取りやめボタンに置く、開いている間は背後の要素にフォーカスを移さない、閉じたらフォーカスを呼び出し側が指定する要素に戻す
  - 馬券記録一覧画面（`/records`）の各行からの削除
    - 行ごとの削除ボタン（行ヘッダーの右端。行の押下による編集画面への遷移・買い目リストトグルと独立させ、押下が行の選択を発火させない。最小タップ領域を確保する）。押下で確認ダイアログを開き、この時点では何も削除しない
    - ダイアログの本文を記録種別（レース記録／簡易記録）で切り替え、レース記録では買い目・買い目の集合も同時に削除されることを示す。削除を取り消せないことを本文と注意ブロックで示す
    - 取りやめたときは何も削除しない
    - 「削除する」押下で `DELETE /records/{recordKey}` を1回だけ呼ぶ（実行中は「削除中…」）
    - `204` を受け取ったらダイアログを閉じ、再取得せずに一覧から当該行を除去する
    - `401` を受け取ったら認証状態を終了し、ログイン画面（`/login`）へ遷移する
    - `404` を受け取ったらダイアログを閉じ、一覧から当該行を除去する
    - `500` を受け取ったらダイアログを閉じずにエラーメッセージを表示し、ボタンを押下可能に戻してやり直せる状態にする
    - ダイアログを閉じたときのフォーカスの戻し先（取りやめ時は削除ボタン、削除完了時は次に存在する行または一覧の先頭）
  - 馬券記録編集画面（`/records/edit?key=<recordKey>`）の「記録の削除」領域からの削除
    - 画面下部の「記録の削除」領域（見出し・「この記録を削除」ボタン・取り消せない旨の注記。保存アクションバーとは別の領域・見た目で区別する。記録が見つからない状態では表示しない）
    - 「この記録を削除」押下で確認ダイアログを開き、記録と含まれるすべての買い目が削除されることと元に戻せないことを示す（ボタンは「キャンセル」「削除する」）
    - キャンセルしたときは何もせず編集画面に戻る
    - 承認で `DELETE /records/{recordKey}` を1回だけ呼び、送信中は再押下させない
    - `204` を受け取ったら馬券記録一覧画面（`/records`）へ遷移する
    - `401` を受け取ったら認証状態を終了し、ログイン画面（`/login`）へ遷移する
    - `404` を受け取ったら記録が見つからない状態を表示する
    - `409` を受け取ったら競合したことを示し、やり直しを促す
    - `500` を受け取ったら削除されていないことを示し、やり直せるようにする
    - ダイアログを閉じたときは「この記録を削除」ボタンにフォーカスを戻す
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-TICKET-004-record-delete.md`, `specs/keiba-log/designs/api.yml`（`/records/{recordKey}` の `delete`）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（1.4・1.6・2節 AP-07）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-07）, `specs/keiba-log/requirements/business-rules.md`（収支額・回収率の算出ルール）
- **概要**:
  - `DELETE /records/{recordKey}` のバックエンド処理の Go ユニットテスト
    - 記録アイテムの削除と生涯収支集計アイテムの減算（寄与額・件数）が1つのトランザクション書き込みで行われること（レース記録・簡易記録、寄与額が0のレース記録を含む）
    - 対象のキーがクレームの `sub` と `recordKey` から組み立てられ、他ユーザーの記録に到達しないこと
    - 存在しない・他ユーザーの記録の指定で `404`、`recordKey` の形式が不正な場合の `400`
    - 成功時の `204`、書き込みの競合時の `409`、DynamoDB の想定外の失敗時の `500`
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-004-*.md` の受け入れ基準をすべて満たしていること

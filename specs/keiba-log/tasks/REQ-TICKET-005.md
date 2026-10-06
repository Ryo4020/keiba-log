# 実装タスク: REQ-TICKET-005 馬券記録一覧

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-TICKET-005` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-TICKET-005-record-list.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`GET /records`・`RecordList`・`Record`・`RaceRecord`・`SimpleRecord`・`Bet`・`BetGroup`・`MethodSpec`・`components.responses`）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（1.5・1.6・2節 AP-03・4節「読み取り整合性」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・属性一覧・関連するアクセスパターン AP-03）, `specs/keiba-log/requirements/user-stories/REQ-TICKET-005-record-list.md`
- **概要**:
  - `GET /records`（馬券記録一覧の取得）のエンドポイント
    - ログイン中のユーザー（JWT の `sub`）のパーティションの馬券記録アイテムのみを対象にする
    - AP-03 の Query（`SK begins_with "RECORD#"`、`ScanIndexForward = false`、`Limit = 20`、結果整合の読み取り、`ProjectionExpression` なし）で日付の新しい順（同一日付は登録日時の新しい順）に取得する
    - 1ページ最大20件固定。取得件数・並び替え・絞り込みのパラメータは受け付けない。Query の 1MB 上限で20件未満になる場合は継続読み取りせずそのまま返す
  - カーソルによるページング
    - `LastEvaluatedKey` を不透明な文字列の `nextCursor` に変換して返し、次ページがない場合は `null` を返す
    - クエリパラメータ `cursor` を `ExclusiveStartKey` に復元する。パーティションキーはカーソルの値ではなく JWT の `sub` から組み立てる
    - 復元できない・形式が不正なカーソルは `400`（`VALIDATION_ERROR`）を返す
  - 馬券記録アイテムから `Record` スキーマへの変換
    - 共通項目（`recordKey`・`recordId`・`recordType`・`date`・`venue`・`createdAt`・`purchaseAmount`・`balanceTargetPurchaseAmount`・`balanceTargetPayoutAmount`）
    - レース記録は `raceNumber`・`confirmationStatus`・`bets`・`betGroups`（買い目の集合の `methodSpec`・`unitAmount`・`betCount`・`amount`・配下の `bets` を含む）を返す
    - 簡易記録は `payoutAmount` を返し、買い目・買い目の集合・レース番号・結果の確定状態を返さない
  - 記録が0件の場合は `items` を空配列、`nextCursor` を `null` で返す
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-005-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/designs/ui-designs/record-list.md`, `specs/keiba-log/designs/ui-designs/home.md`（共通パーツ「記録種別バッジ」「収支表示」「馬券記録リスト行」「新規登録の導線」「空状態表示」の定義）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・4節・5節・6節）, `specs/keiba-log/designs/api.yml`（`GET /records`・`RecordList`・`Record`・`Bet`・`BetGroup`・`MethodSpec`）, `specs/keiba-log/requirements/business-rules.md`（「組合せの表記ルール」「買い方の指定内容の表記ルール」「買い方名の表記」・対象とする券種）, `specs/keiba-log/requirements/user-stories/REQ-TICKET-005-record-list.md`
- **概要**:
  - 馬券記録一覧画面（`/records`）
    - グローバルサイドバー（「馬券記録一覧」を現在位置としてハイライト）、画面タイトル行、新規登録の導線、一覧カードを配置する
    - 画面表示時に `GET /records`（`cursor` 省略）を呼び、`items` を返ってきた順のまま表示する（クライアント側で並べ替えない）。絞り込み・検索・並び替えの操作は置かない
    - 「次のページを読み込む」操作: `nextCursor` があるときだけ表示し、押下で `nextCursor` を `cursor` に指定して取得した記録を末尾に追加表示する。ページ番号ジャンプは置かない
    - 読み込み中（初回は一覧カード全体、追加読み込みはページング操作の位置）、`500` のエラー表示と再読み込み（追加読み込みの失敗時は既存の一覧を保持してページング操作の位置に表示）、`401` のときのログイン画面への遷移
    - 1ページ目の `items` が0件のときは一覧カード領域を空状態表示（文言は本画面向け）に差し替える
  - 一覧の各行（馬券記録リスト行の拡張）
    - 行本体として記録種別バッジ・収支表示・馬券記録リスト行を配置する
    - md以上の行ヘッダー（見出しグループ・ラベル付きの金額グループ）の構成
    - 行の押下で馬券記録編集画面（`/records/edit?key=<recordKey>`。`recordKey` は URL エンコードする）へ遷移する
    - 行ヘッダーの右端に削除ボタンを置く領域を、行の押下領域・買い目リストトグルと重ならないように確保する（削除ボタンの動作と削除の確認フローは本タスクの対象外とする）
  - 共通パーツ「買い目リスト」（本画面がオーナー）
    - レース記録の行にのみ表示し、簡易記録の行には買い目リスト・トグル・結果の確定状況を表示しない
    - トグルで開閉し、初期状態は畳んだ状態で「買い目 N点」（通常の買い目と全集合の買い目の合計件数）を示す。トグルの押下で行の選択を発火させない
    - 展開時は通常の買い目を1点1行（券種名・組・一点当たりの購入額・状態・払戻額）、買い目の集合は見出し行（券種名・買い方名・買い方の指定内容・点数・1点あたりの購入額。合計額は出さない）と配下の買い目（券種名を省略した4項目）で表示する
    - 状態は `resultStatus` に対応させる（`UNCONFIRMED` 確定前、`WON` 的中、`REFUNDED` 返還、`LOST` 空欄）
    - 組・買い方の指定内容・買い方名の表記は `business-rules.md` の表記ルールに従う
    - 追加読み込み・再読み込み後も展開済みの行の開閉状態を維持し、追加された行は畳んだ状態で表示する
    - base・md以上の両方のレイアウト
  - キーボード操作の順序（タイトル行 → 新規登録 → 各行（行本体 → 買い目リストトグル → 削除ボタン）→ ページング操作）、展開した買い目リストの内部をフォーカス対象にしないこと、各操作の最小タップ領域（`tap-target-min`）
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-005-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` ユニットテスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`GET /records`・`RecordList`・`Record`）, `specs/keiba-log/designs/api.md`（3節）, `specs/keiba-log/designs/database.md`（1.5・2節 AP-03）, `specs/keiba-log/designs/db-tables/keiba-log.md`（関連するアクセスパターン AP-03）, `specs/keiba-log/requirements/user-stories/REQ-TICKET-005-record-list.md`
- **概要**:
  - `backend/` の `GET /records` のユニットテスト
    - JWT の `sub` のパーティションのみを Query し、AP-03 の条件（降順・`Limit = 20`・`ProjectionExpression` なし・結果整合の読み取り）で呼び出すこと
    - `LastEvaluatedKey` がある場合に `nextCursor` が返り、その値を `cursor` に指定すると続きの位置から取得されること。ない場合に `nextCursor` が `null` になること
    - 他ユーザーのキーを含むよう改変したカーソルを指定しても、JWT の `sub` 以外のパーティションを読まないこと
    - 不正な形式のカーソルで `400`（`VALIDATION_ERROR`）になること
    - レース記録・簡易記録それぞれが `Record` スキーマどおりに変換されること（簡易記録に買い目・買い目の集合・レース番号・結果の確定状態が含まれないこと）
    - 記録が0件のとき `items` が空配列・`nextCursor` が `null` になること
    - DynamoDB の想定外エラーで `500` になること
- **受け入れ**: 対応する `requirements/user-stories/REQ-TICKET-005-*.md` の受け入れ基準をすべて満たしていること

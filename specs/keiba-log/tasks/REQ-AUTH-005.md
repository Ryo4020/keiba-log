# 実装タスク: REQ-AUTH-005 アカウント削除

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-005` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-005-account-deletion.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/api.yml`（`/me` の `delete`）, `specs/keiba-log/designs/architecture.md`（3.3・5節「Lambda の権限」・5.2「アカウント削除」）, `specs/keiba-log/designs/database.md`（2節 AP-08・AP-09、4節「アカウント削除の手順」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-08・AP-09）
- **概要**:
  - API Gateway（HTTP API）の `DELETE /me` ルート（JWT オーソライザー付き、API 本体の Lambda 統合に向ける）
  - API 本体の Lambda 実行ロールの権限
    - `keiba-log` テーブルに対する AP-08（`Query`）・AP-09（`BatchWriteItem` による削除）の操作
    - `cognito-idp:AdminDeleteUser`（対象は本アプリの User Pool の ARN のみ）
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-005-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-005-account-deletion.md`, `specs/keiba-log/designs/api.yml`（`/me` の `delete`）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/architecture.md`（3.4「ログ」・5節「データ分離」・5.2「アカウント削除」）, `specs/keiba-log/designs/database.md`（1.2・1.6・2節 AP-08・AP-09、4節「アカウント削除の手順」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（概要・PK/SK・「関連するアクセスパターン」の AP-08・AP-09）
- **概要**:
  - `DELETE /me`（`deleteAccount`）のエンドポイント
    - 対象ユーザーは JWT オーソライザーが検証したクレームの `sub` のみで特定する
    - 対象ユーザーのアイテム（生涯収支集計アイテム・すべての馬券記録アイテム）の列挙（AP-08）と一括削除（AP-09）
    - DynamoDB 側の削除がすべて完了した後に、`AdminDeleteUser` で Cognito のユーザーを削除する
    - 途中で失敗した場合も、同じ呼び出しの再実行で残りを削除して完了できる
    - 応答: 成功は `204`（ボディなし）、想定外エラーは `500`（`api.md` 3節の共通エラーレスポンス）
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-005-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-005-account-deletion.md`, `specs/keiba-log/designs/ui-designs/home.md`（2節・3節のサイドバーとアカウントメニューの操作・3.2「アカウント削除の確認ダイアログ」・アクセシビリティ・共通パーツ・Figma のノード）, `specs/keiba-log/designs/ui-overview.md`（1節「独立した画面を持たない要件の扱い」・2節・5節・6節 共通パーツ一覧）, `specs/keiba-log/designs/api.yml`（`/me` の `delete`）, `specs/keiba-log/designs/architecture.md`（5.2「トークンの保持と更新」「アカウント削除」）
- **概要**:
  - 共通パーツ「グローバルサイドバー」のアカウントメニュー（オーナー: ホーム画面）
    - メニュー項目「アカウント削除」（メニューの最後に置き、日常操作の項目から視覚的に分離する）。押下で確認ダイアログを開き、この時点では何も削除しない
  - 共通パーツ「確認ダイアログ」（オーナー: ホーム画面）を使ったアカウント削除の確認
    - 本文・注意ブロックで、馬券記録もすべて削除されることと削除を取り消せないことを示す
    - 取りやめ手段（キャンセルボタン・オーバーレイ押下・Esc）。取りやめたときは何も削除しない
    - 「削除する」押下で `DELETE /me` を1回だけ呼ぶ。実行中は両ボタンを押下不可にし、ラベルを「削除中…」に変える
    - `204` を受け取ったら `localStorage` のトークンを削除し、ログイン画面（`/login`）へ遷移する
    - `401` を受け取ったら認証状態を終了し、ログイン画面へ遷移する
    - `500` を受け取ったらダイアログを閉じずにエラーメッセージを表示し、ボタンを押下可能に戻してやり直せる状態にする
    - 初期フォーカスを取りやめボタンに置く、開いている間は背後の要素にフォーカスを移さない、閉じたら開く操作を行った要素にフォーカスを戻す
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-005-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-005-account-deletion.md`, `specs/keiba-log/designs/api.yml`（`/me` の `delete`）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（2節 AP-08・AP-09、4節「アカウント削除の手順」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「関連するアクセスパターン」の AP-08・AP-09）
- **概要**:
  - `DELETE /me` のバックエンド処理の Go ユニットテスト
    - 対象ユーザーの生涯収支集計アイテム・すべての馬券記録アイテムが削除されること（列挙が複数ページ・削除が複数バッチに分かれる場合、未処理分が返る場合を含む）
    - 対象がクレームの `sub` のユーザーに限られること
    - DynamoDB 側の削除の完了後に Cognito のユーザー削除が呼ばれ、DynamoDB 側の削除が失敗したときは Cognito のユーザー削除が呼ばれないこと
    - 成功時の `204`、DynamoDB・Cognito の失敗時の `500` の応答
    - 途中で失敗した後の再実行で削除を完了できること
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-005-*.md` の受け入れ基準をすべて満たしていること

# 実装タスク: REQ-AUTH-003 ログアウト

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-003` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-003-logout.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-003-logout.md`, `specs/keiba-log/designs/architecture.md`（5.2「ログアウト」「トークンの保持と更新」）, `specs/keiba-log/designs/ui-overview.md`（1節「独立した画面を持たない要件の扱い」・認証区分・画面の URL、5節のサイドバーの切り替え方針、6節「グローバルサイドバー」）, `specs/keiba-log/designs/ui-designs/home.md`（2節、3.1「遷移・開閉」、3.5「共通の操作性」、4節の共通パーツ「グローバルサイドバー（アカウントメニューを含む）」、Figma ノード `19-104`・`23-2`・`23-3`・`23-4`）, `specs/keiba-log/designs/ui-designs/login.md`（ログアウト完了後の表示）
- **概要**:
  - 認証必須の全画面に表示するグローバルサイドバーのアカウントのアイコンボタンと、そこから開くアカウントメニュー（`md` 未満はドロワー内、`md` 以上は常時表示のサイドバー内）
  - アカウントメニューの「ログアウト」項目（確認は求めない）
  - ログアウト処理
    - Cognito の `RevokeToken` で、この端末の Refresh Token を失効させる
    - `localStorage` に保持している ID Token・Access Token・Refresh Token を削除する。`RevokeToken` が通信エラーで失敗しても削除する
    - ログイン画面（`/login`）へ遷移する
  - アカウントメニューの「パスワード変更」「アカウント削除」の動作、およびログイン画面そのものの実装は本タスクの対象外とする
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-003-*.md` の受け入れ基準をすべて満たしていること

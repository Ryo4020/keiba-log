# 実装タスク: REQ-AUTH-007 パスワード再設定の申請

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-007` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-007-password-reset-request.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-007-password-reset-request.md`, `specs/keiba-log/designs/ui-designs/password-reset-request.md`, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・2節「画面遷移の補足」・6節 共通パーツ一覧）, `specs/keiba-log/designs/architecture.md`（3.1・3.2・5.2「パスワード再設定（コード方式）」・5.4）, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義 > アカウント・Cognito の仕様に委ねる値、アカウント 項番2・4・9）, `specs/keiba-log/designs/ui-designs/login.md`（共通パーツ「未認証画面レイアウト」「グローバルフッター」、エラーバナーの見た目、「確認コードの入力・再送」導線の文言）, `specs/keiba-log/designs/ui-designs/password-reset.md`（メールアドレスを引き継いだときの遷移先の初期状態）
- **概要**:
  - パスワード再設定申請画面（`/password-reset`）
    - 共通パーツ「未認証画面レイアウト」「グローバルフッター」を用いた画面構成（見出し・本文スロットのフォーム・画面間の導線スロット）
    - 案内文、メールアドレス入力欄、申請ボタン（送信中のラベル変更・非活性化による二重送信の防止）、補足文（未確認アカウント。入力値・結果によらず固定の文言）
    - 画面間の導線スロットの2件（ログイン画面 `/login`、確認コード入力画面 `/verify-email`（メールアドレスは引き継がない））
  - 入力値のクライアント側チェック（`business-rules.md` の値の制限・未入力）と、不正な項目の入力枠のエラー表示・項目直下の注釈（申請は実行しない）
  - Cognito User Pool の `ForgotPassword` の呼び出しと、応答ごとの扱い（`password-reset-request.md` 3.2・3.3）
    - 成功、存在しないメールアドレス・未確認アカウントでの応答、申請回数の超過（`LimitExceededException` の想定）を、すべて同じ受け付けとして扱い、遷移先・表示・遷移までのタイミングを揃える。アカウントの状態によって返り分かれる応答はすべて受け付けとして扱う
    - 通信エラー・想定外の失敗は、入力値を残したまま、アカウントの状態・申請回数に触れない文言のエラーバナーを出す
    - 送信時のエラーバナー・項目エラーのクリアと出し直し
  - 受け付け後のパスワード再設定画面（`/password-reset/confirm`）への遷移（本画面では案内表示を出さない。送信したメールアドレスをアプリ内の状態で引き継ぎ、URL に含めない）
  - エラーバナー表示時のフォーカス移動・スクリーンリーダーへの通知、補足文とメールアドレス入力欄の関連付け、キーボード操作の順序、最小タップ領域の確保（導線スロットのリンクを含む）（`password-reset-request.md` 3.5）
  - パスワード再設定画面（`/password-reset/confirm`）そのものの実装は本タスクの対象外とする
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-007-*.md` の受け入れ基準をすべて満たしていること

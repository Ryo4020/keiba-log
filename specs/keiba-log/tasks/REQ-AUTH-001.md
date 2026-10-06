# 実装タスク: REQ-AUTH-001 アカウント登録

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-001` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-001-signup.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/designs/ui-designs/signup.md`, `specs/keiba-log/requirements/user-stories/REQ-AUTH-001-signup.md`, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義 > アカウント・Cognito の仕様に委ねる値、アカウント 項番3）, `specs/keiba-log/designs/architecture.md`（3.1・5.2「登録 → 確認コード入力 → ログイン」「登録済みのメールアドレスで登録したとき」）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・2節「画面遷移の補足」・6節 共通パーツ一覧）, `specs/keiba-log/designs/ui-designs/login.md`（共通パーツ「未認証画面レイアウト」「グローバルフッター」）
- **概要**:
  - アカウント登録画面（`/signup`）
    - 共通パーツ「未認証画面レイアウト」「グローバルフッター」を用いた画面構成（見出し・本文スロットのフォーム・「ログイン画面に戻る」導線）
    - メールアドレス入力欄、エラーバナー、アカウント登録ボタン（送信中の表示・二重送信の防止）
  - 共通パーツ「パスワード入力ブロック」（本画面がオーナー。パスワード入力欄・強度要件 5 項目の提示と達成状態の表示・パスワード（確認）入力欄。パスワード再設定画面・パスワード変更画面から流用できる形にする）
  - 入力値のクライアント側チェックと項目ごとのエラー注釈（必須、メールアドレスの形式・最大文字数、パスワードの強度要件・最大文字数、パスワード確認の不一致）
  - Cognito User Pool の `SignUp` の呼び出しと、応答ごとの表示の対応（`signup.md` 3.5）
    - 登録の成功と `UsernameExistsException` を同じ受け付けとして扱い、遷移先・表示・遷移までのタイミングを揃える
    - `InvalidPasswordException` はパスワード入力欄の項目エラー、回数の超過は回数上限のエラーバナー、それ以外は通信エラーのエラーバナー
  - 受け付け後の確認コード入力画面（`/verify-email`）への遷移
    - 入力したメールアドレスだけをアプリ内の状態で引き継ぎ（URL に含めない）、パスワード・パスワード（確認）は破棄する
    - 確認コードの再送は呼ばない。トークンを保持せず、認証状態にしない
  - フォーカス移動・スクリーンリーダーへの通知・キーボード操作順・最小タップ領域（`signup.md` 3.6）
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-001-*.md` の受け入れ基準をすべて満たしていること

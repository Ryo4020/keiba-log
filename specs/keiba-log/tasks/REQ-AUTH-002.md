# 実装タスク: REQ-AUTH-002 ログイン

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-002` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-002-login.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-002-login.md`, `specs/keiba-log/designs/ui-designs/login.md`, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」・2節「画面遷移の補足」・6節 共通パーツ一覧）, `specs/keiba-log/designs/architecture.md`（5.2「未確認アカウントでのログイン」「ログイン」「トークンの保持と更新」）, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義 > アカウント）, `specs/keiba-log/designs/ui-designs/email-verification.md`（メールアドレスを引き継いだときの初期状態）
- **概要**:
  - 共通パーツ「グローバルフッター」（全画面共通。オーナー: ログイン画面）
  - 共通パーツ「未認証画面レイアウト」（中央寄せの単一カード枠。サービス名・見出しスロット・本文スロット・画面間の導線スロット。オーナー: ログイン画面）
  - ログイン画面（`/login`）
    - ログインフォーム（メールアドレス入力欄・パスワード入力欄・ログインボタン）
    - 入力値のクライアント側チェック（`business-rules.md` の値の制限・未入力）と、不正な項目の入力枠のエラー表示・項目直下の注釈
    - 送信中状態（ボタンのラベル変更・非活性化による二重送信の防止）
    - Cognito User Pool への SRP 認証（`InitiateAuth` の `USER_SRP_AUTH`）の呼び出しと、成功時のトークンの保持・ホーム画面（`/`）への遷移
    - 認証情報不一致のエラーバナー（どちらの項目が誤りかを判別できない表示）
    - 未確認アカウント（`UserNotConfirmedException`）のエラーバナーと、バナー内の「確認コードを入力する」導線（判定に使ったメールアドレスを URL 以外の手段で引き継いで確認コード入力画面 `/verify-email` へ遷移する。確認コードの自動再送はしない）
    - 送信時のエラーバナーのクリアと出し直し
    - 画面間の導線スロットの3件（パスワード再設定申請画面 `/password-reset`・アカウント登録画面 `/signup`・確認コード入力画面 `/verify-email`（メールアドレスは引き継がない））
    - エラーバナー表示時のフォーカス移動、キーボード操作の順序、最小タップ領域の確保
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-002-*.md` の受け入れ基準をすべて満たしていること

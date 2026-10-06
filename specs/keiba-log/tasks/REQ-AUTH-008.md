# 実装タスク: REQ-AUTH-008 パスワードの変更

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-008` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-008-password-change.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-008-password-change.md`（#12）, `specs/keiba-log/designs/architecture.md`（1節・3節・3.1「Lambda トリガー」・3.4・4節・5節「Lambda の権限」・5.2「パスワード再設定（コード方式）」「パスワード再設定でトリガーが失敗したとき」・5.4）
- **概要**:
  - Cognito の Post confirmation トリガー用の Lambda 関数（API 本体とは別の関数）
    - `backend/` のトリガーのエントリーポイントのビルド済みバイナリ、`provided.al2023`、arm64、VPC に配置しない
    - タイムアウトは Cognito がトリガーの応答を待つ時間（5 秒の想定）以内
  - トリガー用の関数の実行ロール（関数専用。次のみを許可）
    - `cognito-idp:AdminUserGlobalSignOut`（対象は本アプリの User Pool の ARN のみ）
    - 自関数のロググループへの書き込み
    - DynamoDB への権限は持たせない
  - トリガー用の関数のリソースベースポリシー（呼び出し元を本アプリの User Pool に限定する）
  - CloudWatch Logs: トリガー用の関数のロググループを CDK で明示的に作り、保持期間を 30 日にする
  - User Pool への Post confirmation トリガーの設定（ほかのトリガーは設定しない）
    - User Pool へのトリガーの設定と、実行ロールの権限を User Pool の ARN に限定することが循環参照にならないようにする（`architecture.md` 5.4 の要確認項目。権限の付け方に工夫が要る可能性がある）
  - トリガー用の関数は、トリガーのエントリーポイントのビルド済みバイナリをコードとしてデプロイする（処理を持たない関数を User Pool のトリガーに設定すると `ConfirmSignUp` が失敗するため）
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-008-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-008-password-change.md`（#12）, `specs/keiba-log/designs/architecture.md`（1節・3節・3.4・4節・5.2「パスワード再設定（コード方式）」「パスワード再設定でトリガーが失敗したとき」・5.4 の `triggerSource` の値・トリガーの応答を待つ時間）
- **概要**:
  - `backend/` に Cognito の Post confirmation トリガーのエントリーポイント（API 本体とは別のバイナリとしてビルドする。gin は使わず、Post confirmation のイベントを直接受ける）
  - トリガーの処理
    - `triggerSource` が `PostConfirmation_ConfirmForgotPassword` のときだけ、イベントの `userName` を指定して `AdminUserGlobalSignOut` を呼ぶ
    - それ以外（`PostConfirmation_ConfirmSignUp` など、登録の確認による起動）は何もしない
    - どちらの場合も、受け取ったイベントをそのまま返す
    - `AdminUserGlobalSignOut` が失敗したらエラーを返す（失敗を握りつぶさない）
  - トリガーのログは `triggerSource` と処理結果（失敗時はエラー）だけを出し、`userName`（メールアドレス）は出さない
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-008-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-008-password-change.md`, `specs/keiba-log/designs/ui-designs/password-reset.md`, `specs/keiba-log/designs/ui-designs/password-change.md`, `specs/keiba-log/designs/ui-designs/signup.md`（3.2・3.3 のパスワード入力ブロックの仕様）, `specs/keiba-log/designs/ui-designs/home.md`（3.1「遷移・開閉」、4節の共通パーツ「グローバルサイドバー（アカウントメニューを含む）」）, `specs/keiba-log/designs/ui-overview.md`（1節の認証区分・画面の URL、2節の画面遷移図・画面遷移の補足、6節 共通パーツ一覧）, `specs/keiba-log/designs/architecture.md`（5.1、5.2「パスワード再設定（コード方式）」「パスワード再設定でトリガーが失敗したとき」「トークンの保持と更新」）, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義 > アカウント）
- **概要**:
  - パスワード再設定画面（`/password-reset/confirm`。未認証で利用）
    - 共通パーツ「未認証画面レイアウト」「コード入力欄」「パスワード入力ブロック」「グローバルフッター」を配置した画面
    - 表示時の分岐: パスワード再設定申請画面から引き継いだメールアドレスがあるときは入力フォーム（メールアドレスは表示のみ）、ないとき（再読み込み・URL を直接開いた・履歴から戻った）は「引き継ぎなし」
    - 引き継いだメールアドレスはアプリ内の状態だけで持ち、画面を離れたとき・更新完了・更新結果不明で破棄する
    - 入力値のクライアント側チェック（再設定コードの未入力、空白の除去・全角数字の半角変換、新しいパスワード・確認入力の未入力・値の制限・不一致）と、項目ごとのエラー表示
    - Cognito の `ConfirmForgotPassword`（メールアドレス・再設定コード・新しいパスワード）の呼び出しと、送信中状態（ボタンの非活性化による二重送信・画面離脱の防止）
    - 応答ごとの表示: 成功は「更新完了」（認証状態にしない）、コード誤り・期限切れは同一のエラーバナー、ポリシー違反は新しいパスワード欄の項目エラー、試行回数の超過は回数上限のエラーバナー、トリガー関連のエラーは「更新結果不明」、通信の失敗・その他は通信エラーのエラーバナー
    - 「パスワード再設定を申請し直す」（申請画面 `/password-reset` へ）・「ログイン画面に戻る」「ログイン画面へ進む」（`/login` へ。自動では遷移しない）の導線
    - フォーカス移動・スクリーンリーダーへの通知・入力欄の `inputmode`／`autocomplete` の指定・キーボード操作の順序・最小タップ領域
  - パスワード変更画面（`/account/password`。認証必須）
    - グローバルサイドバー・画面タイトル行・「現在のパスワードの確認」セクション・「新しいパスワードの設定」セクション（共通パーツ「パスワード入力ブロック」）・他の端末のログイン状態に関する注記・「変更する」「キャンセル」・グローバルフッターを配置した画面
    - 入力値のクライアント側チェック（現在のパスワードは必須チェックのみ、新しいパスワード・確認入力の未入力・値の制限・不一致）と、項目ごとのエラー表示
    - 送信前の Access Token の `exp` 判定と、更新できない場合の認証切れの扱い（トークンを破棄してログイン画面へ遷移）
    - Cognito の `ChangePassword` → `GlobalSignOut` → 新しいパスワードでの SRP 再ログインと保持中のトークンの差し替え（新しいパスワードは再ログイン後すぐに破棄）
    - `ChangePassword` の結果ごとの表示: 現在のパスワード不一致（期限内での `NotAuthorizedException`。メッセージ文言で分岐しない）、ポリシー不適合、試行回数超過、通信・サーバーエラー
    - 変更完了系の3状態（変更完了／他の端末のログイン解除が未完了／再ログインが必要）と、②③が両方失敗したときの注意メッセージ
    - 「もう一度試す」（`GlobalSignOut` と再ログインのやり直し）・「ホームへ進む」・「ログイン画面へ進む」・「キャンセル」（前の画面、なければホーム画面へ）の動作
    - フォーカス移動・スクリーンリーダーへの通知・キーボード操作の順序・最小タップ領域
  - グローバルサイドバーのアカウントメニューの「パスワード変更」項目（`/account/password` へ遷移。本画面を開いているときは遷移せず、ドロワー表示時はメニューとドロワーを閉じる）
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-008-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-AUTH-008-password-change.md`（#12）, `specs/keiba-log/designs/architecture.md`（3.4・5.2「パスワード再設定（コード方式）」「パスワード再設定でトリガーが失敗したとき」・5.4 の `triggerSource` の値）
- **概要**:
  - Cognito の Post confirmation トリガーの処理のユニットテスト（`cognito-idp` クライアントは差し替える）
    - `triggerSource` による分岐: `PostConfirmation_ConfirmForgotPassword` ではイベントの `userName` を指定して `AdminUserGlobalSignOut` を呼ぶこと、それ以外（`PostConfirmation_ConfirmSignUp` など）では呼ばないこと、どちらも受け取ったイベントをそのまま返すこと
    - `AdminUserGlobalSignOut` が失敗したときにエラーを返すこと
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-008-*.md` の受け入れ基準をすべて満たしていること

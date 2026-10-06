# 実装タスク: REQ-AUTH-004 アクセス制御

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-AUTH-004` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-AUTH-004-access-control.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（3.1 トークン有効期間・3.3・5節「認可」）, `specs/keiba-log/designs/api.md`（2節）, `specs/keiba-log/designs/api.yml`（`paths`・`components.securitySchemes`）, `specs/keiba-log/requirements/user-stories/REQ-AUTH-004-access-control.md`
- **概要**:
  - API Gateway（HTTP API）の全ルートを JWT オーソライザーで保護する
    - 対象: `api.yml` の全メソッド・パス（`GET /summary`、`GET /records`、`POST /records`、`GET /records/recent`、`GET /records/{recordKey}`、`PUT /records/{recordKey}`、`DELETE /records/{recordKey}`、`DELETE /me`）
    - 未認証で通るルートを設けない。`$default`・`ANY /{proxy+}` は定義しない
    - JWT オーソライザーの発行元は本アプリの User Pool、オーディエンスはアプリクライアント ID
  - API 本体の Lambda に、JWT オーソライザーを経由しない呼び出し経路（関数 URL など）を設けない。関数を呼び出せるのは本アプリの HTTP API のみとする
  - 認証状態の失効に関わる Cognito のトークン有効期間（ID / Access Token 1 時間、Refresh Token 3 日）
  - 全ルートに JWT オーソライザーが付いていること・オーソライザーなしのルートがないことを確認する CDK のテスト
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（3.3・5節「認可」「データ分離」）, `specs/keiba-log/designs/api.md`（1節・2節・3節）, `specs/keiba-log/designs/api.yml`, `specs/keiba-log/designs/database.md`（1.6）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・アクセスパターンの実現方法）, `specs/keiba-log/requirements/user-stories/REQ-AUTH-004-access-control.md`
- **概要**:
  - 全エンドポイントに共通で適用するアクセス制御
    - JWT オーソライザーが検証したクレームからユーザー識別子（`sub`）を取得する。取得できないリクエストは処理せず `401`（`UNAUTHORIZED`）を返す
  - データ分離
    - DynamoDB のキー（`PK = USER#<sub>`）は JWT の `sub` からのみ組み立てる
    - パスパラメータ・クエリパラメータ・リクエストボディで指定されたユーザー識別子は受け取らず、使用しない
    - 対象: `GET /summary`、`GET /records`、`POST /records`、`GET /records/recent`、`GET /records/{recordKey}`、`PUT /records/{recordKey}`、`DELETE /records/{recordKey}`、`DELETE /me`
  - 他ユーザーの馬券記録への要求の拒否
    - `recordKey` を指定する参照・更新・削除は、JWT の `sub` のパーティション内でのみ対象を特定する
    - 他ユーザーの記録・存在しない記録を区別せず `404`（`RESOURCE_NOT_FOUND`）を返す
    - 更新・削除は条件付き書き込みで対象の存在を確認し、他ユーザーのアイテムを書き換えない
  - アカウント削除（`DELETE /me`）の対象を、JWT で特定したユーザー自身のデータと Cognito のユーザーに限定する
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/designs/ui-overview.md`（1節「独立した画面を持たない要件の扱い」「認証区分」「画面の URL」・2節）, `specs/keiba-log/designs/architecture.md`（5.2「トークンの保持と更新」）, `specs/keiba-log/designs/api.md`（3節）, `specs/keiba-log/designs/ui-designs/login.md`, `specs/keiba-log/designs/ui-designs/home.md`, `specs/keiba-log/designs/ui-designs/record-list.md`, `specs/keiba-log/designs/ui-designs/record-create.md`, `specs/keiba-log/designs/ui-designs/record-edit.md`, `specs/keiba-log/designs/ui-designs/password-change.md`, `specs/keiba-log/requirements/user-stories/REQ-AUTH-004-access-control.md`
- **概要**:
  - `ui-overview.md` 1節「認証区分」に従う画面の振り分け
    - 認証必須: ホーム画面（`/`）、馬券記録一覧画面（`/records`。記録の削除を含む）、馬券記録登録画面（`/records/new`）、馬券記録編集画面（`/records/edit`）、買い目入力モーダル、パスワード変更画面（`/account/password`）
    - 未認証で利用: ログイン画面（`/login`）、アカウント登録画面（`/signup`）、確認コード入力画面（`/verify-email`。確認コードの再送を含む）、パスワード再設定申請画面（`/password-reset`）、パスワード再設定画面（`/password-reset/confirm`）。ログインを求めずに利用できる
  - 未認証アクセス時の誘導
    - 認証必須画面を開いたとき、保持しているトークンがない、または期限切れで更新できない場合は、画面の内容を表示せず、API も呼ばずにログイン画面（`/login`）へ遷移する
    - 判定が済むまで認証必須画面の内容を表示しない
    - 元の画面のパスはクエリなどで持ち回らない
  - 認証状態の失効時の誘導
    - 認証必須画面の表示中・操作中に、トークンの更新に失敗した場合、または API の `401` が更新・再試行でも解消しない場合は、トークンを破棄してログイン画面へ遷移する
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-004-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` ユニットテスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/api.yml`, `specs/keiba-log/designs/database.md`（1.6）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・アクセスパターンの実現方法）, `specs/keiba-log/requirements/user-stories/REQ-AUTH-004-access-control.md`
- **概要**:
  - `backend/` のアクセス制御・データ分離のユニットテスト
    - クレームに `sub` がないリクエストが `401`（`UNAUTHORIZED`）になること
    - 全エンドポイントで、DynamoDB のキーが JWT の `sub` から組み立てられること
    - パスパラメータ・クエリパラメータ・リクエストボディにユーザー識別子を含めても使われないこと
    - 他ユーザーの `recordKey` を指定した `GET`・`PUT`・`DELETE /records/{recordKey}` が `404`（`RESOURCE_NOT_FOUND`）になり、他ユーザーのデータが変更されないこと
    - `GET /summary`・`GET /records`・`GET /records/recent` が自分のパーティションのデータのみを返すこと
    - `DELETE /me` が JWT で特定したユーザーのデータと Cognito のユーザーのみを削除対象にすること
- **受け入れ**: 対応する `requirements/user-stories/REQ-AUTH-004-*.md` の受け入れ基準をすべて満たしていること

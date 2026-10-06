# API設計: keiba-log

| 項目 | 内容 |
|---|---|
| 機能名 | `keiba-log` |
| 担当 | api-designer |
| インプット | `designs/architecture.md`, `designs/database.md`, `requirements/functional-requirements.md`, `requirements/user-stories/*.md` |
| 保存先 | `designs/api.md`, `designs/api.yml` |
| エンドポイント詳細 | [api.yml](./api.yml) |

## 1. API概要（ベースURL・共通ヘッダー）

API Gateway（HTTP API）で公開する JSON API。フロントエンド（Next.js）がブラウザから呼び出す唯一のバックエンド経路であり、馬券記録の登録・編集・削除・取得、生涯収支サマリーの取得、アカウント削除を提供する。

| 項目 | 内容 |
|---|---|
| ベースURL | `https://<api-id>.execute-api.<region>.amazonaws.com`（API Gateway のデフォルトエンドポイント。独自ドメインは使用しない） |
| プロトコル | HTTPS のみ |
| リクエスト形式 | JSON（UTF-8） |
| レスポンス形式 | JSON（UTF-8）。ボディを持たない場合は `204 No Content` |

### 共通ヘッダー

| ヘッダー | 方向 | 必須 | 内容 |
|---|---|---|---|
| `Authorization` | リクエスト | 必須 | `Bearer <JWT>`。Cognito User Pool から取得した ID トークンを指定する |
| `Content-Type` | リクエスト | ボディを持つ場合は必須 | `application/json` |
| `Content-Type` | レスポンス | ― | `application/json`（`204` 応答時は付かない） |

リクエストボディにユーザー識別子を含めない。操作対象のユーザーは常に JWT から決まる。

## 2. 認証・認可の概要

### 認証は Cognito User Pool に委譲する

サインアップ・ログイン・ログアウト・メールアドレス確認・パスワード再設定の申請・パスワードの変更は、ブラウザから Cognito User Pool に対して直接行う（`designs/architecture.md`）。これらに対応する API エンドポイントは本 API に存在しない。認証に関わる入力値の検証・メール送信・トークンの発行はすべて Cognito 側の責務であり、本 API は発行済みのトークンを受け取る側に立つ。

本 API がエンドポイントとして提供する認証関連の操作はアカウント削除のみである。`designs/database.md` の「アカウント削除の手順」により、DynamoDB のデータを削除し終えてから Cognito のユーザーを削除する順序制御が必要で、この順序はサーバー側でしか担保できないため、自前のエンドポイント（`DELETE /me`）を設ける。

Cognito User Pool が発行する ID トークンにはユーザーのメールアドレスが含まれるため、フロントエンドはログイン中のメールアドレス表示のために本 API を呼び出す必要はなく、ID トークンから直接取得する。

### 認可

- すべてのエンドポイントが認証必須である。未認証で利用できるエンドポイントは存在しない。
- トークンの検証は API Gateway の JWT オーソライザーが行う。トークンが無い、または無効なリクエストは Lambda に到達せず `401` となる。
- 操作対象のユーザーは JWT から取得したユーザー識別子（Cognito の `sub`）で決まる。パスパラメータ・クエリパラメータ・リクエストボディで指定されたユーザー識別子は受け取らず、使用しない。
- 他ユーザーの馬券記録は識別子を指定しても到達できない。DynamoDB のキーにユーザー識別子が含まれるため（`designs/database.md` 1.6）、他ユーザーの記録を指定した参照・更新・削除は存在しないものとして `404` を返す。存在の有無を区別しないことで、他ユーザーのデータの存在自体も推測できないようにする。

## 3. 共通エラーレスポンス形式

すべてのエラー応答は次の形式に統一する。

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "指定されたリソースが見つかりません",
    "details": []
  }
}
```

| フィールド | 内容 |
|---|---|
| `error.code` | エラー種別を表す固定の識別子。クライアントの分岐はこの値で行う |
| `error.message` | 利用者に提示できる日本語のメッセージ |
| `error.details` | 項目単位の付加情報の配列。該当がない場合は空配列。入力値が不正な場合は、項目ごとに `field`（対象項目のパス）と `message` を持つ要素を並べる |

| HTTPステータス | `error.code` | 発生条件 |
|---|---|---|
| `400` | `VALIDATION_ERROR` | リクエストの形式または値が不正。クエリパラメータの範囲外・型不正も含む（例: `GET /records/recent` の `limit` が 1〜20 の整数でない場合。範囲内に丸めない）。不正な項目を `details` に列挙する |
| `401` | `UNAUTHORIZED` | トークンが無い、無効、または期限切れ |
| `404` | `RESOURCE_NOT_FOUND` | 対象の馬券記録が存在しない、または自分の記録でない |
| `409` | `CONFLICT` | 同一ユーザーのデータに対する書き込みが競合した |
| `500` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー側エラー |

入力値の検証は、`requirements/business-rules.md` の「エンティティ項目定義」および算出・生成ルールに定義された制約に従ってサーバー側で行う。クライアント側の検証の有無にかかわらず、サーバーで不正な値を受け付けない。

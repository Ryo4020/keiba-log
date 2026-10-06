# 実装タスク: 基盤

| 項目 | 内容 |
|---|---|
| 担当 | task-planner |
| 対応要件ID | なし（全機能共通の基盤実装） |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` インフラ基盤
- **担当**: aws-developer
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（1節・3節・3.1〜3.5・4節・5節・5.4・5.5）, `specs/keiba-log/designs/ui-overview.md`（1節「画面の URL」の出力ファイル）, `specs/keiba-log/designs/database.md`（3節・4節）, `specs/keiba-log/designs/db-tables/keiba-log.md`（概要・PK/SK、GSI・LSI を定義しないこと）, `specs/keiba-log/designs/api.md`（1節・2節）, `specs/keiba-log/designs/api.yml`（`paths`）, `specs/keiba-log/requirements/business-rules.md`（エンティティ項目定義 > アカウント）
- **概要**:
  - `infra/` に AWS CDK（TypeScript）プロジェクトを初期化し、スタックの骨格を作る（環境名を受け取りリソース名に付与できる構成）
  - リポジトリ直下の `README.md`（構成と各プロジェクトのビルド・デプロイ手順）
  - S3 + CloudFront による `frontend/` の静的エクスポート成果物の配信
    - S3: パブリックアクセス禁止、OAC 経由のアクセスのみ許可、サーバーサイド暗号化
    - CloudFront: デフォルトドメイン（`*.cloudfront.net`）と AWS 管理の証明書、HTTP から HTTPS へのリダイレクト
    - CloudFront Functions（ビューワーリクエスト）: 拡張子のないパスを静的エクスポートの出力ファイル名に書き換える汎用ルールを 1 つ（`ui-overview.md` 1節「画面の URL」の出力ファイル名に合わせる。`/` → `index.html`、`/records/edit` → `records/edit.html` など）。存在しないパスは `404.html` を返す
    - レスポンスヘッダーポリシー（全ページ共通の 1 つ）: 5.5 の CSP、HSTS、`X-Content-Type-Options`、フレーム埋め込みの禁止
      - CSP の `<region>`・`<api-id>` は、CDK が環境ごとにデプロイ時の値を埋め込む
  - Cognito User Pool（3.1）
    - サインイン識別子はメールアドレス（ユーザー名としてメールアドレスを使う）、セルフサインアップ有効
    - メールアドレスの自動確認、確認メッセージはコード方式
    - パスワードポリシー（最小 8 文字、半角英大文字・半角英小文字・数字・記号を各 1 文字以上）
    - パスワード再設定はコード方式、アカウントの復旧手段は確認済みのメールアドレスのみ
    - メール送信は Cognito の標準メール機能（SES は使わない。3.2）
    - ユーザー存在エラーの抑止を有効、トークン失効を有効
    - トークン有効期間: ID / Access Token は既定（1 時間）、Refresh Token は 3 日
    - ドメイン・Managed Login は設定しない。未確認アカウントを削除する仕組みは作らない
    - 機能プランは Lite
  - Cognito アプリクライアント（1 つ。クライアントシークレットなし。許可する認証フローは `ALLOW_USER_SRP_AUTH`・`ALLOW_REFRESH_TOKEN_AUTH` のみ）
  - API Gateway（HTTP API）（3.3）
    - デフォルトエンドポイント（独自ドメインは使わない）
    - `api.yml` の各メソッド・パスを個別のルートとして定義し、すべて API 本体の Lambda 統合に向ける。`$default`・`ANY /{proxy+}` は使わない
    - JWT オーソライザー（発行元 `https://cognito-idp.<region>.amazonaws.com/<userPoolId>`、オーディエンスはアプリクライアント ID）を全ルートに付ける
    - CORS: 許可するオリジンは CloudFront のドメイン（開発環境のみ `http://localhost:3000` を加える。`*` は使わない）、メソッドは `GET`・`POST`・`PUT`・`DELETE`、ヘッダーは `Authorization`・`Content-Type`、`allowCredentials` は無効、`maxAge` は 1 時間
  - API 本体の Lambda 関数（`backend/` の API 本体のエントリーポイントのビルド済みバイナリ、`provided.al2023`、arm64、VPC に配置しない）
  - API 本体の Lambda 実行ロール（関数専用。次のみを許可）
    - `keiba-log` テーブルへの必要な操作
    - `cognito-idp:AdminDeleteUser`（対象は本アプリの User Pool の ARN のみ）
    - 自関数のロググループへの書き込み
  - CloudWatch Logs: API 本体の関数のロググループを CDK で明示的に作り、保持期間を 30 日にする
  - DynamoDB テーブル `keiba-log`（キー定義、GSI・LSI は定義しない、オンデマンドキャパシティ、PITR、削除保護、サーバーサイド暗号化）と、スロットリング・システムエラーの監視
  - フロントエンド・バックエンドが参照する設定値（リージョン、User Pool ID、アプリクライアント ID、API エンドポイント、テーブル名など）の出力・受け渡し
  - Cognito の Post confirmation トリガー（トリガー用の Lambda 関数、その実行ロール・リソースベースポリシー・ロググループ、User Pool へのトリガーの設定）は本タスクの対象外とする。本タスクでは User Pool に Lambda トリガーを設定しない
- **受け入れ**: `designs/architecture.md` のインフラ構成（3節・3.1〜3.5・5節・5.5。Post confirmation トリガーに関する部分を除く）、`designs/database.md` 4節、`designs/db-tables/keiba-log.md` と一致していること

### `[ ]` バックエンド基盤
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（1節・2節・3.3・3.4・4節・5節）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/api.yml`（`components.securitySchemes`・`components.responses`・`ErrorResponse`・`ErrorDetail`）, `specs/keiba-log/designs/database.md`（1.6）, `specs/keiba-log/designs/db-tables/keiba-log.md`（概要・PK/SK）
- **概要**:
  - `backend/` に Go モジュールと gin を初期化し、クリーンアーキテクチャの骨格を作る
  - エントリーポイントごとに別のバイナリとしてビルドできる構成（`provided.al2023`・arm64 向け）と、API 本体のエントリーポイント
  - API Gateway（HTTP API）のペイロード（バージョン 2.0）を gin のリクエストに変換する Lambda アダプター
  - JWT オーソライザーが検証したクレームからユーザー識別子（`sub`）を取得する共通処理
  - 共通エラーレスポンス（`api.md` 3節の形式・エラーコード）と、想定外エラー時の `500` 応答
  - CORS ヘッダーは gin で付けない（API Gateway の CORS 設定で処理する）
  - CloudWatch Logs に出力するアプリケーションログの共通処理（JWT・メールアドレス・入力値の全体を出さない）
  - DynamoDB クライアント・Cognito（`cognito-idp`）クライアントの初期化と、テーブル名・User Pool ID などの設定値の読み込み
  - Cognito の Post confirmation トリガーのエントリーポイントと処理は本タスクの対象外とする
- **受け入れ**: `designs/architecture.md` の技術構成・Lambda の構成・セキュリティ設計（2節・3.3・3.4・4節・5節。Post confirmation トリガーに関する部分を除く）および `designs/api.md` 1節〜3節と一致していること

### `[ ]` フロントエンド基盤
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（1節〜5節・3.5・5.1・5.2「トークンの保持と更新」・5.4・5.5）, `specs/keiba-log/designs/ui-overview.md`（1節の認証区分・1節の画面の URL・2節の画面遷移の補足・3節〜5節・4節のタイポグラフィ）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/api.yml`（`components.securitySchemes`・`components.responses`・`ErrorResponse`・`ErrorDetail`）, `specs/keiba-log/designs/ui-designs/signup.md`・`login.md`・`email-verification.md`・`password-reset-request.md`・`password-reset.md`（メールアドレスの引き継ぎの記載）
- **概要**:
  - `frontend/` に Next.js（App Router）と Tailwind CSS を初期化し、静的エクスポート（`output: 'export'`）でビルドする構成にする
  - 静的エクスポートに合わせたルート構成
    - `ui-overview.md` 1節「画面の URL」の全パスのページを置く（ページの中身は各要件のタスクで実装する）
    - 末尾スラッシュなしの出力にする（`/` → `index.html`、`/records/edit` → `records/edit.html` など、`ui-overview.md` 1節「画面の URL」の出力ファイル名と一致させる）
    - 動的ルートを使わず、画面に渡すパラメータはクエリパラメータで受け取る（編集画面は `/records/edit?key=<recordKey>`。`recordKey` は URL エンコードする）
    - `useSearchParams` を使うページを Suspense 境界で囲む構成
    - 存在しないパス用のページ（`404.html` として出力される）
  - `ui-overview.md` 4節のデザイントークン（カラー・タイポグラフィ・スペーシング・角丸・影・レイアウト寸法）と 5節のブレークポイントの Tailwind 設定
  - フォント設定（Noto Sans JP を `next/font/google` でセルフホスト（ウェイト 400・700、`font-display: swap`）し、`font-family-base` の先頭に使う。`font-numeric` も設定する。Inter は使用しない）
  - ルートレイアウト（言語・画面背景・本文の基本スタイル・フォーカスリング）
  - 未認証で利用する画面群と認証必須の画面群を分けるルート構成と、それぞれのレイアウトの枠
    - 未認証で認証必須画面（`/` を含む）を開いたときに `/login` へ遷移させる仕組み（認証必須画面群のレイアウトで判定する。元の画面のパスは持ち回らない）
  - Cognito User Pool の認証クライアント
    - ブラウザから User Pool の API を直接呼ぶ接続設定（リージョン・User Pool ID・アプリクライアント ID。クライアントシークレットなし）
    - SRP 認証（`USER_SRP_AUTH`）
    - ID Token・Access Token・Refresh Token の `localStorage` への保持・取得・削除
    - `exp` による有効期限（期限間近を含む）の判定と、`InitiateAuth`（`REFRESH_TOKEN_AUTH`）によるトークンの更新
    - 更新に失敗したときのトークンの破棄とログイン画面への遷移
  - API クライアント
    - ベース URL、`Authorization: Bearer <ID トークン>` の付与、JSON の送受信、`204` の扱い、共通エラーレスポンスの解釈
    - 呼び出し前の `exp` 判定と期限間近の場合の更新、`401` を受けたときの 1 回だけの更新と再試行、更新に失敗したときのトークンの破棄とログイン画面への遷移
  - 未認証画面の間でメールアドレスを引き継ぐ仕組み（画面遷移時のアプリ内の状態で値を渡し、URL・クエリ文字列に含めない。再読み込み・URL を直接開いた場合は引き継がれていない状態として扱える。値の受け渡しと破棄ができる）
  - XSS 対策の前提（`architecture.md` 5.5 の CSP の下で動作する）
    - 外部のスクリプト・外部のフォントを読み込まない
    - 通信先は自オリジン・Cognito User Pool の API・API Gateway のエンドポイントのみ
    - フォームはブラウザ標準の送信を使わず、JavaScript で処理する
    - `dangerouslySetInnerHTML` などで HTML を直接挿入しない
  - User Pool ID・アプリクライアント ID・API エンドポイントなどの環境ごとの設定値の読み込み
  - `ui-overview.md` 6節の共通パーツと、各画面でのメールアドレスの引き継ぎの使い方（引き継ぐ値・破棄するタイミング）は本タスクの対象外とする
- **受け入れ**: `designs/architecture.md`（3.5・5節・5.2「トークンの保持と更新」・5.5）・`designs/ui-overview.md`（3節〜5節）の内容、および `designs/ui-designs/` の各画面に記載されたメールアドレスの引き継ぎの方式と一致していること

### `[ ]` テスト基盤
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/designs/architecture.md`（2節・3.4）, `specs/keiba-log/designs/api.md`（2節・3節）
- **概要**:
  - `backend/` の Go ユニットテストの土台（`testing` と testify、`go test ./...` での実行）
  - DynamoDB クライアント・Cognito（`cognito-idp`）クライアントなどの外部依存をテストで差し替える仕組みと、gin のハンドラーをテストするための共通ヘルパー
  - バックエンド基盤の共通処理（ユーザー識別子の取得・共通エラーレスポンス・想定外エラー時の `500` 応答）のユニットテスト
  - Cognito の Post confirmation トリガーのユニットテストは本タスクの対象外とする
- **受け入れ**: `designs/architecture.md` の技術構成と、`designs/api.md` 2節・3節の認可・エラーレスポンスの仕様と一致していること

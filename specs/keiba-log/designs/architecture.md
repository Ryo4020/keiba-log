# アーキテクチャ設計: keiba-log

| 項目 | 内容 |
|---|---|
| 機能名 | `keiba-log` |
| 担当 | system-architect |
| インプット | `requirements/overview.md`, `requirements/non-functional-requirements.md` |
| 保存先 | `designs/architecture.md` |
| Miroボード | [keiba-log アーキテクチャ構成図](https://miro.com/app/board/uXjVHp_j1j0=/?moveToWidget=3458764684070555711) / [keiba-log 認証フロー](https://miro.com/app/board/uXjVHp_j1j0=/?moveToWidget=3458764685918220982) |

## 1. システム全体のアーキテクチャ

ユーザーのブラウザ（スマートフォン / PC）から利用するレスポンシブ Web アプリで、画面配信・認証・API・データ保存をそれぞれマネージドサービスに分離した構成をとる。

構成要素と繋がりは次の通り。

- **クライアント**: ブラウザ上で動作する Next.js アプリ（静的エクスポートしたファイルを配信）。登録・確認コード入力・ログイン・パスワード再設定を含むすべての画面をアプリ自身が持つ。
- **配信レイヤー**: CloudFront（`*.cloudfront.net`）が HTTPS で画面を配信し、オリジンの S3 から Next.js のビルド成果物を返す。API は経由しない。
- **認証レイヤー**: ブラウザが Cognito User Pool の API（`cognito-idp` の公開エンドポイント）を直接呼び、登録・確認コードの検証・ログイン・トークン更新・ログアウト・パスワード変更・パスワード再設定を行う。トークンはブラウザが保持する（5.2）。確認メール・パスワード再設定メールは Cognito の標準メール機能で送る（3.2）。
- **API レイヤー**: API Gateway（HTTP API）が `Authorization: Bearer <ID トークン>` を JWT オーソライザーで検証し、通ったリクエストだけを Lambda に渡す。ブラウザから別オリジンの API を呼ぶため CORS を設定する。
- **アプリケーション**: Lambda 上の Go + gin が API 本体（馬券の登録・編集・削除・一覧、生涯収支の算出、アカウント削除）を実行する。ユーザー識別子はオーソライザーが検証したクレームの `sub` から取得する。これとは別に、Cognito の Post confirmation トリガーとして Go の Lambda 関数を 1 つ置き、パスワード再設定の完了時に他端末の認証状態を失効させる（3.4・5.2）。
- **データ**: 馬券・収支データは DynamoDB の `keiba-log` テーブルに保存する。実行ログは CloudWatch Logs に出力する。

データフローは「ブラウザ → CloudFront → S3（画面）」「ブラウザ ⇄ Cognito User Pool（認証）」「ブラウザ → API Gateway → Lambda → DynamoDB（データ）」の 3 系統。Lambda と Cognito の間のやり取りは 2 つに限る。アカウント削除時に API 本体の Lambda が呼ぶ `AdminDeleteUser` と、`ConfirmForgotPassword` の処理中に Cognito がトリガー用の Lambda を同期で呼び出し、その Lambda が Cognito に対して呼ぶ `AdminUserGlobalSignOut`（Cognito → トリガー用 Lambda → Cognito）。外部データ連携は行わず、レースや馬券の情報はすべてクライアントからの入力として API 経由で保存される。

## 2. 技術スタック

| 領域 | 技術 |
|---|---|
| フロントエンド | Next.js（App Router）、Tailwind CSS |
| バックエンド | Go言語、ginフレームワーク |
| データベース | Amazon DynamoDB |
| インフラ | AWS（サーバーレス・コスト最適化優先） |
| IaC | AWS CDK（TypeScript） |

## 3. クラウドアーキテクチャ

リクエストがない間は課金が発生しない構成を基本方針とし、常時稼働のコンピュートリソースやマネージドな仮想ネットワーク構成は使用しない。Lambda は VPC に配置しない。

| AWSサービス | 用途 |
|---|---|
| Amazon CloudFront | 画面の配信。デフォルトドメイン（`*.cloudfront.net`）と AWS 管理の証明書で HTTPS を提供する。CloudFront Functions で URL を S3 のファイル名に書き換える（3.5） |
| Amazon S3 | Next.js 静的エクスポートの成果物の保管（パブリックアクセスは禁止し、CloudFront の OAC 経由でのみ配信） |
| Amazon Cognito User Pool | メールアドレス + パスワード認証、確認コードによるメールアドレス確認、コード方式のパスワード再設定、トークン発行、確認メール・再設定メールの送信（3.1・3.2） |
| Amazon API Gateway（HTTP API） | JWT オーソライザーによるトークン検証、CORS、Lambda への転送（3.3） |
| AWS Lambda（API 本体） | Go + gin による API 本体の実行。単一の関数（3.4） |
| AWS Lambda（Post confirmation トリガー） | パスワード再設定の完了時に `AdminUserGlobalSignOut` で他端末の認証状態を失効させる。Go、API 本体とは別の関数（3.4・5.2） |
| Amazon DynamoDB（`keiba-log`） | 馬券・収支データの永続化（オンデマンドキャパシティ、PITR 有効） |
| Amazon CloudWatch Logs | 各 Lambda のログの記録（3.4） |
| AWS IAM | 各リソースへの最小権限アクセス制御 |

### 3.1 Cognito の設定

| 項目 | 設定 |
|---|---|
| サインイン識別子 | メールアドレス（ユーザー名としてメールアドレスを使う） |
| セルフサインアップ | 有効 |
| メールアドレス確認 | 自動確認の対象をメールアドレスにし、確認メッセージは**コード方式**にする |
| パスワードポリシー | 最小 8 文字。半角英大文字・半角英小文字・数字・記号をそれぞれ 1 文字以上必須にする。最大文字数と記号として扱う文字の範囲は Cognito の仕様に従う |
| パスワード再設定 | コード方式（`ForgotPassword` → `ConfirmForgotPassword`、5.2）。アカウントの復旧手段は確認済みのメールアドレスのみ |
| メール送信 | Cognito の標準メール機能（3.2） |
| Lambda トリガー | Post confirmation のみ。トリガー用の関数（3.4）を設定する。ほかのトリガーは使わない |
| 未確認アカウント | 削除の仕組みは作らず、Cognito に残したままにする。確認コードの期限切れは再送で対応する |
| ドメイン・Managed Login | 使わない（画面はすべてアプリが持つ） |
| アプリクライアント | 1 つ。クライアントシークレットなし（ブラウザから直接呼ぶため秘密を保持できない） |
| 許可する認証フロー | `ALLOW_USER_SRP_AUTH`、`ALLOW_REFRESH_TOKEN_AUTH` のみ。SRP によりパスワードそのものは Cognito にも送らない |
| ユーザー存在エラーの抑止 | 有効（ログイン失敗時・パスワード再設定の申請時にメールアドレスの存在有無を区別しない） |
| トークン失効 | 有効（ログアウトで `RevokeToken` を使う） |
| トークン有効期間 | ID / Access Token は既定（1 時間）。Refresh Token は認証状態の有効期間に合わせて 3 日 |
| 機能プラン | Lite で必要な機能（パスワード認証・確認コード・パスワード再設定・標準メール機能・トークン失効・Post confirmation トリガー）が揃う見込み（5.4 で要確認） |

### 3.2 メール送信（Cognito 標準メール機能）

確認メール（`SignUp`・`ResendConfirmationCode`）とパスワード再設定メール（`ForgotPassword`）は、Cognito の標準メール機能で送る。SES は使わない。

| 項目 | 内容 |
|---|---|
| 送信数の上限 | 1 AWS アカウントあたり 1 日 50 通。引き上げはできない。開発環境と本番環境を同じ AWS アカウントに置くと上限を共有する |
| 送信元 | Cognito 既定のアドレス（`no-reply@verificationemail.com`）。変更できない |
| 準備 | 不要。User Pool のメール設定で Cognito の標準メール機能を選ぶだけで、送信元の検証や申請はない |
| 料金 | 追加料金なし |
| 送信数を抑える方針 | 画面の遷移に合わせてコードを自動で再送しない。再送はユーザーの操作でのみ行う（5.2） |

上限を超えそうになった場合の選択肢:

- **SES に切り替える**: User Pool のメール設定を SES に変え、送信元のメールアドレス ID を SES で検証し、本番環境はサンドボックス解除を申請する。独自ドメインがない場合、フリーメール等の From では DMARC により迷惑メール扱いになるおそれがあり、申請が通らない可能性もある。
- **独自ドメインを取得して SES に切り替える**: SES でドメイン ID を検証し、そのドメインの From で送る。到達率の懸念は小さくなるが、ドメインの取得・DNS の管理が発生する。
- **開発環境を別の AWS アカウントに分ける**: 本番の上限を開発・テストで消費しない。上限そのものは増えない。

### 3.3 API Gateway（ルート・JWT オーソライザー・CORS）

| 項目 | 設定 |
|---|---|
| エンドポイント | デフォルトエンドポイント（`https://<api-id>.execute-api.<region>.amazonaws.com`）。独自ドメインは使わない |
| ルート | `api.yml` の各メソッド・パスを個別のルートとして定義し、すべて同じ Lambda 統合に向ける。`$default` や `ANY /{proxy+}` は使わない（未定義のパスを Lambda に到達させず、CORS のプリフライトがオーソライザー付きのルートに当たらないようにするため） |
| JWT オーソライザー | 発行元 `https://cognito-idp.<region>.amazonaws.com/<userPoolId>`、オーディエンスはアプリクライアント ID。全ルートに付ける。ID トークンを受け取り（`api.md` 1 節）、署名・`iss`・`aud`・`exp` を検証する |
| CORS: 許可するオリジン | `https://<distribution>.cloudfront.net`。開発環境のみ `http://localhost:3000` を加える。`*` は使わない |
| CORS: 許可するメソッド | `GET`、`POST`、`PUT`、`DELETE` |
| CORS: 許可するヘッダー | `Authorization`、`Content-Type` |
| CORS: その他 | 認証情報（Cookie）を使わないため `allowCredentials` は無効。プリフライトの結果のキャッシュ（`maxAge`）は 1 時間 |

CORS は API Gateway の CORS 設定で行い、プリフライト（`OPTIONS`）には API Gateway が応答する。gin では CORS ヘッダーを付けない。

### 3.4 Lambda と CloudWatch Logs

| 項目 | 方針 |
|---|---|
| 関数の構成 | 2 つ。API 本体は単一の関数で gin が全ルートを受ける（API の規模が小さく、関数を分けてもデプロイと設定の手間が増えるだけのため）。Post confirmation トリガーは API 本体とは別の関数にする（呼び出し元・入力・必要な権限が API と異なり、API 本体に `AdminUserGlobalSignOut` の権限を持たせないため） |
| ランタイム | どちらも OS 専用ランタイム（`provided.al2023`）で Go のビルド済みバイナリを実行する。アーキテクチャは arm64。トリガー用も Go にするのは、API 本体とビルド・デプロイの手順とランタイムを揃えられ、AWS SDK for Go から `AdminUserGlobalSignOut` を呼べるため（Node.js にすると言語とビルド手順が 2 系統になる） |
| コードの配置 | どちらも `backend/` に置き、別々のエントリーポイントとしてビルドする |
| API 本体のイベント | HTTP API のペイロード（バージョン 2.0）を gin のリクエストに変換するアダプターを使う |
| トリガー用の関数 | gin は使わず、Post confirmation のイベントを直接受ける。`triggerSource` が `PostConfirmation_ConfirmForgotPassword` のときだけ、イベントの `userName` を指定して `AdminUserGlobalSignOut` を呼ぶ。それ以外（登録の確認による起動）は何もしない。どちらの場合も受け取ったイベントをそのまま返す。`AdminUserGlobalSignOut` が失敗したらエラーを返す（5.2）。タイムアウトは Cognito がトリガーの応答を待つ時間（5 秒の想定、5.4）以内にする |
| ログ | 関数ごとにロググループを CDK で明示的に作り、保持期間を 30 日にする（既定の「失効しない」ではログが溜まり続けるため）。アプリのログに JWT・メールアドレス・入力値の全体を出さない。トリガー用の関数は `triggerSource` と処理結果（失敗時はエラー）だけを出し、`userName`（メールアドレス）は出さない |

### 3.5 静的配信（CloudFront + S3）と動的な URL

Next.js は静的エクスポートで S3 に置く。静的エクスポートではビルド時に決まらないパス（例: `/records/[recordKey]/edit`）のページを生成できないため、**編集画面はクエリパラメータ方式**（`/records/edit?key=<recordKey>`）にする。

| 案 | 判断 |
|---|---|
| クエリパラメータ方式 | **採用**。ページは `/records/edit` の 1 枚で静的に生成でき、`recordKey` はクライアントで読み取って API を呼ぶ。インフラの追加がない。`recordKey` は URL エンコードして渡す |
| CloudFront Functions でパスを書き換える | 不採用。`/records/<key>/edit` を 1 枚の HTML に書き換えても、Next.js 側はビルド時に存在しないパスとしてパラメータを扱えず、パスをアプリが自前で解析することになる。書き換えのルールも画面の追加に合わせて保守が要る |

画面の URL（パス・クエリ）の一覧は `designs/ui-overview.md` 1 節「画面の URL」を正とする。パスは末尾スラッシュなしで、クエリを使うのは編集画面の `key` だけ。

Next.js の `trailingSlash` は既定の `false` のままにする。静的エクスポートはこのとき各ページを「パス + `.html`」のファイル名で出力する前提で（`/records/edit` → `records/edit.html`、`/` → `index.html`。5.4 で要確認）、ui-overview の「出力ファイル（参考）」と一致する。

S3 を OAC 経由のオリジンにすると、拡張子のないパスは `.html` のファイルに解決されない。CloudFront Functions（ビューワーリクエスト）に次の汎用ルールを 1 つ置き、画面を追加してもルールの変更は要らないようにする。存在しないパスは `404.html` を返す。

| リクエストのパス | 書き換え後（S3 のキー） |
|---|---|
| `/` | `index.html` |
| 末尾スラッシュなしで拡張子のないパス（例: `/login`、`/records`、`/records/edit`） | 末尾に `.html` を付ける（`login.html`、`records.html`、`records/edit.html`） |
| 拡張子のあるパス（`/_next/...` のスクリプト・CSS・フォント、`/404.html` など） | 書き換えない |
| 末尾スラッシュ付きのパス（`/` を除く。例: `/records/`） | 書き換えない。正のパスではないため、該当するファイルがなく `404.html` になる |

クエリは書き換えの判定にもキャッシュキーにも使わない。`key` の値によらず同じ `records/edit.html` を返し、`key` はクライアントで読む。

## 4. ディレクトリ構成

```
keiba-log/
├── frontend/    Next.js アプリケーション
├── backend/     Go による API（gin）と Cognito の Post confirmation トリガー
├── infra/       AWS CDK（TypeScript）
├── specs/       仕様ドキュメント
└── README.md
```

## 5. セキュリティ設計

- **認証**: Cognito User Pool によるメールアドレス + パスワード認証。ブラウザが Cognito を直接呼ぶ（5.2）。メールアドレス確認を完了するまでログインできない。
- **認可**: API はすべて JWT オーソライザーで ID トークンを検証する。トークンが無い・無効・期限切れのリクエストは Lambda に到達せず `401` になる。
- **データ分離**: DynamoDB のキー設計にユーザー識別子（Cognito の `sub`）を含め、Lambda はオーソライザーが検証したクレームの `sub` のデータのみを操作する。リクエストのボディ・クエリ・パスで指定された値は信用しない。
- **トークンの保護**: トークンはブラウザの `localStorage` に保持する（理由と対策は 5.2）。XSS がトークン漏えいに直結するため、画面に外部のスクリプトを読み込まず、CloudFront のレスポンスヘッダーポリシーでセキュリティヘッダー（CSP、HSTS、`X-Content-Type-Options`、フレーム埋め込みの禁止）を付ける。CSP のディレクティブと、インラインスクリプトの扱いは 5.5 に示す。
- **CORS**: API が受け付けるオリジンを CloudFront のドメインに限定する（3.3）。Cookie を使わないため CSRF の対策は不要。
- **通信の保護**: クライアントと AWS 間の通信はすべて HTTPS。CloudFront のデフォルトドメインと AWS 管理の証明書を使い、独自ドメイン・証明書の運用は発生しない。ビューワーからの HTTP アクセスは HTTPS へリダイレクトする。
- **オリジンへのアクセス制限**: S3 バケットはパブリックアクセスを禁止し、CloudFront（OAC）からのアクセスのみを許可する。
- **保管データの保護**: S3 と DynamoDB はサーバーサイド暗号化を有効にする。
- **Lambda の権限**: 関数ごとに実行ロールを分け、次のみを許可する。
  - API 本体
    - `keiba-log` テーブルへの必要な操作
    - `cognito-idp:AdminDeleteUser`（対象は本アプリの User Pool の ARN のみ。`DELETE /me` で使う）
    - CloudWatch Logs への書き込み（自関数のロググループのみ）
  - Post confirmation トリガー
    - `cognito-idp:AdminUserGlobalSignOut`（対象は本アプリの User Pool の ARN のみ）
    - CloudWatch Logs への書き込み（自関数のロググループのみ）
    - DynamoDB への権限は持たない。関数を呼び出せるのは本アプリの User Pool のみとする（関数のリソースベースポリシー）

### 5.1 パスワード変更時の認証状態の制御（`REQ-AUTH-008` 補足）

ログイン中の画面操作によるパスワード変更（再設定経由は 5.2）は、ブラウザが Cognito を直接呼んで行う。「操作した端末はログインを維持する」「他の端末の認証状態は失効させる」の両立と、「現在のパスワード不一致」と「認証切れ」の判別が必要になる。

**処理の流れ**

1. 送信前に Access Token の `exp` を判定する。期限切れ（または期限間近）なら Refresh Token で更新を試み、更新できなければ認証切れとしてトークンを破棄し、ログイン画面へ遷移する。
2. 有効な Access Token と現在のパスワード・新しいパスワードで `ChangePassword` を呼ぶ。
3. 成功したら `GlobalSignOut` を呼び、このユーザーのすべての Refresh Token を失効させる（他端末を含む）。
4. 手元にある新しいパスワードで再ログイン（SRP 認証）し、得たトークンで保持中のトークンを差し替える。新しいパスワードは再ログイン後すぐに破棄する。
5. 再ログインに失敗した場合は、トークンを破棄し、パスワードが変更済みであることと新しいパスワードでログインし直す必要があることを画面で伝える。ログイン画面へは自動で遷移させず、ユーザーの操作で進ませる（`designs/ui-designs/password-change.md` の「変更完了（再ログインが必要）」状態）。

**「現在のパスワード不一致」と「認証切れ」の判別**

- 手順 1 で Access Token が期限内であることを確かめてから送るため、期限内で `ChangePassword` が `NotAuthorizedException` を返した場合は「現在のパスワード不一致」として扱う。
- 例外のメッセージ文言による分岐は行わない。

**制約**

- `GlobalSignOut` で失効するのは Refresh Token（と Cognito の API に対する Access Token）であり、API Gateway の JWT オーソライザーは失効を確認しない。他端末が持つ ID / Access Token は、有効期限（最大 1 時間）が切れるまで API を呼べる。他端末はその後のトークン更新に失敗し、ログイン画面へ遷移する。

### 5.2 認証フロー

シーケンスは Miro の「[keiba-log 認証フロー](https://miro.com/app/board/uXjVHp_j1j0=/?moveToWidget=3458764685918220982)」に示す。画面はすべてアプリが持ち、ブラウザから Cognito User Pool の API を直接呼ぶ。

**登録 → 確認コード入力 → ログイン**

1. アカウント登録画面でメールアドレスとパスワードを入力し、`SignUp` を呼ぶ。User Pool がユーザーを作成し（`sub` 生成、未確認）、標準メール機能で確認コードを載せた確認メールを送る。
2. 確認コード入力画面へ遷移する。ユーザーがメールのコードを入力し、`ConfirmSignUp` を呼ぶ。成功するとアカウントが有効になる。このとき Post confirmation トリガーも起動するが、トリガー用の関数は何もしない。
3. ログイン画面へ遷移し、ユーザーがメールアドレスとパスワードでログインする（確認後の自動ログインは行わない）。

トリガーの実行そのものが失敗した場合（Lambda の障害など）、`ConfirmSignUp` はトリガー関連のエラーを返す想定で、アカウントが確認済みになったかは分からない（5.4）。確認コード入力画面は確認の完了を断定せず、ログインを試すよう案内する。未確認のままならログイン時に `UserNotConfirmedException` となり、確認コード入力画面へ戻る（下記）。

**登録済みのメールアドレスで登録したとき（`UsernameExistsException`）**

`SignUp` はユーザー存在エラーの抑止の対象外で、そのメールアドレスのユーザーが既にあれば、未確認・確認済みを問わず `UsernameExistsException` を返す（5.4 で要確認）。登録画面でアカウントの存在有無を区別しないため、次の方針で扱う（画面の詳細は UI 設計で決める）。

- `UsernameExistsException` でも成功時と同じ案内（確認メールを送ったこと）を表示し、確認コード入力画面へ進める。コードの自動再送はしない。既存のアカウントが未確認なら、ユーザーは再送操作で新しいコードを受け取れる。
- 確認コード入力画面では、アカウントの状態で表示が変わらないように応答をまとめる。`ConfirmSignUp` の誤り・期限切れ・確認済みアカウントへの確認（`NotAuthorizedException` の想定）はすべて同じ「コードが正しくないか有効期限切れ」の表示にし、`ResendConfirmationCode` は確認済みアカウントで失敗した場合（`InvalidParameterException` の想定）も成功と同じ表示にする。
- 制約: 既存の未確認アカウントには、最初の登録時のパスワードが残り、今回入力したパスワードは反映されない。確認後にログインできない場合はパスワード再設定を使う。

**未確認アカウントでのログイン**

- ログイン（`InitiateAuth` の SRP 認証）が `UserNotConfirmedException` を返した場合は、確認コード入力画面へ遷移させる。トークンは発行されない。
- 遷移時にコードを自動で再送はしない（送信数を抑える）。手元のコードが期限切れ・紛失の場合は再送操作を使う。

**確認コードの再送**

- 確認コード入力画面の再送操作で `ResendConfirmationCode` を呼ぶ。コードの誤り（`CodeMismatchException`）・期限切れ（`ExpiredCodeException`）のときも再送へ誘導する。

**ログイン**

- `InitiateAuth`（`USER_SRP_AUTH`）で認証し、ID Token・Access Token・Refresh Token を受け取る。
- 認証失敗はメールアドレスの存在有無を区別せずに表示する（ユーザー存在エラーの抑止）。

**パスワード再設定（コード方式）**

1. パスワード再設定申請画面でメールアドレスを入力し、`ForgotPassword` を呼ぶ。確認済みのアカウントであれば、Cognito が標準メール機能で再設定コードを載せたメールを送る。ユーザー存在エラーの抑止により、存在しないメールアドレスでも成功と同じ応答になるため、画面は存在有無を区別しない案内を表示して再設定画面へ進む。
2. パスワード再設定画面で、メールに届いた再設定コードと新しいパスワードを入力し、`ConfirmForgotPassword`（メールアドレス・再設定コード・新しいパスワード）を呼ぶ。メールアドレスは申請画面から引き継ぐ。
3. `ConfirmForgotPassword` の処理の中で Cognito が Post confirmation トリガーを同期で呼び出し、トリガー用の関数が `AdminUserGlobalSignOut` でこのユーザーのすべての Refresh Token を失効させる（他端末を含む）。ブラウザは失効の処理を行わない。成功が返ったらログイン画面へ進む。自動ログインはしない。

| 応答 | 扱い |
|---|---|
| `CodeMismatchException` | コードの誤り。入力し直すか、申請画面から再申請（`ForgotPassword` を再度呼ぶ）へ誘導する |
| `ExpiredCodeException` | コードの期限切れ。再申請へ誘導する |
| `InvalidPasswordException` | パスワードポリシー違反。画面での事前検証で防ぎ、届いた場合は不正な項目として示す |
| `LimitExceededException` | 試行回数の超過。時間をおいて再度試すよう案内する |
| トリガー関連のエラー（`UserLambdaValidationException`・`UnexpectedLambdaException` などの想定。5.4） | トリガーの失敗（下記）。パスワードが更新されたかを断定しない表示にする |

- 再設定コードの有効期限は 1 時間の想定（5.4 で要確認）。Cognito は期限をクライアントに返さないため、**期限切れが分かるのは `ConfirmForgotPassword` を送信したときだけ**で、画面は残り時間を表示できず、期限切れのコードでも新しいパスワードの入力自体は受け付けてしまう。
- 他端末の失効: `ConfirmForgotPassword` は既存の Refresh Token を失効させない前提で設計する（5.4）。`REQ-AUTH-008` #12（再設定コード経由の更新が完了したら、更新前に成立していたすべての認証状態を失効させる）を満たすため、Post confirmation トリガーで `AdminUserGlobalSignOut` を呼ぶ。サーバー側で実行されるため、ブラウザが途中で閉じられても失効は漏れない。登録の確認（`ConfirmSignUp`）でも同じトリガーが起動するが、`triggerSource` で区別して何もしない。
- 制約: `AdminUserGlobalSignOut` で失効するのは Refresh Token（と Cognito の API に対する Access Token）であり、API Gateway の JWT オーソライザーは失効を確認しない。他端末が持つ ID / Access Token は、有効期限（最大 1 時間）が切れるまで API を呼べる。他端末はその後のトークン更新に失敗し、ログイン画面へ遷移する（5.1 の制約と同じ）。

**パスワード再設定でトリガーが失敗したとき**

- トリガー用の関数は、`AdminUserGlobalSignOut` が失敗したらエラーを返し、失敗を握りつぶさない。握りつぶすと画面には成功と表示され、ユーザーは他端末が失効していないことに気づけないため。`AdminUserGlobalSignOut` は繰り返し呼んでも結果が変わらないため、再試行されても問題はない。
- Cognito はトリガーのエラーを `ConfirmForgotPassword` のエラー（トリガー関連の例外）として返す想定。Post confirmation は確認の後に起動するトリガーのため、パスワードは更新済みの可能性があるが、実際にどうなるかは確認できていない（5.4）。
- 画面は「パスワードが変更されたかどうかを確認できなかった」ことを伝え、新しいパスワードでのログインと、申請画面からの再申請のどちらにも進めるようにする。同じコードでの再送信は促さない（更新済みならコードは使用済みで通らない想定のため）。
- 残るリスク: パスワードは更新済みで失効だけが失敗した場合、他端末は Refresh Token の有効期限（最大 3 日）までログインしたままになる。失敗はトリガー用の関数のログに残す。MVP では自動で失効をやり直す仕組みは作らない。ユーザーはログイン後のパスワード変更（5.1）で他端末を失効させられる。

**トークンの保持と更新**

| 項目 | 方針 |
|---|---|
| 保持場所 | 3 つのトークンを `localStorage` に保持する |
| `localStorage` にする理由 | 認証状態を 3 日保つ必要があり、メモリだけではリロード・タブを閉じた時点で失われる。`sessionStorage` はタブごとに分かれ、新しいタブで毎回ログインが必要になる。HttpOnly Cookie はブラウザが Cognito を直接呼ぶ構成では使えない（Cookie を発行するサーバーがない） |
| リスクの考え方 | `localStorage` は同一オリジンの JavaScript から読めるため、XSS が起きるとトークンを持ち出される。ID / Access Token だけをメモリに置いても、より強い Refresh Token が `localStorage` に残るため効果が小さい。保持場所で守るのではなく、XSS を起こさない（外部スクリプトを読み込まない、React のエスケープに任せて HTML を直接挿入しない）こと、CSP で外部スクリプトの読み込みとトークンの送り先を制限すること（5.5）と、トークンの有効期間を短くすること（ID / Access 1 時間、Refresh 3 日）で被害を抑える |
| API 呼び出し | `Authorization: Bearer <ID トークン>` を付ける |
| 更新 | API 呼び出しの前に ID / Access Token の `exp` を判定し、期限間近なら `InitiateAuth`（`REFRESH_TOKEN_AUTH`）で更新する。API が `401` を返した場合は 1 回だけ更新して再試行する |
| 認証切れ | 更新に失敗した（Refresh Token の期限切れ・失効）場合はトークンを破棄し、ログイン画面へ遷移する |

**ログアウト**

1. `RevokeToken` で Refresh Token を失効させる（この端末の Refresh Token と、そこから発行されたトークンが対象）。
2. `localStorage` のトークンを削除し、ログイン画面へ遷移する。`RevokeToken` が通信エラーで失敗しても、端末のトークンは削除する。

**アカウント削除**

1. ブラウザが `DELETE /me` を呼ぶ。
2. API 本体の Lambda が DynamoDB（`keiba-log`）の対象ユーザーのデータをすべて削除し（`designs/database.md` の手順）、その後 `AdminDeleteUser` で Cognito のユーザーを削除する。
3. ブラウザはトークンを削除し、ログイン画面へ遷移する。

- 制約: `AdminDeleteUser` の後も、他端末が持つ ID / Access Token は、API Gateway の JWT オーソライザーが失効を確認しないため、有効期限（最大 1 時間）が切れるまで API を通過する。その間に、削除済みユーザーの `sub` をキーとする馬券記録が作られうる。業務ルール上は許容しており（`business-rules.md` 認証状態のルール 3）、MVP では対策を設けない。他端末はその後のトークン更新に失敗し、ログイン画面へ遷移する（5.1 の制約と同じ）。

### 5.3 未決事項

現時点で未決事項はない。

### 5.4 公式ドキュメントでの確認状況

本設計の作成時、Web 検索・取得の手段がなく、AWS / Next.js の公式ドキュメントを直接取得して照合できていない。メインエージェントが公式ドキュメントで確認した項目を「確認済み」に、それ以外を照合が済むまで「要確認」として扱う。

**確認済み**

| 項目 | 確認元 |
|---|---|
| Post confirmation トリガーが `ConfirmSignUp`・`AdminConfirmSignUp`・`ConfirmForgotPassword` で起動すること | [Post confirmation Lambda trigger](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-post-confirmation.html) |
| Next.js App Router は HTML にインラインのスクリプトを出力すること。nonce 方式は全ページを動的レンダリングにする必要があり、静的エクスポートでは使えないこと（5.5） | [Next.js: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy)（Static vs Dynamic Rendering with CSP） |
| nonce を使わない場合の例として `script-src 'self' 'unsafe-inline'` が示されていること。`'unsafe-eval'` は開発時のみ必要で、本番では不要なこと（5.5） | [Next.js: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) |
| Subresource Integrity（`experimental.sri`）は App Router のみ・実験的な機能で、ビルド時に決まるスクリプトのみが対象なこと（5.5） | [Next.js: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy) |
| CloudFront のレスポンスヘッダーポリシーの CSP の値は最大 1,783 文字で、引き上げを申請できること（5.5） | [CloudFront のクォータ](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html) |
| `next/font/google` はフォントを静的アセットとして同じドメインから配信し、閲覧時にブラウザから Google へリクエストを送らないこと（5.5） | [Next.js: Font Optimization](https://nextjs.org/docs/app/getting-started/fonts) |

**要確認**

| 項目 | 本設計での前提 |
|---|---|
| HTTP API の JWT オーソライザーが、ID トークンでは `aud`、Access Token では `client_id` をオーディエンスと照合すること | 照合する（ID トークンを使う） |
| HTTP API に CORS を設定すると、プリフライトに API Gateway が自動で応答し、統合側の CORS ヘッダーは無視されること | 自動で応答する |
| Cognito 標準メール機能の上限が 1 AWS アカウント 1 日 50 通で引き上げ不可であること、上限に達したときの `SignUp`・`ResendConfirmationCode`・`ForgotPassword` の応答 | ユーザーの提示内容をそのまま記載。上限到達時の応答は未確認 |
| ユーザー存在エラーの抑止が有効でも、正しいパスワードの未確認ユーザーには `UserNotConfirmedException` が返ること | 返る |
| ユーザー存在エラーの抑止が有効なときの `ResendConfirmationCode` の応答 | 存在しないメールアドレスでも成功に見える応答 |
| `SignUp` は抑止の設定に関係なく、登録済みメールアドレスで、未確認・確認済みを問わず `UsernameExistsException` を返すこと | 返す |
| 確認済みアカウントに対する `ConfirmSignUp`・`ResendConfirmationCode` の応答 | それぞれ `NotAuthorizedException`・`InvalidParameterException`（どちらも画面ではまとめて扱う。5.2） |
| 確認コードの有効期限 | 24 時間 |
| Cognito が未確認アカウントを自動で削除しないこと | 削除しない |
| ユーザー存在エラーの抑止が有効なときの `ForgotPassword` の応答（存在しないメールアドレス・未確認のアカウント）と、そのときメールが送られないこと | どちらも成功と同じ応答で、メールは送られない |
| 再設定コードの有効期限と、使用済み・再申請で古くなったコードが無効になること | 1 時間。使用済み・古いコードは無効 |
| `ConfirmForgotPassword` が、誤りで `CodeMismatchException`、期限切れで `ExpiredCodeException` を返すこと | 返す |
| `ConfirmForgotPassword` の成功で既存の Refresh Token（他端末のログイン）が失効しないこと | 失効しない。[トークン失効のドキュメント](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html)に失効する旨の記述は見当たらないが、失効しないと明記した記述も確認できていない。失効していた場合でもトリガーの `AdminUserGlobalSignOut` は無害 |
| Post confirmation トリガーの `triggerSource` の値の名前 | `ConfirmForgotPassword` による起動は `PostConfirmation_ConfirmForgotPassword`、`ConfirmSignUp` による起動は `PostConfirmation_ConfirmSignUp` |
| トリガーが失敗（エラー・タイムアウト）したときの `ConfirmForgotPassword`・`ConfirmSignUp` の応答（例外名）、パスワードの更新・アカウントの確認が済んでいるか、Cognito による再試行の有無 | `UserLambdaValidationException`・`UnexpectedLambdaException` などを返す。更新・確認が済んだかは不明として扱う（5.2） |
| Cognito がトリガーの応答を待つ時間 | 5 秒 |
| トリガーのイベントの `userName` を `AdminUserGlobalSignOut` の `Username` にそのまま渡せること | 渡せる |
| CDK で User Pool にトリガーを設定しつつ、トリガー用の関数の実行ロールの権限をこの User Pool の ARN に限定できること（User Pool と実行ロールが循環参照にならないこと） | できる（権限を付ける方法の工夫が要る可能性がある） |
| `GlobalSignOut`・`RevokeToken`・`AdminUserGlobalSignOut`・`AdminDeleteUser` 後も、発行済みの JWT は API Gateway の JWT オーソライザーを有効期限まで通過すること | 通過する（5.1・5.2 の制約） |
| クライアントシークレットなしのアプリクライアントで `RevokeToken` を呼べること | 呼べる |
| Refresh Token の有効期間を 3 日に設定できること | 設定できる |
| Cognito の Lite プランで 3.1 の機能（Post confirmation トリガーを含む）がすべて使えること、料金 | 使える。本アプリの規模では無料枠に収まる |
| Next.js の静的エクスポートで、`generateStaticParams` のない動的ルートが使えないこと。`useSearchParams` を使うページは Suspense 境界が必要なこと | 使えない / 必要 |
| OAC 経由の S3 オリジンでは、拡張子のないパスが `.html` のファイルに解決されないこと | 解決されない（CloudFront Functions で書き換える。3.5） |
| Next.js の静的エクスポートが、`trailingSlash: false`（既定）のとき各ページを「パス + `.html`」で出力すること（`/records/edit` → `records/edit.html`、`/` → `index.html`） | 出力する（3.5） |
| 静的エクスポート（`output: 'export'`）でも `next/font/google` が使え、フォントファイルが出力に含まれて自オリジンから配信されること | 使える（5.5 の `font-src 'self'`） |
| Go の Lambda で `provided.al2023` と arm64 を使えること | 使える |
| Next.js の静的エクスポートの出力が、インラインのスタイル（`<style>` 要素・`style` 属性）を含み、`style-src` に `'unsafe-inline'` が必要なこと | 必要（5.5） |
| Next.js の静的エクスポートで、画面遷移時の取得（RSC ペイロードなど）が同じオリジンへの通信に限られること | 限られる（`connect-src 'self'` で足りる。5.5） |
| ブラウザから Cognito を呼ぶクライアントライブラリが、`eval` 等の動的なコード実行を使わず、通信先が `cognito-idp` のエンドポイントのみであること | 使わない / のみ（5.5） |
| 画面のフォームを JavaScript で処理する限り、`form-action 'none'` で動作が妨げられないこと | 妨げられない（5.5） |
| CDK でデプロイ時に決まる API Gateway のエンドポイントを、レスポンスヘッダーポリシーの CSP の値に埋め込めること | 埋め込める（5.5） |

5.1 は、以前承認された版と照合済み。

### 5.5 CSP（Content Security Policy）

CloudFront のレスポンスヘッダーポリシーで、すべてのレスポンスに次の CSP を付ける。ポリシーは全ページ共通の 1 つ。`<region>`・`<api-id>` は CDK が環境ごとにデプロイ時の値を埋め込む（5.4 で要確認）。値の長さは CloudFront の上限（1,783 文字）に対して十分に短い。

```
default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self' https://cognito-idp.<region>.amazonaws.com https://<api-id>.execute-api.<region>.amazonaws.com; frame-ancestors 'none'; base-uri 'none'; form-action 'none'; object-src 'none'
```

| ディレクティブ | 値 | 理由 |
|---|---|---|
| `default-src` | `'self'` | 個別に指定しない種類は同じオリジンに限る |
| `script-src` | `'self' 'unsafe-inline'` | 下記「インラインスクリプトの扱い」。`'unsafe-eval'` は本番では不要なため含めない |
| `style-src` | `'self' 'unsafe-inline'` | Tailwind CSS のビルド済み CSS は同じオリジンから読む。Next.js が出力するインラインのスタイルを許可する（5.4 で要確認）。スタイルの注入ではスクリプトは実行されない |
| `img-src` | `'self' data:` | 画像は同じオリジンと、HTML・CSS に埋め込まれた `data:` のみ。外部への画像リクエストによる持ち出しを防ぐ |
| `font-src` | `'self'` | フォント（Noto Sans JP）は `next/font/google` でセルフホストする（`designs/ui-overview.md` 4 節）。フォントファイルはビルド時に取得して静的アセットとして同じオリジンから配信され、閲覧時にブラウザから Google へのリクエストは発生しないため、自オリジンだけで足りる（5.4） |
| `connect-src` | `'self'`、Cognito User Pool の API、API Gateway のデフォルトエンドポイント | `'self'` は静的エクスポートの画面遷移時の取得のため（5.4 で要確認）。それ以外の `fetch`・XHR の送り先を禁止し、トークンの持ち出し先を制限する |
| `frame-ancestors` | `'none'` | 他サイトへの埋め込みを禁止する（クリックジャッキング対策） |
| `base-uri` | `'none'` | `<base>` の注入による相対 URL の乗っ取りを防ぐ。アプリは `<base>` を使わない |
| `form-action` | `'none'` | フォームは JavaScript で処理し、ブラウザ標準の送信は使わない。JavaScript の処理が始まる前に送信されたとき、パスワードがクエリ文字列として URL に載ることと、フォームによる外部への送信を防ぐ（5.4 で要確認） |
| `object-src` | `'none'` | プラグイン要素を使わない |

**開発環境の扱い**

- AWS 上の開発環境は本番と同じ CSP を使い、`<region>`・`<api-id>` を開発環境の値にする。CSP の違反による不具合はデプロイ後の開発環境で確認する。
- ローカルの開発サーバー（`next dev`、`http://localhost:3000`）は CloudFront を通らないため、この CSP は付かない。開発サーバーが必要とする `'unsafe-eval'` と `localhost` は、デプロイする CSP に含めない。

**インラインスクリプトの扱い**

Next.js App Router は各ページの HTML にインラインのスクリプトを出力するため、`script-src 'self'` だけでは画面が動かない。

| 案 | 判断 |
|---|---|
| `'unsafe-inline'` を許可する | **採用**。静的エクスポート + S3 + CloudFront の構成のまま動き、CSP を全ページ共通の 1 つにできる。Next.js のドキュメントが nonce を使わない場合の例として示す形 |
| nonce | 不採用。nonce を使うには全ページを動的レンダリングにする必要があり、静的エクスポートでは使えない。画面をサーバー（Lambda など）で生成する構成に変えることになり、静的配信・サーバーレスの方針から外れる |
| インラインスクリプトのハッシュをヘッダーに並べる | 不採用。ハッシュはページごと・ビルドごとに変わるが、レスポンスヘッダーポリシーは全ページ共通で、値は最大 1,783 文字。全ページ分を並べると上限を超えるおそれが高く、ビルドのたびにポリシーを更新する仕組みも要る |
| ビルド後に各 HTML へハッシュ入りの `<meta>` の CSP を書き込む | 不採用。Next.js の出力を後処理する独自の仕組みになり、Next.js の更新で壊れやすい。`<meta>` の CSP では `frame-ancestors` が効かず、ヘッダーとの二重管理になる |
| Subresource Integrity（`experimental.sri`） | 不採用。スクリプトファイルに integrity 属性を付けて改ざんを検知する機能で、インラインスクリプトの実行を許可する仕組みではないため、この問題の解決にならない。App Router のみ・実験的な機能でもある |

**`'unsafe-inline'` で失われる防御と残る対策**

- 失われる防御: HTML の注入が起きた場合、注入された `<script>` 要素・イベントハンドラー属性（`onerror` など）・`javascript:` URL が実行される。CSP は XSS の実行そのものを止めない。
- 残る対策:
  - XSS を作らない: React のエスケープに任せ、HTML を直接挿入しない（`dangerouslySetInnerHTML` などを使わない）。
  - 外部のスクリプトを読み込まない: `script-src` に外部のオリジンを含めないため、外部のスクリプトファイルの読み込みは CSP で止まる。
  - トークンの持ち出し先を制限する: `connect-src` を自オリジン・Cognito・API に限り、`img-src`・`form-action` も外部に開かないため、`fetch`・XHR・画像・フォームでは外部へ送れない。ただし、ページの遷移（`location` の書き換え）や新しいウィンドウを開く操作による持ち出しは CSP では止められない。
  - 被害の期間を限る: トークンの有効期間を短くする（ID / Access Token 1 時間、Refresh Token 3 日）。

## 6. 非機能要件への対応

| 観点 | 対応方針 |
|---|---|
| Security | 認証は Cognito User Pool に集約し、ブラウザが SRP で認証してトークンを保持する。API は全ルートを JWT オーソライザーで保護し、CORS で許可するオリジンを限定する。データ分離はオーソライザーが検証した `sub` をキーに含めたアクセス制御で担保する。トークンを狙う XSS は、HTML を直接挿入しない・外部スクリプトを読み込まない方針で防ぎ、CSP で外部スクリプトの読み込みとトークンの送り先を制限する（5.5）。 |
| Data Integrity | 親子関係を持つデータは同一パーティションに配置し、複数アイテムをまたぐ更新は DynamoDB のトランザクション書き込みで原子的に行う。中途半端な状態が残らないようにする。 |
| Availability / Recovery | マネージドサービスのみで構成し、可用性はサービス側の冗長化に委ねる。`keiba-log` テーブルは PITR を有効にして任意時点への復元手段を確保する。 |
| Compatibility | Next.js + Tailwind CSS でレスポンシブに実装し、スマートフォンと PC の双方の画面サイズに単一の Web アプリで対応する。認証の画面もアプリ自身が持つため、同じデザインで両方に対応する。 |

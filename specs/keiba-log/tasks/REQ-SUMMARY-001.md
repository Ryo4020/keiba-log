# 実装タスク: REQ-SUMMARY-001 生涯収支サマリー表示

| 項目 | 内容 |
|---|---|
| 要件ID | `REQ-SUMMARY-001` |
| 担当 | task-planner |
| 受け入れ基準 | `requirements/user-stories/REQ-SUMMARY-001-lifetime-summary.md` を参照 |

## ステータス凡例
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

## タスク一覧

### `[ ]` バックエンド
- **担当**: backend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-SUMMARY-001-lifetime-summary.md`, `specs/keiba-log/requirements/business-rules.md`（「収支額・回収率の算出ルール」の「回収率の表記の精度」「利益・損失・均衡の判定」を含む）, `specs/keiba-log/designs/api.yml`（`/summary` の `get`、`components.schemas.Summary`）, `specs/keiba-log/designs/api.md`（1節〜3節）, `specs/keiba-log/designs/architecture.md`（3.4「ログ」・5節「データ分離」）, `specs/keiba-log/designs/database.md`（1.4・1.6・2節 AP-01、4節「読み取り整合性」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（PK/SK・「生涯収支集計アイテムの属性」・「関連するアクセスパターン」の AP-01）
- **概要**:
  - `GET /summary`（`getSummary`）のエンドポイント
    - 対象ユーザーは JWT オーソライザーが検証したクレームの `sub` のみで特定する
    - 生涯収支集計アイテムの取得（AP-01。`GetItem`、強い整合性のある読み取り）
    - 集計アイテムが存在しない場合は、購入額合計・払戻額合計・記録件数を0として扱う
    - 収支額（`balance` = 払戻額合計 − 購入額合計）と回収率（`recoveryRate` = 払戻額合計 ÷ 購入額合計 × 100）を取得時に算出する。購入額合計が0のときは `recoveryRate` を `null` にする
    - `recoveryRate` は小数第2位を四捨五入した小数第1位までの値にして返す（例 85.26 → 85.3、100% ちょうど → 100.0、0% → 0.0）。丸めは最終の値のみで、算出の途中（`R ÷ P` の商など）では丸めない。`balance` は丸めずに整数のまま返す
    - 期間の指定・絞り込みのパラメータは受け付けない
    - 応答: 成功は `200`（`Summary`）、想定外エラーは `500`（`api.md` 3節の共通エラーレスポンス）
- **受け入れ**: 対応する `requirements/user-stories/REQ-SUMMARY-001-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` フロントエンド
- **担当**: frontend-developer
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-SUMMARY-001-lifetime-summary.md`, `specs/keiba-log/requirements/business-rules.md`（「収支額・回収率の算出ルール」の「回収率の表記の精度」「利益・損失・均衡の判定」）, `specs/keiba-log/designs/ui-designs/home.md`（1節・2節・3.1「遷移・開閉」・3.3「データの状態による表示の分岐」・3.4「読み込み中とエラー」・3.5「共通の操作性」・4節の共通パーツ「グローバルサイドバー（アカウントメニューを含む）」「収支表示」「空状態表示」・5節、Figma ノード `17-3`・`18-4`・`19-2`・`19-27`・`19-104`・`21-2`・`23-2`・`23-3`・`23-4`・`23-10`）, `specs/keiba-log/designs/ui-overview.md`（1節の画面の URL・3節〜5節・6節 共通パーツ一覧）, `specs/keiba-log/designs/api.yml`（`/summary` の `get`、`components.schemas.Summary`）, `specs/keiba-log/designs/api.md`（2節「認証は Cognito User Pool に委譲する」）, `specs/keiba-log/designs/architecture.md`（5.2「トークンの保持と更新」）
- **概要**:
  - ホーム画面（`/`）の枠
    - 画面タイトル行（画面タイトル「ホーム」と、`md` 未満でのサイドバー開閉トリガー。ヘッダーの帯は設けない）
  - 共通パーツ「グローバルサイドバー」（オーナー: ホーム画面）
    - サービス名、ナビ項目「ホーム」（現在位置のハイライト。押下しても遷移せず、ドロワー表示時は閉じるだけ）・「馬券記録一覧」（`/records` へ遷移。ドロワー表示時は閉じてから遷移）
    - `md` 未満はオーバーレイを伴うドロワー（開閉トリガーで開き、オーバーレイ押下・閉じるボタン・Esc・項目の選択で閉じる）、`md` 以上は画面左に常時表示
    - アカウントのアイコンボタンと、そこから開くアカウントメニューの枠（メニュー外の押下・Esc で閉じる。ドロワーを閉じたときは同時に閉じる）
    - アカウントメニューの「ログイン中のメールアドレス」の表示項目（ID トークンから取得する。押下不可で、フォーカスを受け取らない）
    - ドロワー・アカウントメニューを開いている間は背後の要素にフォーカスを移さず、閉じたら開く操作を行った要素にフォーカスを戻す
  - 共通パーツ「収支表示」（オーナー: ホーム画面）
    - 収支額を符号付きで表示し、0は均衡の表記にする。プラス・マイナス・ゼロでカラートークン（`color-profit` / `color-loss` / `color-even`）を使い分ける
    - 主数値として使う場面では、利益／損失／均衡の状態チップを併用する（色だけで正負を伝えない）
    - 状態チップ・色・符号は `balance` の正負で決める（正は利益、負は損失、0は均衡）。回収率の表示値では決めない（回収率が 100.0% と表示されていても、`balance` が負なら損失、正なら利益の表記にする）
  - 共通パーツ「空状態表示」（オーナー: ホーム画面）
    - 見出し・説明・アクションスロットの構成。ホーム画面ではアクションスロットを使わない
  - 収支サマリー領域
    - `GET /summary` を呼び、収支額（`balance`）を主数値、回収率（`recoveryRate`）を副数値として表示する
    - 補助情報として購入額合計（`totalPurchaseAmount`）・払戻額合計（`totalPayoutAmount`）・記録件数（`recordCount`）をラベル付きで表示する
    - 回収率は API が返す小数第1位までの値を、受け取った値のまま小数第1位まで表示する（例 85.3%、100.0%、0.0%。末尾の 0 も出す）。整数に丸めない・再丸めや再計算をしない
    - `recoveryRate` が `null` のときは、値の代わりにプレースホルダーと「購入額合計が0円のため回収率を算出できません」の注釈を表示する
    - `recordCount` が0のときは、領域を空状態表示に差し替える
    - ホーム画面を表示するたびに `GET /summary` を呼び、前回の取得結果を使い回さない
    - 読み込み中は本領域だけをプレースホルダーに差し替える。`500` のときは本領域をエラー表示と「再読み込み」ボタン（`GET /summary` のみを再取得）に差し替える。`401` のときはログイン画面へ遷移する
    - 集計値を画面で再計算しない。集計期間の指定・絞り込みの操作を置かない
  - アカウントメニューの「パスワード変更」「ログアウト」「アカウント削除」の各項目の動作、直近記録リスト領域、新規登録の導線は本タスクの対象外とする
- **受け入れ**: 対応する `requirements/user-stories/REQ-SUMMARY-001-*.md` の受け入れ基準をすべて満たしていること

### `[ ]` テスト
- **担当**: unit-test-generator
- **参照仕様**: `specs/keiba-log/requirements/user-stories/REQ-SUMMARY-001-lifetime-summary.md`, `specs/keiba-log/requirements/business-rules.md`（「収支額・回収率の算出ルール」の「回収率の表記の精度」「利益・損失・均衡の判定」を含む）, `specs/keiba-log/designs/api.yml`（`/summary` の `get`、`components.schemas.Summary`）, `specs/keiba-log/designs/api.md`（2節・3節）, `specs/keiba-log/designs/database.md`（1.4・2節 AP-01、4節「読み取り整合性」）, `specs/keiba-log/designs/db-tables/keiba-log.md`（「生涯収支集計アイテムの属性」・「関連するアクセスパターン」の AP-01）
- **概要**:
  - `GET /summary` のバックエンド処理の Go ユニットテスト
    - 収支額・回収率の算出（利益・損失・均衡、払戻額合計が0で購入額合計が0より大きい場合の回収率0％）
    - 購入額合計が0のときに `recoveryRate` が `null` になること
    - 回収率の丸め（小数第2位を四捨五入して小数第1位まで）
      - 四捨五入の境界: 85.25 → 85.3（切り上げ側）、85.249… → 85.2（切り捨て側）
      - 100% ちょうど → 100.0、払戻額合計が0で購入額合計が0より大きい場合 → 0.0
      - 99.95 以上 100 未満 → 100.0 になること（100 超 100.05 未満も 100.0 になること）
      - 丸めは最終の値のみで、途中の商を丸めていないこと
    - `balance` の正負は丸めの影響を受けないこと（回収率が 100.0 に丸められる場合でも、`balance` は払戻額合計 − 購入額合計のままで、99.95 以上 100 未満なら負、100 超 100.05 未満なら正になる）
    - 集計アイテムが存在しない場合に、各合計・記録件数が0、`recoveryRate` が `null` で返ること
    - 取得対象がクレームの `sub` のユーザーの集計アイテムに限られ、強い整合性のある読み取りで取得されること
    - 成功時の `200`、DynamoDB の失敗時の `500` の応答
- **受け入れ**: 対応する `requirements/user-stories/REQ-SUMMARY-001-*.md` の受け入れ基準をすべて満たしていること

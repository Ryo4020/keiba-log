# テーブル定義: keiba-log

| 項目 | 内容 |
|---|---|
| 機能名 | `keiba-log` |
| 担当 | db-designer |
| 対象テーブル | `keiba-log` |
| インプット | `designs/database.md`（対象テーブル分のみ） |

## 概要

本アプリの単一テーブル。ユーザー識別子（Cognito の `sub`）を分割の単位（パーティションキー）とし、同一パーティションに次の2種類のアイテムを格納する。

- **生涯収支集計アイテム**: ユーザーごとに1件
- **馬券記録アイテム**: レース記録・簡易記録の両方をこのアイテム種類で表す。買い目・買い目の集合はネストして保持する

アイテム種類はソートキーのプレフィックスで区別する。

## パーティションキー（PK）・ソートキー（SK）

| キー | 属性名 | 型 | 値の構成 |
|---|---|---|---|
| パーティションキー | `PK` | S | `USER#<sub>`（`<sub>` は Cognito の `sub`） |
| ソートキー | `SK` | S | 生涯収支集計アイテム: `SUMMARY` 固定<br>馬券記録アイテム: `RECORD#<date>#<createdAt>#<recordId>` |

### ソートキーの構成要素（馬券記録アイテム）

| 要素 | 型 | 内容 |
|---|---|---|
| `date` | S | 馬券記録の「日付」属性の値そのもの（`YYYY-MM-DD`）。ゼロ埋め済みの形式なので辞書順が日付順と一致する |
| `createdAt` | S | 登録日時（ISO 8601、ミリ秒精度、UTC、`YYYY-MM-DDTHH:mm:ss.sssZ`）。同一日付内の順序を決める |
| `recordId` | S | 馬券記録の一意な識別子（UUID）。同一ミリ秒での登録が重複した場合の最終的なタイブレーカー |

### 設計の要点

- **アイテム種類の区別**: `SK` が `SUMMARY` かどうか、または `RECORD#` で始まるかどうかで区別する。馬券記録アイテムはさらに属性 `recordType`（`RACE` / `SIMPLE`）でレース記録・簡易記録を区別する。
- **ユーザー識別子が分割の単位**: `PK` に `sub` を含めるため、全アクセスパターンが単一パーティションに閉じる（`database.md` 1.2・1.6 に対応）。
- **並び順の自然な表現**: `date` → `createdAt` → `recordId` の順で連結した文字列は、`date` 昇順・同一 `date` 内は `createdAt` 昇順という辞書順になる。一覧・直近記録リストの取得（AP-02・AP-03）は `SK` の `begins_with(RECORD#)` で Query し、`ScanIndexForward=false`（降順）を指定することで「日付の新しい順、同日付は登録日時の新しい順」がそのまま得られる。逆順に並べ替えるための特別なエンコード（値の反転など）は不要。
- **馬券記録1件への到達（AP-04・AP-06・AP-07）**: `SK` に `date` と `createdAt` を含むため、記録1件を一意に特定するには `date`・`createdAt`・`recordId` の3要素（＝ `SK` の `RECORD#` 以降の文字列）が必要になる。この3要素は一覧取得（AP-02・AP-03）のレスポンスに含まれる属性からそのまま得られるため、編集・削除の対象を指定する際はこの3要素（または `SK` そのもの）を記録の識別情報として渡す構成にする。`recordId` 単独では `GetItem`・`UpdateItem`・`DeleteItem` の対象にできない点に注意（GSI を追加しない設計を選んだトレードオフ。後述）。
- **日付の変更を伴う更新（AP-06）**: 「日付」は編集可能な属性だが `SK` の一部でもあるため、日付を変更する更新は同一 `SK` への `UpdateItem` では行えない。この場合は「旧 `SK` のアイテムを `Delete`」+「新しい `SK` でアイテムを `Put`」+「集計アイテムの寄与額差分更新」の3アクションを1つの `TransactWriteItems` に含めて原子的に行う。日付を変更しない更新は「対象アイテムの `UpdateItem`」+「集計アイテムの差分更新」の2アクションで済む。

## 属性一覧

### 生涯収支集計アイテムの属性

| 属性名 | 型 | 説明 |
|---|---|---|
| `PK` | S | `USER#<sub>` |
| `SK` | S | `SUMMARY` 固定 |
| `itemType` | S | `SUMMARY` 固定。アイテム種類の識別用 |
| `totalPurchaseAmount` | N | 購入額合計（`P`）。馬券記録の寄与額の加算・減算で差分更新する |
| `totalPayoutAmount` | N | 払戻額合計（`R`）。同上 |
| `recordCount` | N | 馬券記録の件数。空状態の判定に使う |

収支額・回収率は保存せず、`totalPurchaseAmount`・`totalPayoutAmount` から読み取り時に算出する（`database.md` 1.4）。

### 馬券記録アイテムの共通属性（レース記録・簡易記録に共通）

| 属性名 | 型 | 説明 |
|---|---|---|
| `PK` | S | `USER#<sub>` |
| `SK` | S | `RECORD#<date>#<createdAt>#<recordId>` |
| `itemType` | S | `RECORD` 固定。アイテム種類の識別用 |
| `recordId` | S | 馬券記録の識別子（UUID） |
| `recordType` | S | `RACE`（レース記録）／`SIMPLE`（簡易記録）。登録後に変更しない |
| `date` | S | 日付（`YYYY-MM-DD`） |
| `venue` | S | 競馬場 |
| `createdAt` | S | 登録日時（ISO 8601、ミリ秒精度） |
| `purchaseAmount` | N | 購入額。レース記録は買い目の購入額の自動算出合計、簡易記録はユーザー入力値 |
| `summaryContributionPurchase` | N | この記録が生涯収支集計の `totalPurchaseAmount` に寄与する額 |
| `summaryContributionPayout` | N | この記録が生涯収支集計の `totalPayoutAmount` に寄与する額 |

共通属性を追加・改名した場合は、下記「AP-02 の射影属性」の一覧も見直す（射影はホワイトリスト方式のため、列挙漏れは直近記録リストでの値の欠落になる）。

`summaryContributionPurchase` / `summaryContributionPayout` の算出は `business-rules.md` の「収支額・回収率の算出ルール」に従う。レース記録は結果状態が「返還」「結果未確定」の買い目を除いた購入額・払戻額の合計、簡易記録は `purchaseAmount`／`payoutAmount` をそのまま用いる。

### レース記録固有の属性

| 属性名 | 型 | 説明 |
|---|---|---|
| `raceNumber` | N | レース番号 |
| `confirmationStatus` | S | 結果の確定状態。`PENDING`（確定前）／`CONFIRMED`（確定後） |
| `bets` | L（各要素は M） | 通常の買い目のリスト（構造は下記「買い目」を参照） |
| `betGroups` | L（各要素は M） | 買い目の集合のリスト（構造は下記「買い目の集合」を参照） |

#### 買い目（`bets` および `betGroups[].bets` の要素、M の内部構造）

| キー | 型 | 説明 |
|---|---|---|
| `betId` | S | 買い目の識別子（UUID） |
| `ticketType` | S | 券種。`WIN`（単勝）／`PLACE`（複勝）／`BRACKET_QUINELLA`（枠連）／`QUINELLA`（馬連）／`WIDE`（ワイド）／`EXACTA`（馬単）／`TRIO`（三連複）／`TRIFECTA`（三連単） |
| `combination` | L（各要素は N） | 組合せ（対象の番号のリスト）。要素数は券種の選択数（単勝・複勝は1、二連系は2、三連系は3）に対応。着順を区別する券種（馬単・三連単）は配列の順序が着順を表す |
| `amount` | N | 購入額。通常の買い目はユーザーが買い目ごとに入力した値、買い目の集合に含まれる買い目は所属する集合の `unitAmount` と一致する値 |
| `resultStatus` | S | 結果状態。`UNCONFIRMED`（結果未確定）／`WON`（的中）／`LOST`（不的中）／`REFUNDED`（返還） |
| `payoutAmount` | N（結果状態が `WON` のときのみ値を持つ） | 払戻額 |

#### 買い目の集合（`betGroups` の要素、M の内部構造）

| キー | 型 | 説明 |
|---|---|---|
| `betGroupId` | S | 買い目の集合の識別子（UUID） |
| `ticketType` | S | 券種。値の範囲は買い目の `ticketType` と同じ。登録後に変更しない |
| `method` | S | 買い方。`BOX`（ボックス）／`NAGASHI`（ながし）／`FORMATION`（フォーメーション）。登録後に変更しない |
| `methodSpec` | M | 買い方の指定内容。構造は `method` に対応し、以下のキーのうち該当するものを持つ |
| `unitAmount` | N | 1点あたりの購入額 |
| `betCount` | N | 点数（自動算出）。当該集合に含まれる買い目の件数 |
| `amount` | N | 購入額（自動算出）。`unitAmount × betCount` |
| `bets` | L（各要素は M） | 当該集合に含まれる買い目のリスト（構造は上記「買い目」と同一） |

`methodSpec`（M）が持つキーは `method` の値によって次のように異なる。

| `method` | `methodSpec` のキー | 型 | 説明 |
|---|---|---|---|
| `BOX` | `selections` | L（各要素は N） | 選択した対象（馬番または枠番）のリスト |
| `NAGASHI` | `axis` | L（各要素は N） | 軸として選んだ対象（1つまたは2つ） |
| `NAGASHI` | `partners` | L（各要素は N） | 相手として選んだ対象のリスト |
| `NAGASHI` | `axisPosition` | S（値を持つ場合のみ） | 軸の着順固定の指定（馬単・三連単の着順指定ながし）。`1着`／`2着`／`3着` に対応する値、またはマルチの場合は値を持たない |
| `NAGASHI` | `multi` | BOOL | マルチ（軸が取りうるすべての着順を網羅する指定）かどうか |
| `FORMATION` | `positions` | L（各要素は L〈各要素は N〉） | 着順（構成位置）ごとの選択集合のリスト。配列の順序が着順（構成位置）に対応する |

### 簡易記録固有の属性

| 属性名 | 型 | 説明 |
|---|---|---|
| `payoutAmount` | N | 払戻額（ユーザー入力） |

簡易記録は券種・買い方・組合せ・レース番号・結果状態・結果の確定状態を持たない（`business-rules.md` の「簡易記録」参照）。

## GSI

**定義しない。**

`database.md` のアクセスパターン一覧（AP-01〜AP-09）はいずれも `PK`・`SK` のみで実現できる（下記「関連するアクセスパターン」参照）。一覧・直近記録リストの並び順は `SK` の構成（`date`→`createdAt`→`recordId`）と `ScanIndexForward` の指定で表現でき、点参照（AP-04・AP-06・AP-07）は一覧から得られる `date`・`createdAt`・`recordId` を用いて `SK` を直接組み立てられるため、`recordId` から逆引きするための GSI は不要と判断した。不要な GSI を増やさない方針（`database.md`）に従う。

## LSI

**定義しない。** `SK` 単独で AP-01〜AP-09 すべてに対応できるため、別のソート順を提供する LSI は不要。

## 関連するアクセスパターン

| # | アクセスパターン | 実現方法 |
|---|---|---|
| AP-01 | 生涯収支サマリーの取得 | `GetItem`（強い整合性のある読み取り）。`PK = USER#<sub>`, `SK = SUMMARY` |
| AP-02 | 馬券記録概要リストの取得（呼び出し側が指定した件数を先頭から、日付の新しい順） | `Query`。`PK = USER#<sub>`, `SK begins_with "RECORD#"`, `ScanIndexForward = false`, `Limit` に呼び出し側が指定した件数（1〜20件、デフォルト5件）を渡す、結果整合の読み取り。ページングは行わない。`ProjectionExpression` で下記「AP-02 の射影属性」に列挙した属性のみを取得し、`bets` / `betGroups` は取得しない |
| AP-03 | 馬券記録一覧の取得（20件ずつ、カーソルページング） | `Query`。`PK = USER#<sub>`, `SK begins_with "RECORD#"`, `ScanIndexForward = false`, `Limit = 20`、`ExclusiveStartKey` にカーソルを指定、結果整合の読み取り。`ProjectionExpression` は指定せず、`bets` / `betGroups` を含む全属性を取得する（AP-02 との違い）。Query の 1MB 上限により1ページが20件未満で返ることは許容し、継続読み取りで20件に揃えない |
| AP-04 | 馬券記録1件の取得（編集画面向け） | `GetItem`（強い整合性のある読み取り）。`PK = USER#<sub>`, `SK = RECORD#<date>#<createdAt>#<recordId>`（一覧等から得た値で組み立てる） |
| AP-05 | 馬券記録の登録 | `TransactWriteItems`：① 新しい `SK`（現在時刻の `createdAt` と新規 `recordId` を採番）で馬券記録アイテムを `Put`。② 集計アイテムに対し `UpdateItem`（`ADD totalPurchaseAmount`, `ADD totalPayoutAmount`, `ADD recordCount 1`）。集計アイテムが未作成でも `ADD` により作成される |
| AP-06 | 馬券記録の更新 | 日付を変更しない場合: `TransactWriteItems`：① 対象アイテムを `UpdateItem`（`PK`/`SK` は不変、属性を `SET`）で更新。② 集計アイテムに旧寄与額の減算と新寄与額の加算を `ADD` で反映。<br>日付を変更する場合: `TransactWriteItems`：① 旧 `SK` のアイテムを `Delete`（`ConditionExpression: attribute_exists(PK)`）。② 新しい `SK`（新しい `date`、`createdAt`・`recordId` は既存値を維持）でアイテムを `Put`。③ 集計アイテムの差分更新を `ADD` |
| AP-07 | 馬券記録の削除 | `TransactWriteItems`：① 対象アイテムを `Delete`（`ConditionExpression: attribute_exists(PK)`）。② 集計アイテムに対し `UpdateItem`（`ADD totalPurchaseAmount -<寄与額>`, `ADD totalPayoutAmount -<寄与額>`, `ADD recordCount -1`）。ネストした買い目・買い目の集合は記録アイテムの削除で同時に消える |
| AP-08 | アカウント削除時の列挙 | `Query`。`PK = USER#<sub>` のみを指定（`SK` 条件なし）。集計アイテム・すべての馬券記録アイテムが列挙される |
| AP-09 | アカウント削除時の削除 | AP-08 で列挙した `PK`/`SK` の組を用いて `BatchWriteItem`（削除リクエスト、25件ずつ）を繰り返す。アイテム数が多い場合は複数バッチに分割し、失敗分は再実行する |

### AP-02 の射影属性

AP-02（馬券記録概要リスト）の `ProjectionExpression` で取得する属性は次の12個。**この一覧が射影の正本**であり、実装はこの通りに指定する。

| 属性名 | 用途 |
|---|---|
| `PK` | 記録の識別情報 |
| `SK` | 記録の識別情報。AP-04・AP-06・AP-07 の点参照に必要 |
| `recordId` | 記録の識別情報 |
| `createdAt` | 記録の識別情報（`SK` の構成要素を単独の値として扱うため） |
| `recordType` | 記録種別（`RACE` / `SIMPLE`）。表示の切り替えに使う |
| `date` | 日付 |
| `venue` | 競馬場 |
| `purchaseAmount` | 購入額 |
| `summaryContributionPurchase` | 集計寄与額（購入額） |
| `summaryContributionPayout` | 集計寄与額（払戻額） |
| `raceNumber` | レース番号。馬券記録概要リストの行で `{venue} 第{raceNumber}R` として表示する。レース記録のアイテムのみが持ち、簡易記録のアイテムには存在せず、その場合は返らない |
| `confirmationStatus` | レース記録の結果の確定状態。簡易記録のアイテムには存在せず、その場合は返らない |

含めない属性とその理由。

- `bets` / `betGroups`: AP-02 は買い目を表示しないため（`database.md` 1.5）。射影の目的そのもの。
- `itemType`: `SK begins_with "RECORD#"` で馬券記録アイテムに限定済みのため、識別に使わない。
- `payoutAmount`（簡易記録固有）: 簡易記録の払戻額は `summaryContributionPayout` と同値のため重複する。

注意点。

- **射影はホワイトリスト方式である。** DynamoDB に「この属性を除く」という指定はなく、列挙した属性だけが返る。したがって馬券記録アイテムの共通属性を追加したとき、この一覧への追加を忘れると直近記録リストでその値が欠ける不具合になる。属性を追加・改名する際は必ずこの一覧を見直す。
- `date` は DynamoDB の予約語のため、`ProjectionExpression` では `ExpressionAttributeNames`（例: `#date`）を介して指定する。
- **`ProjectionExpression` は消費 RCU を減らさない。** RCU は射影前のアイテムサイズで計算されるため読み取りコストは変わらない。目的はレスポンスサイズ・転送量・デシリアライズコストを買い目の件数に依存しない一定量に抑えることにある。

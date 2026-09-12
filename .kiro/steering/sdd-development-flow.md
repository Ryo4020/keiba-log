---
name: sdd-development-flow
description: 仕様駆動開発フロー。10エージェントによる要件定義→設計→タスク分解→実装→レビューの段階的実行と承認ゲート
inclusion: auto
---

# 仕様駆動開発フロー：段階的エージェント実行ガイド

カスタムエージェントを用いた仕様駆動開発の実行フローを定義する。各エージェントが自分の担当ドキュメントを作成し、**成果物ごとにユーザーの承認を受けてから次工程に進む**。

Kiro IDE組み込みのSpec機能（`.kiro/specs/`）は使用しない。仕様ドキュメントはすべて**プロジェクトルート直下の `specs/` ディレクトリ**で管理する。

---

## ディレクトリ構成

```
specs/
└── <機能名>/                  # ケバブケース（例: user-authentication）
    ├── requirements.md        # 要件定義（requirements-analyst）
    ├── architecture.md        # アーキテクチャ設計（system-architect）
    ├── database.md            # データベース設計（db-designer）
    ├── api.md                 # API設計（api-designer）
    ├── ui-design.md           # 画面設計（web-ui-designer）
    └── tasks.md               # 実装タスクリスト（task-planner）
```

機能ごとに `specs/<機能名>/` を作成する。1つの巨大な仕様にまとめず、機能単位で分割する。

---

## ファイル所有権

各ドキュメントには**唯一のオーナーエージェント**がいる。オーナー以外は読み取り専用（`tasks.md` のみ例外）。

| ファイル | オーナー（作成・更新） | 更新可能な他エージェント |
|---|---|---|
| `requirements.md` | requirements-analyst | なし |
| `architecture.md` | system-architect | なし |
| `database.md` | db-designer | なし |
| `api.md` | api-designer | なし |
| `ui-design.md` | web-ui-designer | なし |
| `tasks.md` | task-planner（作成） | 実装4エージェント（進捗更新）、code-reviewer（タスク追加） |

この所有権分離により、複数エージェントが同一ファイルを編集する競合が発生しない。

---

## フロー概要

```
Phase 1: 要件定義
  └─ requirements-analyst → requirements.md → [承認] →

Phase 2: 設計（逐次実行）
  ├─ 2.1 system-architect  → architecture.md → [承認] →
  ├─ 2.2 db-designer       → database.md    → [承認] →
  ├─ 2.3 api-designer      → api.md         → [承認] →
  └─ 2.4 web-ui-designer   → ui-design.md   → [承認] →

Phase 3: タスク分解
  └─ task-planner → tasks.md → [承認] →

Phase 4: 実装（タスク単位、依存なしタスクは並列可）
  ├─ backend-developer     → [タスクごとに承認]
  ├─ frontend-developer    → [タスクごとに承認]
  ├─ aws-developer         → [タスクごとに承認]
  └─ unit-test-generator   → [タスクごとに承認]

Phase 5: レビュー
  └─ code-reviewer → チャットで報告 → [ユーザー判断] → 修正必要ならtasks.mdに追加してPhase 4へ
```

---

## Phase 1: 要件定義

### 成果物
`specs/<機能名>/requirements.md`（担当: requirements-analyst）

### 実行手順
1. ユーザーが機能要求を提示
2. requirements-analystを起動:
   ```
   invoke requirements-analyst with "以下の機能要件を specs/<機能名>/requirements.md に整理してください: [ユーザー要求]"
   ```
3. エージェントがヒアリングを実施し、要件を整理。各要件に `REQ-<領域>-<連番>` 形式のIDを付与
4. **【承認ゲート】** ユーザーが確認:
   - ✅ 承認 → Phase 2へ
   - ❌ 修正要求 → requirements-analystに追加指示して再生成

### 確認ポイント
- 機能要件・非機能要件が網羅されているか
- エッジケース・エラー処理が考慮されているか
- すべての要件に要件IDが付与されているか
- 曖昧な表現がないか

---

## Phase 2: 設計

設計は4エージェントを**逐次実行**する。各エージェントは前段の設計を前提に自分の担当領域を設計するため、順序を守る。

### 2.1 アーキテクチャ設計

**成果物**: `specs/<機能名>/architecture.md` + Miroのアーキテクチャ図

1. 起動:
   ```
   invoke system-architect with "specs/<機能名>/requirements.md を基にアーキテクチャを設計し、specs/<機能名>/architecture.md を作成してください。Miroにアーキテクチャ図を作成してください。"
   ```
2. エージェントの作業:
   - `requirements.md` を読み込み
   - `architecture.md` にコンポーネント構成・技術スタック・データフロー・インフラ構成・セキュリティ設計・ディレクトリ構造を記述
   - Miroボードにアーキテクチャ図・シーケンス図を作成
3. **【承認ゲート】** 確認ポイント: 技術選定の妥当性、コンポーネント分割、Miroの図の正確性

### 2.2 データベース設計

**成果物**: `specs/<機能名>/database.md` + MiroのER図

1. 起動:
   ```
   invoke db-designer with "requirements.md と architecture.md を基にDynamoDBのデータモデルを設計し、specs/<機能名>/database.md を作成してください。MiroにER図を作成してください。"
   ```
2. エージェントの作業:
   - `requirements.md`・`architecture.md` を読み込み
   - `database.md` にアクセスパターン一覧・テーブル定義（PK/SK/GSI）・アイテム例・キャパシティ設計を記述
   - MiroボードにER図を作成
3. **【承認ゲート】** 確認ポイント: アクセスパターンの網羅性、キー設計、GSIの必要性

### 2.3 API設計

**成果物**: `specs/<機能名>/api.md`

1. 起動:
   ```
   invoke api-designer with "requirements.md・architecture.md・database.md を基にREST API仕様を設計し、specs/<機能名>/api.md を作成してください。"
   ```
2. エージェントの作業:
   - `requirements.md`・`architecture.md`・`database.md` を読み込み
   - `api.md` にエンドポイント一覧・リクエスト/レスポンススキーマ・エラーレスポンス・OpenAPI仕様を記述
3. **【承認ゲート】** 確認ポイント: エンドポイント設計、スキーマ、エラーハンドリング戦略

### 2.4 画面設計

**成果物**: `specs/<機能名>/ui-design.md` + Figmaのデザイン

1. 起動:
   ```
   invoke web-ui-designer with "requirements.md・architecture.md・api.md を基にUI/UX設計を行い、specs/<機能名>/ui-design.md を作成してください。Figmaでワイヤーフレームとデザインを作成してください。"
   ```
2. エージェントの作業:
   - `requirements.md`・`architecture.md`・`api.md` を読み込み
   - `ui-design.md` に画面一覧・画面遷移・コンポーネント設計・デザイントークン・画面とAPIの対応を記述
   - Figmaにワイヤーフレームとビジュアルデザインを作成
3. **【承認ゲート】** 確認ポイント: 画面フロー、UIコンポーネント設計、Figmaデザイン

---

## Phase 3: タスク分解

### 成果物
`specs/<機能名>/tasks.md`（担当: task-planner）

### 実行手順
1. 起動:
   ```
   invoke task-planner with "specs/<機能名>/ 配下の全設計ドキュメントを基に実装タスクを specs/<機能名>/tasks.md に分解してください。"
   ```
2. エージェントの作業:
   - `requirements.md`・`architecture.md`・`database.md`・`api.md`・`ui-design.md` を読み込み
   - Wave（並列実行可能な単位）ごとにタスクを整理
   - 各タスクに担当エージェント・依存関係・参照仕様・対応要件ID・受け入れ条件を記載
3. **【承認ゲート】** 確認ポイント:
   - タスクの粒度（1エージェントが1セッションで完了できるサイズか）
   - 依存関係とWave分割の妥当性
   - 全要件がいずれかのタスクでカバーされているか

### tasks.md のステータス管理
| 記号 | 意味 |
|---|---|
| `[ ]` | 未着手 |
| `[/]` | 実装中 |
| `[x]` | 完了 |

---

## Phase 4: 実装

### 担当エージェントと参照ドキュメント
| エージェント | 実装対象 | 主に参照するドキュメント |
|---|---|---|
| backend-developer | Go/gin/クリーンアーキテクチャ | `architecture.md`, `api.md`, `database.md` |
| frontend-developer | Next.js/Tailwind CSS | `ui-design.md`, `api.md` |
| aws-developer | AWS CDK（TypeScript） | `architecture.md`, `database.md` |
| unit-test-generator | Go テスト（testing/testify） | 実装コード, `api.md` |

### 実行モデル
各エージェントは**tasks.mdのタスクを1つずつ**処理する:

1. `tasks.md` から担当タスクを確認し、ステータスを `[ ]` → `[/]`（実装中）に更新
2. タスクの「参照仕様」に記載されたドキュメントを読む
3. 実装を完了
4. 受け入れ条件をチェックし、ステータスを `[x]`（完了）に更新
5. **【承認ゲート】** ユーザーが実装を確認
   - ✅ 承認 → 次のタスクへ
   - ❌ 修正要求 → 同じエージェントに追加指示

### 起動例
```
invoke backend-developer with "specs/<機能名>/tasks.md のタスク 2.1 を実装してください。"
```

### 並列実行
同一Wave内で依存関係のないタスクは並列実行可能（例: backend-developerとfrontend-developerを同時起動）。ただし各タスク完了後に個別に承認ゲートを挟む。

---

## Phase 5: レビュー

### 成果物
- レビューレポート（**チャット出力のみ**。ファイルには保存しない）
- 修正が必要な場合のみ `specs/<機能名>/tasks.md` への新規タスク追加

### 実行手順
1. 起動:
   ```
   invoke code-reviewer with "実装されたコードをレビューし、コード品質・セキュリティ・規約準拠を確認してください。"
   ```
2. エージェントの作業:
   - `tasks.md` でレビュー対象タスクを確認
   - 実装コードを読み、品質・セキュリティ・規約準拠を評価
   - **レビュー結果をチャットで提示**（この時点では `tasks.md` を編集しない）
3. **【ユーザー判断】**
   - ✅ 「問題なし」「修正不要」 → 開発完了
   - 🔁 「修正が必要」 → code-reviewerが指摘事項を新規タスク（例: `タスク 2.1-R1`）として `tasks.md` に追加 → Phase 4に戻り該当エージェントが対応

### レビュー指摘の優先度
| 優先度 | 扱い |
|---|---|
| 🔴 Critical | 原則タスク化（セキュリティ脆弱性・データ破損・アーキテクチャ違反） |
| 🟡 Major | 原則タスク化（規約逸脱・パフォーマンス問題・型安全性） |
| 🟢 Minor | ユーザーの意向を確認してから判断 |

---

## 承認ゲートの運用

各エージェント実行後、以下の形式で確認を求める:

```
[エージェント名]が[ファイルパス]を作成しました。

確認ポイント:
- [確認項目1]
- [確認項目2]

内容を確認し、承認する場合は「承認」と入力してください。
修正が必要な場合は、具体的な修正内容を指示してください。
```

**ユーザーの明確な承認なしに次工程のエージェントを起動しない。**

---

## 要件の追跡性

`requirements.md` で付与した要件ID（`REQ-<領域>-<連番>`）を、後続の全ドキュメントで参照する。

```
requirements.md（REQ-AUTH-001 を定義）
  ↓ 「対応要件」表で参照
architecture.md / database.md / api.md / ui-design.md
  ↓ 「対応要件」欄で参照
tasks.md（タスク 2.1 の対応要件: REQ-AUTH-001）
  ↓
実装コード
```

これにより「この実装はどの要件由来か」「この要件は実装されたか」を双方向に追跡できる。

---

## 注意事項

### ファイル所有権の厳守
- 各エージェントは**自分がオーナーのファイルのみ**を編集する
- 他のエージェントのファイルを修正したい場合は、そのオーナーエージェントを起動する
- `tasks.md` のみ、実装エージェント（進捗更新）とcode-reviewer（タスク追加）が更新できる

### `.kiro/` は編集禁止
すべてのエージェントは `.kiro/` 配下（steering / agents / settings / hooks）への書き込みを禁止されている。設定変更はユーザーまたはメインエージェントが行う。

### MCP使用制限
- **Miroボード**: steering file `miro.md` に記載のボードのみ使用
- **Figmaファイル**: steering file `figma.md` に記載のファイルのみ使用

### 設計変更が発生した場合
実装中に設計の不備が判明した場合:
1. 実装を止めず、`tasks.md` の該当タスクに「備考」として記録
2. ユーザーに報告
3. ユーザーの判断で該当設計エージェント（例: db-designer）を再起動して設計を修正
4. 修正後、task-plannerでタスクを再検討

---

## フロー実行例

```
# 新機能「ユーザー認証」の開発

## Phase 1
User: 「ユーザー登録とログイン機能を実装したい」
Assistant: invoke requirements-analyst...
[specs/user-authentication/requirements.md 生成]
User: 「承認」

## Phase 2.1
Assistant: invoke system-architect...
[architecture.md 生成、Miro図作成]
User: 「Lambdaの部分をFargateに変更してください」
Assistant: [修正]
User: 「承認」

## Phase 2.2
Assistant: invoke db-designer...
[database.md 生成、ER図作成]
User: 「承認」

## Phase 2.3
Assistant: invoke api-designer...
[api.md 生成]
User: 「承認」

## Phase 2.4
Assistant: invoke web-ui-designer...
[ui-design.md 生成、Figmaデザイン作成]
User: 「承認」

## Phase 3
Assistant: invoke task-planner...
[tasks.md 生成: Wave 1に3タスク、Wave 2に4タスク]
User: 「承認」

## Phase 4
Assistant: invoke aws-developer with "タスク 1.1 を実装"
[CDKコード生成、tasks.md のステータスを [x] に更新]
User: 「承認」
Assistant: invoke backend-developer with "タスク 1.2 を実装"
...

## Phase 5
Assistant: invoke code-reviewer...
[レビュー結果をチャットで提示: 🔴 Critical 1件、🟢 Minor 2件]
User: 「Criticalだけ修正して」
Assistant: [タスク 2.1-R1 を tasks.md に追加]
Assistant: invoke backend-developer with "タスク 2.1-R1 を実装"
User: 「承認」
```

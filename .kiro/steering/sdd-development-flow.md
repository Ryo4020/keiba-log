---
name: sdd-development-flow
description: 仕様駆動開発フロー。要件定義→設計→タスク分解→実装→レビューの段階的実行と承認ゲート
inclusion: auto
---

# 仕様駆動開発フロー：段階的エージェント実行ガイド

カスタムエージェントを用いた仕様駆動開発の実行フローを定義する。各エージェントが自分の担当ドキュメントを作成し、**成果物ごとにユーザーの承認を受けてから次工程に進む**。

Kiro IDE組み込みのSpec機能（`.kiro/specs/`）は使用しない。仕様ドキュメントはすべて**プロジェクトルート直下の `specs/` ディレクトリ**で管理する。

---

## ディレクトリ構成

```
specs/
└── <機能名>/                          # ケバブケース（例: user-authentication）
    ├── requirements/
    │   ├── overview.md                        # プロジェクト概要・MVP機能（requirements-analyst）
    │   ├── functional-requirements.md         # 機能一覧・ユーザーストーリー列挙・要件ID体系（functional-analyst）
    │   ├── business-rules.md                  # ドメイン固有のビジネスルール（business-rules-analyst）
    │   ├── user-stories/                      # 要件ID単位の受け入れ基準（user-story-analyst）
    │   │   └── <要件ID>-<機能名>.md
    │   └── non-functional-requirements.md     # 非機能要件（nfr-analyst）
    ├── designs/
    │   ├── architecture.md    # アーキテクチャ設計（system-architect）
    │   ├── database.md        # テーブル一覧・アクセスパターン（db-designer、モデル概要セッション）
    │   ├── db-tables/          # テーブルごとの詳細定義（db-designer、テーブルごとに個別起動）
    │   │   └── <table名>.md
    │   ├── api.md             # API概要（api-designer）
    │   ├── api.yml             # エンドポイント詳細（OpenAPI 3.0）（api-designer）
    │   ├── ui-overview.md     # 画面一覧・画面遷移・デザイントークン（ui-architect）
    │   └── ui-designs/         # 画面ごとの詳細デザイン仕様（web-designer）
    │       └── <画面名>.md
    └── tasks/                  # 機能（要件ID）ごとの実装タスク（task-planner）
        ├── foundation.md       # 全機能共通の基盤タスク（最初に1度だけ作成）
        └── <要件ID>.md
```

Phase 2（設計）の全成果物は `specs/<機能名>/designs/` 配下にまとめる。

機能ごとに `specs/<機能名>/` を作成する。1つの巨大な仕様にまとめず、機能単位で分割する。

---

## フロー概要

```
Phase 1: 要件定義（逐次実行）
  ├─ 1.1 requirements-analyst      → requirements/overview.md                    → [承認] →
  ├─ 1.2 functional-analyst        → requirements/functional-requirements.md     → [承認] →
  ├─ 1.3 business-rules-analyst    → requirements/business-rules.md              → [承認] →
  ├─ 1.4 user-story-analyst        → requirements/user-stories/*.md              → [承認] →
  └─ 1.5 nfr-analyst               → requirements/non-functional-requirements.md → [承認] →

Phase 2: 設計（逐次実行、成果物は designs/ 配下）
  ├─ 2.1 system-architect  → designs/architecture.md                                       → [承認] →
  ├─ 2.2 db-designer       → designs/database.md（モデル概要）                              → [承認] →
  │                        → designs/db-tables/<table名>.md（テーブルごとに個別起動・承認） →
  ├─ 2.3 api-designer      → designs/api.md, designs/api.yml                               → [承認] →
  ├─ 2.4 ui-architect      → designs/ui-overview.md                                        → [承認] →
  └─ 2.5 web-designer      → designs/ui-designs/<画面名>.md（画面ごとに起動・承認）           →

Phase 3: タスク分解（機能ごとに個別起動）
  └─ task-planner → tasks/foundation.md（最初の1回のみ）→ tasks/<要件ID>.md（機能ごと）→ [承認] →

Phase 4: 実装（機能ファイル単位、機能間は並列可）
  ├─ backend-developer     → [機能ファイルごとに承認]
  ├─ frontend-developer    → [機能ファイルごとに承認]
  ├─ aws-developer         → [機能ファイルごとに承認]
  └─ unit-test-generator   → [機能ファイルごとに承認]

Phase 5: レビュー
  └─ code-reviewer → チャットで報告 → [ユーザー判断] → 修正必要なら対象の tasks/<要件ID>.md に追加してPhase 4へ
```

---

## Phase 1: 要件定義

要件定義は5エージェントを**逐次実行**する。各エージェントは自分の担当範囲のみを扱い、担当範囲外の項目（ドメインルール・受け入れ基準・非機能要件など）は成果物に書かない。前段の成果物を参照しながら進めるため、順序を守る。

---

## Phase 2: 設計

設計は5エージェントを**逐次実行**する。各エージェントは前段の設計を前提に自分の担当領域を設計するため、順序を守る。成果物はすべて `specs/<機能名>/designs/` 配下に作成する。

db-designerはまず `designs/database.md` を作成するセッションを実行し、その後、`designs/db-tables/<table名>.md` を1つずつ作成する。
ui-architect（画面一覧・画面遷移・デザイントークン）が完了した後、web-designer は `designs/ui-overview.md` に列挙された**画面ごとに個別起動**し、画面単位で承認を受ける。

---

## Phase 3: タスク分解

task-plannerは `requirements/functional-requirements.md` に列挙された**機能（要件ID）ごとに個別起動**する。1回の起動で1機能のタスクを `specs/<機能名>/tasks/<要件ID>.md` に作成し、機能ごとに承認ゲートを挟む。

全機能に共通する基盤実装（要件IDに紐付かない土台）は、個別機能の起動に先立って**最初に1度だけ**例外的に作成し、`tasks/foundation.md` に保存する。

タスクの粒度は「1機能 × 1領域」（例: 「ユーザー登録のバックエンド」）。

タスクの依存や実装確認については、steering file `implementation-rules.md` に定義されている。

---

## Phase 4: 実装

実装エージェントは、担当する機能の `specs/<機能名>/tasks/<要件ID>.md`（または `tasks/foundation.md`）を指定されて起動し、その中の自分の領域（バックエンド／フロントエンド／インフラ／テスト）のタスクを処理する。参照仕様は `specs/<機能名>/designs/` 配下のファイルを指す。

---

## Phase 5: レビュー

レビューレポートは、**チャット出力のみ**で、修正が必要な場合のみレビュー対象タスクが属する `specs/<機能名>/tasks/<要件ID>.md`（または `tasks/foundation.md`）への新規タスク追加。追加するタスクにも依存関係は記載しない。

---

## 注意事項

### ファイル所有権の厳守
- 各エージェントは**自分がオーナーのファイルのみ**を編集する
- `tasks/*.md` のみ、task-planner（作成）・実装エージェント（進捗更新）・code-reviewer（タスク追加）が更新できる

### `.kiro/` は編集禁止
すべてのエージェントは `.kiro/` 配下（steering / agents / settings / hooks）への書き込みを禁止されている。設定変更はユーザーまたはメインエージェントが行う。

### MCP使用制限
- **Miroボード**: steering file `miro.md` に記載のボードのみ使用
- **Figmaファイル**: steering file `figma.md` に記載のファイルのみ使用
  - **作成・編集**: web-designer のみ
  - **閲覧のみ**: frontend-developer（実装時にデザインを参照する）

### デザインの正本はFigma
`designs/ui-designs/<画面名>.md` には「どんな要素があり、どう動くか」の概略しか書かない。レイアウト・寸法・配色・タイポグラフィはFigmaが正本であり、mdには記載しない（二重管理を避ける）。frontend-developer は md で構成と振る舞いを把握し、md に記載された node-id から Figma を参照して見た目を実装する。

### 関連するsteering file
| ファイル | 内容 | 対象エージェント |
|---|---|---|
| `ears-notation.md` | 受け入れ基準のEARS記法の定義と書き方 | user-story-analyst |
| `implementation-rules.md` | 実装時の共通ルール（着手前の既存実装の確認・重複実装の防止・完了判定） | 実装4エージェント、code-reviewer |

### 設計変更が発生した場合
実装中に設計の不備が判明した場合:
1. 実装を止めず、対象の `tasks/*.md`の該当タスクに「備考」として記録
2. ユーザーに報告
3. ユーザーの判断で該当設計エージェントを再起動して設計を修正
4. 修正後、task-plannerで該当機能のタスクを再検討

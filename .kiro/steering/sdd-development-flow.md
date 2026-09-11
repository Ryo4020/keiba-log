---
name: sdd-development-flow
description: SDD方式の段階的開発フロー。10エージェントによる要件定義→設計→タスク分解→実装→レビューの自律実行と承認ゲート
inclusion: auto
---

# SDD開発フロー：段階的エージェント実行ガイド

このSteeringは、Kiroのカスタムエージェントを用いた**Spec-Driven Development（SDD）**の実行フローを定義します。各エージェントが段階的に自律実行し、**成果物生成後に必ずユーザーの承認を受ける**ことで、品質と方向性を保ちながら開発を進めます。

---

## フロー概要

```
Phase 1: 要件定義
  └─ requirements-analyst → [承認] →

Phase 2: 技術設計
  └─ system-architect → [承認] →
  └─ db-designer → [承認] →
  └─ api-designer → [承認] →
  └─ web-ui-designer → [承認] →

Phase 3: タスク分解
  └─ task-planner → [承認] →

Phase 4: 実装
  ├─ backend-developer → [承認]
  ├─ frontend-developer → [承認]
  ├─ aws-developer → [承認]
  └─ unit-test-generator → [承認] →

Phase 5: レビュー
  └─ code-reviewer → [最終確認]
```

---

## Phase 1: 要件定義

### 担当エージェント
- **requirements-analyst**

### 成果物
- `.kiro/specs/<機能名>/requirements.md`

### 実行手順
1. **ユーザー**が機能要求を提示
2. **requirements-analyst**を起動:
   ```
   invoke requirements-analyst with "以下の機能要件をEARS形式でrequirements.mdに整理してください: [ユーザー要求]"
   ```
3. エージェントがヒアリングを実施（必要に応じて追加質問）
4. EARS形式でrequirements.mdを生成
5. **【承認ゲート】** ユーザーが確認:
   - ✅ 承認 → Phase 2へ
   - ❌ 修正要求 → requirements-analystに追加指示して再生成

### 成果物の確認ポイント
- すべての機能要件がEARS形式（WHEN...THE SYSTEM SHALL...）で記述されているか
- エッジケースやエラー処理が網羅されているか
- 曖昧な表現がないか

---

## Phase 2: 技術設計

### 担当エージェント（実行順序）
1. **system-architect** （システムアーキテクチャ）
2. **db-designer** （データモデル）
3. **api-designer** （API仕様）
4. **web-ui-designer** （UI/UX設計）

### 成果物
- `.kiro/specs/<機能名>/design.md`
  - §1: システムアーキテクチャ（system-architect）
  - §2: データモデル（db-designer）
  - §3: API仕様（api-designer）
  - §4: UI/UX設計（web-ui-designer）
- Miroボード（`miro.md` 参照）の図
- Figmaファイル（`figma.md` 参照）のデザイン

### 実行手順

#### 2.1 システムアーキテクチャ設計
1. **system-architect**を起動:
   ```
   invoke system-architect with "requirements.mdを基にシステムアーキテクチャを設計し、design.mdの§1を作成してください。Miroにアーキテクチャ図を作成してください。"
   ```
2. エージェントが以下を実施:
   - requirements.mdを読み込み
   - design.mdを新規作成し、§1にシステムアーキテクチャを記述
   - Miroボードにアーキテクチャ図・シーケンス図を作成
3. **【承認ゲート】** ユーザーが確認:
   - design.md §1の妥当性
   - Miroの図の正確性
   - ✅ 承認 → 2.2へ
   - ❌ 修正要求 → system-architectに追加指示

#### 2.2 データモデル設計
1. **db-designer**を起動:
   ```
   invoke db-designer with "requirements.mdとdesign.md §1を基にDynamoDBのデータモデルを設計し、design.mdの§2を追記してください。MiroにER図を作成してください。"
   ```
2. エージェントが以下を実施:
   - requirements.mdとdesign.md §1を読み込み
   - design.mdに§2を追記（テーブル設計、GSI、アクセスパターン）
   - MiroボードにER図を作成
3. **【承認ゲート】** ユーザーが確認:
   - DynamoDBのテーブル設計（PK/SK、GSI）
   - アクセスパターンの網羅性
   - Miroの図
   - ✅ 承認 → 2.3へ
   - ❌ 修正要求 → db-designerに追加指示

#### 2.3 API仕様設計
1. **api-designer**を起動:
   ```
   invoke api-designer with "requirements.mdとdesign.md §1-2を基にREST API仕様を設計し、design.mdの§3を追記してください。"
   ```
2. エージェントが以下を実施:
   - requirements.mdとdesign.md §1-2を読み込み
   - design.mdに§3を追記（エンドポイント、リクエスト/レスポンス、エラーコード）
3. **【承認ゲート】** ユーザーが確認:
   - エンドポイント設計の妥当性
   - リクエスト/レスポンススキーマ
   - エラーハンドリング戦略
   - ✅ 承認 → 2.4へ
   - ❌ 修正要求 → api-designerに追加指示

#### 2.4 UI/UX設計
1. **web-ui-designer**を起動:
   ```
   invoke web-ui-designer with "requirements.mdとdesign.md §1-3を基にUI/UX設計を行い、design.mdの§4を追記してください。Figmaでワイヤーフレームとデザインを作成してください。"
   ```
2. エージェントが以下を実施:
   - requirements.mdとdesign.md §1-3を読み込み
   - design.mdに§4を追記（画面構成、コンポーネント設計、レスポンシブ対応）
   - Figmaファイルにワイヤーフレームとビジュアルデザインを作成
3. **【承認ゲート】** ユーザーが確認:
   - 画面フローの妥当性
   - UIコンポーネント設計
   - Figmaデザイン
   - ✅ 承認 → Phase 3へ
   - ❌ 修正要求 → web-ui-designerに追加指示

---

## Phase 3: タスク分解

### 担当エージェント
- **task-planner**

### 成果物
- `.kiro/specs/<機能名>/tasks.md`

### 実行手順
1. **task-planner**を起動:
   ```
   invoke task-planner with "requirements.mdとdesign.mdを基に実装タスクをtasks.mdに分解してください。"
   ```
2. エージェントが以下を実施:
   - requirements.mdとdesign.mdを読み込み
   - tasks.mdを生成（タスクごとに説明、依存関係、ステータスを記載）
3. **【承認ゲート】** ユーザーが確認:
   - タスクの粒度と完全性
   - 依存関係の妥当性
   - 必須/オプションの区別
   - ✅ 承認 → Phase 4へ
   - ❌ 修正要求 → task-plannerに追加指示

---

## Phase 4: 実装

### 担当エージェント（並列実行可能）
- **backend-developer** （バックエンド実装）
- **frontend-developer** （フロントエンド実装）
- **aws-developer** （インフラ実装）
- **unit-test-generator** （ユニットテスト生成）

### 成果物
- バックエンドコード（Go/gin/クリーンアーキテクチャ）
- フロントエンドコード（Next.js/Tailwind CSS）
- インフラコード（AWS CDK/TypeScript）
- ユニットテストコード（Go: testing, testify）

### 実行手順

#### 実行モデル
各エージェントは**tasks.mdのタスクを1つずつ選択**し、以下を実施:
1. タスクの説明と依存関係を確認
2. 実装を完了
3. tasks.mdのステータスを`[x]`に更新
4. **【承認ゲート】** ユーザーが実装を確認
   - ✅ 承認 → 次のタスクへ
   - ❌ 修正要求 → エージェントに追加指示

#### 4.1 バックエンド実装
1. **backend-developer**を起動:
   ```
   invoke backend-developer with "tasks.mdのTask [番号]を実装してください。"
   ```
2. エージェントが以下を実施:
   - design.md §2-3を参照してバックエンドを実装
   - Go/gin/クリーンアーキテクチャに準拠
   - tasks.mdのステータスを更新
3. **【承認ゲート】** 実装完了後にユーザーが確認

#### 4.2 フロントエンド実装
1. **frontend-developer**を起動:
   ```
   invoke frontend-developer with "tasks.mdのTask [番号]を実装してください。"
   ```
2. エージェントが以下を実施:
   - design.md §3-4を参照してフロントエンドを実装
   - Next.js/Tailwind CSSを使用
   - tasks.mdのステータスを更新
3. **【承認ゲート】** 実装完了後にユーザーが確認

#### 4.3 インフラ実装
1. **aws-developer**を起動:
   ```
   invoke aws-developer with "tasks.mdのTask [番号]を実装してください。"
   ```
2. エージェントが以下を実施:
   - design.md §1を参照してAWS CDKでインフラを実装
   - サーバーレスアーキテクチャでコスト最適化
   - tasks.mdのステータスを更新
3. **【承認ゲート】** 実装完了後にユーザーが確認

#### 4.4 ユニットテスト生成
1. **unit-test-generator**を起動:
   ```
   invoke unit-test-generator with "tasks.mdのTask [番号]に対応するユニットテストを生成してください。"
   ```
2. エージェントが以下を実施:
   - バックエンドコードを読み込み
   - Go testing/testifyでユニットテストを生成
   - tasks.mdのステータスを更新
3. **【承認ゲート】** テストコード確認後にユーザーが承認

---

## Phase 5: レビュー

### 担当エージェント
- **code-reviewer**

### 成果物
- レビューレポート

### 実行手順
1. **code-reviewer**を起動:
   ```
   invoke code-reviewer with "実装されたコードをレビューし、コード品質、セキュリティ、規約準拠を確認してください。"
   ```
2. エージェントが以下を実施:
   - 実装コードを読み込み
   - コード品質（可読性、保守性）を評価
   - セキュリティ脆弱性をチェック
   - コーディング規約への準拠を確認
   - レビューレポートを生成
3. **【最終確認】** ユーザーがレビュー結果を確認:
   - ✅ 承認 → 開発完了
   - ❌ 修正要求 → 該当エージェントに差し戻し

---

## エージェント起動の実務的な方法

### invoke_sub_agent による起動例
```
invoke backend-developer with "tasks.mdのTask 3.1: ユーザー認証エンドポイントの実装を行ってください。design.md §2-3を参照してください。"
```

### 承認ゲートの実装
各エージェント実行後、以下の確認を実施:
```
[エージェント名]が[成果物名]を生成しました。

確認ポイント:
- [確認項目1]
- [確認項目2]

内容を確認し、承認する場合は「承認」と入力してください。
修正が必要な場合は、具体的な修正内容を指示してください。
```

---

## 注意事項

### ファイル編集の競合回避
- **design.md**は4つのエージェントが逐次編集するため、**必ず順序を守る**
- 並行編集は行わない（system-architect → db-designer → api-designer → web-ui-designer）

### MCP使用制限の徹底
- **Miroボード**: `miro.md` steering fileに記載のボードのみ使用
- **Figmaファイル**: `figma.md` steering fileに記載のファイルのみ使用
- エージェントのプロンプトに明示されているため、自動的に遵守される

### タスクの並行実行
- Phase 4では、**依存関係のないタスクは並列実行可能**
- ただし、各タスク完了後に個別に承認ゲートを挟む
- 例: backend-developerとfrontend-developerは並行実行可能

---

## フロー実行例

```
# 新機能「ユーザー認証」の開発

## Phase 1
User: 「ユーザー登録とログイン機能を実装したい」
Assistant: invoke requirements-analyst...
[requirements.md生成]
User: 「承認」

## Phase 2.1
Assistant: invoke system-architect...
[design.md §1生成、Miro図作成]
User: 「MiroのLambda部分をAPI Gatewayに変更してください」
Assistant: [修正]
User: 「承認」

## Phase 2.2
Assistant: invoke db-designer...
[design.md §2追記、ER図作成]
User: 「承認」

## Phase 2.3
Assistant: invoke api-designer...
[design.md §3追記]
User: 「承認」

## Phase 2.4
Assistant: invoke web-ui-designer...
[design.md §4追記、Figmaデザイン作成]
User: 「承認」

## Phase 3
Assistant: invoke task-planner...
[tasks.md生成]
User: 「承認」

## Phase 4.1
Assistant: invoke backend-developer with "Task 1.1を実装"
[バックエンドコード生成]
User: 「承認」
Assistant: invoke backend-developer with "Task 1.2を実装"
...

## Phase 5
Assistant: invoke code-reviewer...
[レビューレポート生成]
User: 「承認、リリース準備を進めてください」
```

---

## まとめ

このSteeringに従うことで:
- ✅ 各エージェントが自律的に専門分野のタスクを実行
- ✅ 成果物ごとにユーザー承認を受け、品質を担保
- ✅ 段階的に開発を進め、手戻りを最小化
- ✅ SDDの方針（Requirements → Design → Tasks → Implementation → Review）に準拠

**次回の開発時**: このSteeringを参照しながら、Phase 1から順次エージェントを起動してください。

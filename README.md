# Shared Agent

ブラウザーゲーム向けの共有Copilot Agentパックです。

## 運用方針

このリポジトリは各プロジェクトの `.github/shared` にGitサブモジュールとして
追加します。プロジェクト固有の指示、Agent、Skillは親リポジトリの `.github`
で保守し、共有する内容だけをここへ置きます。

## ワークフロー

```text
ゲーム仕様
  ↓
ゲーム設計・ADR
  ↓
実装計画・タスク
  ↓
実装
  ↓
仕様照合・検証
```

主要成果物は次の場所を想定します。

```text
specs/<feature>/
  spec.md
  design.md
  plan.md
  tasks.md
  verification.md
docs/adr/
  0001-<decision>.md
```

## Skills

- `game-spec`: ゲーム体験と受け入れ条件を仕様化
- `game-design`: 設計と必要なADRを作成
- `game-plan`: 実装計画と依存関係を整理
- `game-implement`: tasksに従って実装
- `game-verify`: 仕様・性能・ブラウザー挙動を検証

## Agents

- `game-architect.agent.md`: 設計・ADRのレビュー
- `game-reviewer.agent.md`: 実装と検証のレビュー

## 契約

- 共有資産は `.github/shared` を正本にします。
- 親プロジェクト固有の `.github` ファイルを共有リポジトリから上書きしません。
- 実行可能ファイルの追加・変更はレビュー対象です。
- 生成物のテンプレートは `templates/` に置きます。
- 外部アクセス、依存関係導入、Git操作は呼び出し側で明示承認します。

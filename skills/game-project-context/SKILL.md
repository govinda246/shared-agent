---
name: game-project-context
description: 仕様作成前にプロジェクト固有の制約、既存ゲーム、ADR、セキュリティ境界を読み込む。
---

# Game Project Context

既存のREADME、Copilot指示、ADR、現在のゲームコード、テスト構成を確認する。
外部通信、依存追加、サンドボックス、実行権限、対象ブラウザー、性能目標を整理し、
後続Skillが参照できる短い前提一覧を返す。

不足情報は推測で補うのではなく、仕様に仮定として記録するか質問する。

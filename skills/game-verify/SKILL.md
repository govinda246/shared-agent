---
name: game-verify
description: 実装をゲーム仕様、ADR、テスト、性能、ブラウザー要件と照合して検証する。
---

# ゲーム検証

`spec.md`、`design.md`、ADR、`tasks.md` と実装を比較し、
`verification.md` を作成する。

## 検証範囲

- 仕様IDごとの実装・テスト対応
- 単体・統合テスト
- build、typecheck、lint
- 対象ブラウザーと解像度
- キーボード、マウス、タッチ
- pause/resume、tab切替、再読込
- FPS、フレーム時間、初回ロード
- 画像・音声の失敗時挙動
- ADRとの整合性
- 手触りや視覚面の手動確認

自動検証できない項目は未実施として明記し、推測で合格にしない。

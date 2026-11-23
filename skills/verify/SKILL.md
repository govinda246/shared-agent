---
name: verify
description: 実装を仕様、ADR、テスト、性能、ランタイム要件と照合して検証する。
---

# 検証

`spec.md`、`design.md`、ADR、`tasks.md` と実装を比較し、
`verification.md` を作成する。

開始前に `hooks.yml` の `before_verify`、完了前に `after_verify` を処理する。

## 検証範囲

- 仕様IDごとの実装・テスト対応
- 単体・統合テスト
- build、typecheck、lint
- 対象ランタイム、ホストアプリ、デバイス、解像度
- 対象入力方式
- キャンセル、再起動、再読込、ホスト連携
- 性能、メモリ、ロード時間
- 外部リソースまたはアセット失敗時の挙動
- ADRとの整合性
- ドメイン固有の手動確認

自動検証できない項目は未実施として明記し、推測で合格にしない。

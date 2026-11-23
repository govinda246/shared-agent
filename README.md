# Shared Development Agent

仕様、設計、実装、検証を段階化する汎用 Copilot Agent
パックです。目的は、モデルを常に細かい作業へ委任することではなく、
**1タスクを正しく完了するまでの総コストを下げること**です。

## 基本方針

- 仕様と受け入れ条件を先に確定する
- 軽微な作業は委任せず、現在のセッションで直接処理する
- 委任は十分な実装出力があり、受け渡しコストを吸収できる場合だけ行う
- 重要判断と定型実装を分離する
- 独立レビューは高リスク変更だけへ起動する
- テストだけで操作感、視覚、音を合格扱いしない
- 複数人で同一のモデル経路を使えるよう、共有Agentへ既定モデルを固定する

詳細な委任条件は [ROUTING.md](./ROUTING.md) に定義しています。

## Agent 構成

```text
                    ユーザー
                       |
                   conductor
                   /   |   \
            Direct  Implementer  Verifier
                     |            |
               architect / reviewer
                    （高リスクのみ）
```

| Agent | 主な責務 | 起動する条件 |
| --- | --- | --- |
| `conductor` | 工程判定、Task Packet、最終判断 | すべての依頼の窓口。カスタムAgent一覧に表示 |
| `implementer` | 承認済み設計に沿った実装 | Conductorからのみ起動 |
| `verifier` | テスト、build、仕様対応の証拠収集 | Conductorからのみ起動 |
| `architect` | 状態、ライフサイクル、外部連携、ADRの独立設計レビュー | Conductorからのみ起動 |
| `reviewer` | 実装と検証結果の独立レビュー | Conductorからのみ起動 |

Agentごとのモデルは共有パックで `model-policy: required` により固定します。
既定はConductor、Architect、Reviewerが `MAI Code 2`、ImplementerとVerifierが
`MAI Code 2 Flash` です。変更時は、親プロジェクトのモデル運用文書と変更履歴を
更新し、共有Agentへ反映します。

カスタムAgent一覧には `user-invocable: true` の `conductor` だけを表示します。
その他のAgentは `user-invocable: false` とし、Conductorから委任する内部Agentとして
使用します。

## 経路の選び方

ファイル数だけでルーティングしません。判断対象は仕様の不確実性、状態空間、
失敗時の影響、既読コンテキスト、委任後の再確認コストです。

| 経路 | 例 | 避けるべき例 |
| --- | --- | --- |
| Direct | 数行修正、短い検索、既読ファイルの小変更 | 会話履歴を渡して1行直す |
| Implementer | 入出力と対象ファイルが確定した機能実装 | 仕様を実装担当に決めさせる |
| Verifier | 対象テスト、typecheck、仕様ID照合 | 手動確認できない操作を自動合格にする |
| Independent Review | 状態、ライフサイクル、同期、互換性、性能への影響 | 単純な差分を同じ証拠で再確認する |

### 直接処理を優先する作業

- 数行から数十行程度の修正
- 1回の検索やファイル読込で終わる調査
- 親がすでに必要なコンテキストを持つ変更
- テスト結果の表示や短い要約
- 委任準備より直接実行のほうが短い作業

### 汎用の独立レビューゲート

- 状態所有、ライフサイクル、状態遷移
- 時間処理、並列処理、同期、永続化
- 入力方式、ホスト連携、外部プロセス
- 性能、メモリ、ロード時間、互換性
- ADRが必要な変更

ドメイン固有のゲートは、プロジェクト固有SkillまたはHookへ追加します。
ゲーム、DCCツール、ツールアプリ、ライブラリのために共有Agent本体を分岐させません。

## Task Packet

子へ渡すのは会話履歴全体ではなく、次を含む短いTask Packetです。

```text
タスクID
完了条件
受け入れ条件
編集可能ファイル
参照する仕様・設計・ADRの該当箇所
実行する検証
禁止事項
試行上限
親へ差し戻す条件
```

次のいずれかで、実装担当は再試行せずConductorへ戻します。

- 同じ失敗結果が2回続く
- 新しい仮説や証拠が得られない
- 編集範囲外への波及が必要
- 仕様またはADRが不足している
- 一意に決められない設計判断が必要
- コストまたは試行上限へ到達した

## ワークフロー

```text
機能仕様
    ↓ 受け入れ条件レビュー
設計・ADR
    ↓ 代替案・影響・再検討条件レビュー
実装計画・タスク
    ↓ 依存関係・検証ゲートレビュー
実装
    ↓ 証拠収集
仕様照合・検証
    ↓ 高リスクの場合のみ独立レビュー
完了報告
```

主要成果物は次を想定します。

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

| Skill | 役割 |
| --- | --- |
| `project-context` | プロジェクト固有の制約と現状を読む |
| `spec` / `spec-review` | 仕様と受け入れ条件を作成・レビュー |
| `technical-context` | 実装、入力、時間、アセット、性能の前提を抽出 |
| `design` / `adr-review` | 設計とADRを作成・レビュー |
| `plan` / `plan-review` | 依存関係、追跡、検証ゲートを計画・レビュー |
| `implementation-preflight` / `implement` | 実装前提を確認し、承認済みタスクを実装 |
| `verification-context` / `verify` / `verification-review` | 検証範囲と証拠を確認・レビュー |

## ドメインSkillの切替

共有Skillは汎用工程を提供します。ドメイン固有の確認は、対象リポジトリまたは
サブモジュールの `.github/skills/` に追加し、Conductorが依頼に応じて併読します。

例:

| ドメイン | 追加Skillの例 |
| --- | --- |
| 2Dブラウザーゲーム | game-context、browser-verification、scene-safety |
| DCCツール | host-api-context、undo-redo、plugin-verification |
| ツールアプリ | persistence-context、permission-review、app-verification |
| ライブラリ | public-api-context、compatibility-review、package-verification |

共有Hookをドメインごとに分岐させず、共有の工程HookへドメインSkillを追加する形で
親プロジェクト固有Hookを拡張します。

## Hooks

[`hooks.yml`](./hooks.yml) は工程前後の必須・任意Skillを宣言します。
Hookはプロセスを自動起動せず、対応するAgentやSkillを呼ぶための制御データです。

| Event | Mode | 目的 |
| --- | --- | --- |
| `before_spec` | required | プロジェクト文脈 |
| `after_spec` | required | スコープと受け入れ条件 |
| `before_design` | required | 技術・セキュリティ制約 |
| `after_design` | required | ADRレビュー |
| `after_plan` | required | 追跡・依存関係・検証ゲート |
| `before_implement` | required | 実装前提確認 |
| `after_implement` | required | 仕様・テスト・ドメイン検証 |
| `before_verify` | optional | 対象ランタイム、ホスト、端末、性能予算 |
| `after_verify` | required | 証拠と未実施項目のレビュー |

必須Hookが失敗した場合、後続工程へ進みません。プロジェクト固有Hookは共有Hookを
変更せず、親プロジェクトの `.github/hooks.yml` へ追加します。

## 利用方法

このリポジトリを親プロジェクトの `.github/shared` に追加し、次の場所を
Copilot の検索対象にします。

```text
.github/agents/
.github/shared/agents/
.github/shared/skills/
```

プロジェクト固有の指示、Agent、Skill、Hookは親プロジェクトの `.github` で保守し、
共有内容はこのリポジトリで保守します。

## モデル設定とコスト測定

複数人で同じプロジェクトを管理するときの再現性を優先し、共有Agentの
frontmatterへモデルと `model-policy: required` を明記します。既定は次の構成です。

- Conductor/Direct、Architect/Reviewer: MAI Code 2
- `implementer` / `verifier`: MAI Code 2 Flash
- フォールバックではなく、必要時は共有設定を更新して新モデルへ切り替える

Fusionなど別のルーティング方式を利用する場合も、共有Agentの
`model` / `model-policy` を意図的に変更し、その理由、期間、解除条件を
親プロジェクトのモデル運用文書へ記録します。

代表タスクで少なくとも次を実測してください。

1. 指定モデルが実際に選ばれたか
2. 総入力・出力トークン
3. 親と子それぞれの呼び出し回数
4. 再試行回数
5. タスク完了時間
6. テスト結果と人手介入量
7. 1タスクを完了するまでの総クレジット

## GitHub Pages

[`docs/index.html`](./docs/index.html) に、初心者向けの視覚的な運用ガイドを置きます。
GitHub Actions が `main` への変更で静的サイトを公開します。外部ランタイムや
Webアプリ依存はありません。

## 契約

- `.github/shared` を共有正本として利用する
- 親プロジェクト固有ファイルを共有リポジトリから上書きしない
- 実行可能ファイルの追加・変更をレビュー対象にする
- 生成物テンプレートを `templates/` に置く
- 外部アクセス、依存関係追加、Git操作、破壊的操作、権限変更は呼び出し側で承認する
- 機密情報や大きなアセットをAgentコンテキストへ入れない

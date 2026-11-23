# Shared Game Agent

ブラウザーゲームの仕様、設計、実装、検証を段階化する共有 Copilot Agent
パックです。目的は、モデルを常に細かい作業へ委任することではなく、
**1タスクを正しく完了するまでの総コストを下げること**です。

## 基本方針

- 仕様と受け入れ条件を先に確定する
- 軽微な作業は委任せず、現在のセッションで直接処理する
- 委任は十分な実装出力があり、受け渡しコストを吸収できる場合だけ行う
- 重要判断と定型実装を分離する
- 独立レビューは高リスク変更だけへ起動する
- テストだけで操作感、視覚、音を合格扱いしない
- モデル名や料金を共有プロンプトへ固定しない

詳細な委任条件は [ROUTING.md](./ROUTING.md) に定義しています。

## Agent 構成

```text
                    ユーザー
                       |
                 game-conductor
                   /   |   \
            Direct  Implementer  Verifier
                     |            |
              game-architect / game-reviewer
                    （高リスクのみ）
```

| Agent | 主な責務 | 起動する条件 |
| --- | --- | --- |
| `game-conductor` | 工程判定、Task Packet、最終判断 | すべての依頼の窓口 |
| `game-implementer` | 承認済み設計に沿った実装 | 範囲と受け入れ条件が明確で、出力がまとまっている |
| `game-verifier` | テスト、build、仕様対応の証拠収集 | 実装済みで検証項目が明確 |
| `game-architect` | 状態、時間、Scene、入力、ADRの独立設計レビュー | 変更困難性が高い、または選択肢が競合する |
| `game-reviewer` | 実装と検証結果の独立レビュー | 高リスク変更、検証失敗、仕様逸脱の懸念 |

Agentごとのモデルは共有パックで固定しません。利用中の Copilot が対応する
方法で親プロジェクトやユーザー設定から割り当ててください。

## 経路の選び方

ファイル数だけでルーティングしません。判断対象は仕様の不確実性、状態空間、
失敗時の影響、既読コンテキスト、委任後の再確認コストです。

| 経路 | 例 | 避けるべき例 |
| --- | --- | --- |
| Direct | 数行修正、短い検索、既読ファイルの小変更 | 会話履歴を渡して1行直す |
| Implementer | 入出力と対象ファイルが確定した機能実装 | 仕様を実装担当に決めさせる |
| Verifier | 対象テスト、typecheck、仕様ID照合 | 手動確認できない操作を自動合格にする |
| Independent Review | Scene、状態、時間、同期、性能への影響 | 単純な差分を同じ証拠で再確認する |

### 直接処理を優先する作業

- 数行から数十行程度の修正
- 1回の検索やファイル読込で終わる調査
- 親がすでに必要なコンテキストを持つ変更
- テスト結果の表示や短い要約
- 委任準備より直接実行のほうが短い作業

### 独立レビューが必要な変更

- Sceneの生成・破棄、状態所有、状態遷移
- delta time、固定タイムステップ
- 入力の優先順位
- pause/resume、focus喪失
- 永続化、ネットワーク同期
- アセット失敗
- FPS、フレーム時間、メモリ、初回ロード
- ADRが必要な変更

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
ゲーム仕様
    ↓ 受け入れ条件レビュー
ゲーム設計・ADR
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
| `game-project-context` | プロジェクト固有の制約と現状を読む |
| `game-spec` / `game-spec-review` | 仕様と受け入れ条件を作成・レビュー |
| `game-technical-context` | 実装、入力、時間、アセット、性能の前提を抽出 |
| `game-design` / `game-adr-review` | 設計とADRを作成・レビュー |
| `game-plan` / `game-plan-review` | 依存関係、追跡、検証ゲートを計画・レビュー |
| `game-implementation-preflight` / `game-implement` | 実装前提を確認し、承認済みタスクを実装 |
| `game-verification-context` / `game-verify` / `game-verification-review` | 検証範囲と証拠を確認・レビュー |

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
| `after_implement` | required | 仕様・テスト・ブラウザー検証 |
| `before_verify` | optional | 対象ブラウザー、端末、性能予算 |
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

モデル固有のfrontmatterや料金体系はCopilotの実装ごとに変化するため、
共有Agentへ恒久固定しません。利用者側で次を設定します。

- Conductor/Directで使うモデル
- `game-implementer` で使う軽量モデル
- `game-architect` / `game-reviewer` で使う独立判断モデル
- フォールバックと試行上限

代表タスクで少なくとも次を実測してください。

1. 指定モデルが実際に選ばれたか
2. 総入力・出力トークン
3. 親と子それぞれの呼び出し回数
4. 再試行回数
5. タスク完了時間
6. テスト結果と人手介入量
7. 1タスクを完了するまでの総クレジット

## GitHub Pages

公開ガイド: https://govinda246.github.io/shared-agent/

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

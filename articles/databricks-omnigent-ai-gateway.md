---
title: "Omnigent × Unity AI Gateway — 組み合わせて使うエージェントガバナンス"
emoji: "🧭"
type: "tech"
topics:
  - "databricks"
  - "omnigent"
  - "aigateway"
  - "agent"
  - "governance"
published: true
published_at: "2026-08-10 22:45"
---

マネージド Omnigent でエージェントを動かし、Unity AI Gateway でモデル呼び出しを統治する。本記事では役割の切り分け、アーキテクチャ、デモ実装（`omnigent_demo`）での確認手順をまとめます。

## 結論（先に）

| レイヤー | 製品 | 担うこと |
| --- | --- | --- |
| 実行・オーケストレーション | **Omnigent** | セッション UI、ハーネス切替、エージェント YAML、セッションポリシー、Sandbox 実行、Share |
| 統制・観測 | **Unity AI Gateway** | Foundation Model / 外部モデルへの経路、権限、予算・レート、ガードレール、トレーシング |

Omnigent は「エージェント艦隊の操縦席」、AI Gateway は「すべてのモデル呼び出しが通る関所」です。マネージド Sandbox ではモデルアクセスは必ず AI Gateway 経由になり、自前 API キーの持ち込みはできません。

## なぜ組み合わせるか

エージェント単体のデモでは足りない論点が、エンタープライズではすぐに現れます。

1. **マルチハーネス** — Claude Code / Codex / 自作エージェントを同じ運用面で扱いたい
2. **セッション内ガバナンス** — シェル実行前の承認、ツール回数上限、セッション予算
3. **組織横断の統制** — モデル権限、支出キャップ、監査ログ、MCP アクセス

1 と 2 は Omnigent の得意領域、3 は Unity AI Gateway（および Unity Catalog）の得意領域です。マネージド Omnigent on Databricks は、公式にも「Model access through the Foundation Model APIs and AI Gateway」と明記されており、最初からこの組み合わせを前提にしています。

## アーキテクチャ

```text
ブラウザ  <workspace-url>/omnigent
    │
    ▼
マネージド Omnigent サーバー（ワークスペース IdP 連携）
    │
    ▼
Databricks Sandbox（隔離実行ホスト）
    │  エージェント・ツール・ファイル操作
    ▼
Unity AI Gateway
    │  権限 / 予算 / ガードレール / トレース
    ▼
Foundation Model APIs（例: databricks-claude-*, databricks-gpt-*）
```

ポイントは次の 3 つです。

- **Omnigent** は「誰が・どのエージェントで・どのハーネスで」作業するかを束ねる
- **Sandbox** は実行場所。モデル呼び出しはここから Gateway へ強制ルーティングされる
- **AI Gateway** は「どのモデルを・いくらまで・どのような制約で」呼べるかを組織ポリシーとして強制する

ローカルホスト（`omni host`）を使う場合でも、モデル認証で Databricks を選べば Gateway 統治を維持できます。逆に BYO API キーにすると、その経路は Gateway の外に出ます。

## 役割分担の具体例

| やりたいこと | 主に触る場所 | 例 |
| --- | --- | --- |
| ハーネスを `claude-sdk` → `openai-agents` に変える | Omnigent（YAML / UI） | `executor.harness` の 1 行 |
| シェル前に人間承認を入れる | Omnigent セッションポリシー | `ask_on_os_tools` |
| セッション支出を 1.5 USD で止める | Omnigent `cost_budget`（＋組織側の Gateway 予算） | デモ YAML の `max_cost_usd` |
| ワークスペース全体でモデル権限を制限する | Unity Catalog / AI Gateway | `system.ai` の権限、Foundation Model permissions |
| 呼び出しの監査・トレース | AI Gateway | unified tracing / inference telemetry |
| マルチエージェントの委譲設計 | Omnigent | 親 + researcher / summarizer |

セッションポリシーと組織ポリシーは競合しません。実務では「組織は広めの上限、セッションはより厳しく」が自然です。マネージド Omnigent ではカスタム Python ポリシーは使えず、ビルトインのコンテキストポリシーのみです。

## デモ実装の要点

学習用 DAB（`omnigent_demo`）では、Omnigent × AI Gateway 向けに次のような資材を置いています。

| パス | 内容 |
| --- | --- |
| `agents/gateway-governed.yaml` | Databricks auth + ビルトインポリシーの Gateway 連携デモエージェント |
| `tasks/gateway-prompts.md` | Composer に貼る確認用プロンプト |
| `src/notebooks/02_omnigent_ai_gateway.py` | 組み合わせの要約 Notebook |

### エージェント定義の骨格

`gateway-governed.yaml` の要点は次のとおりです。

```yaml
name: gateway_governed

executor:
  harness: claude-sdk
  model: databricks-claude-sonnet-4-6
  auth:
    type: databricks
    profile: DEFAULT   # CLI / ローカル時。Sandbox ではワークスペース IdP

policies:
  approve_os_tools:
    type: function
    handler: omnigent.policies.builtins.safety.ask_on_os_tools

  cap_tool_calls:
    type: function
    handler: omnigent.policies.builtins.safety.max_tool_calls_per_session
    factory_params:
      limit: 15

  session_budget:
    type: function
    handler: omnigent.policies.builtins.cost.cost_budget
    factory_params:
      max_cost_usd: 1.50
      ask_thresholds_usd: [0.40, 0.90]
      expensive_models: ["opus", "sonnet", "gpt-5"]
```

ここでの `auth.type: databricks` と `databricks-*` モデル ID が、「モデル呼び出しを AI Gateway 側に載せる」実装上のスイッチです。Sandbox ホストではそもそも BYO キーが使えず、同じ経路に固定されます。

## ハンズオン手順（最短）

### 0. 前提

- Omnigent / Sandbox プレビューが有効
- リージョンが Unity AI Gateway（および Sandbox）対応
- （推奨）デモ資材を `databricks bundle deploy` 済み

### 1. 資材をデプロイ

```bash
databricks bundle validate
databricks bundle deploy
databricks bundle run demo_portal   # 任意: プロンプト配布用ポータル
```

同期先の目安:

```text
/Workspace/Users/<you>/.bundle/omnigent_demo/<target>/files/
  ├── agents/gateway-governed.yaml
  ├── tasks/gateway-prompts.md
  └── ...
```

### 2. Omnigent でセッション開始

1. `<workspace-url>/omnigent` を開く
2. New session → ホスト **Sandbox**
3. `tasks/gateway-prompts.md` の「前提確認」を投げる

モデル応答が返り、ツールコールが見えれば、Omnigent → Sandbox → AI Gateway → Foundation Model の経路は生きています。

### 3. Gateway 連携エージェントを立てる

- YAML をチャットに貼り「同等のエージェントを立てて」と依頼する
- または `gateway-prompts.md` の YAML 生成依頼プロンプトを使う

### 4. 二重ガバナンスを観察する

| 確認 | 操作 | 期待 |
| --- | --- | --- |
| OS 承認 | ディレクトリ一覧などシェルを伴う指示 | ASK → 承認 / 拒否 |
| ツール上限 | 小さなファイルを連続生成 | 上限後にブロック説明 |
| コスト | 長文生成 + 低めの `cost_budget` | 警告 → 高価モデル時はダウングレード誘導 |
| 組織側 | AI Gateway / UC でモデル権限や予算を確認 | Omnigent 外でも同じモデル経路が統治対象 |

### 5. 切り分け

| 症状 | まず見る場所 |
| --- | --- |
| `/omnigent` が 404 | Omnigent プレビュー |
| Sandbox が出ない | Sandbox プレビュー / リージョン |
| モデル呼び出し失敗 | AI Gateway / Foundation Model の利用可否・権限 |
| ポリシーが効かない | セッション info パネル、YAML の `policies` |

## 設計上の注意

1. **Omnigent ポリシー ≠ AI Gateway ポリシー**  
   前者はセッション／エージェント定義、後者は組織のコントロールプレーン。デモの `cost_budget` は「エージェント側のソフトなブレーキ」、Gateway の hard spend cap は「組織のハード上限」と捉えると説明しやすいです。

2. **Sandbox では BYO キー不可**  
   学習・統制デモでは利点です。ローカルホストで外部キーを使うと Gateway 外に出るため、記事・ハンズオンの主軸は Sandbox + Databricks auth に置いてください。

3. **マネージド版の制約**  
   カスタム Python ポリシー関数は不可。ビルトイン（`ask_on_os_tools` / `max_tool_calls_per_session` / `cost_budget` など）で足りる範囲をデモにします。

4. **Cursor ハーネス**  
   `harness: cursor` には `auth.type: databricks` を付けない（既存デモと同じ注意）。

## まとめ

- **Omnigent** でエージェントを定義・実行・共有し、**AI Gateway** でモデル経路を組織統治する
- マネージド Sandbox ではこの組み合わせがデフォルト経路になる
- 実装のスイッチは `auth.type: databricks` + `databricks-*` モデルと、セッション側ビルトインポリシー
- `agents/gateway-governed.yaml` と `tasks/gateway-prompts.md` で、経路確認から二重ガバナンス観察まで一通り再現できる

## 参考

- [Omnigent on Databricks](https://docs.databricks.com/aws/en/omnigent/)
- [Omnigent quickstart](https://docs.databricks.com/aws/en/omnigent/quickstart)
- [AI governance with Unity AI Gateway](https://docs.databricks.com/aws/en/ai-gateway/)
- [Govern model APIs](https://docs.databricks.com/aws/en/ai-gateway/model-services)
- [AI governance at Data + AI Summit 2026（Unity AI Gateway）](https://www.databricks.com/blog/ai-governance-data-ai-summit-2026-whats-new-unity-ai-gateway)

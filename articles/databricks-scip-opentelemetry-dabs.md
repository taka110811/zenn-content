---
title: "SCIPの数理最適化をOpenTelemetryで計装し、DatabricksのUCに探索ログを残してみた"
emoji: "🧮"
type: "tech"
topics:
  - "opentelemetry"
  - "databricks"
  - "scip"
  - "optimization"
  - "unitycatalog"
published: true
published_at: "2026-08-11 10:50"
---

前回までの記事では、AI エージェントを OpenTelemetry で計装し、トレースを Databricks Unity Catalog に残すところまでやった。

- [AIエージェントをOpenTelemetryで計装し、DatabricksのLakehouseにトレースを残してみた](https://zenn.dev/babysteps/articles/databricks-opentelemetry-ai-agent-dabs)
- [OpenTelemetryでAIエージェントのLLM比較と失敗スパンアラートまでやってみた](https://zenn.dev/babysteps/articles/databricks-otel-llm-compare-alert)

今回は対象を **LLM ではなく SCIP（数理最適化ソルバ）** に変える。MIP を解くとき、「最適値がいくつだったか」だけでは足りないことが多い。遅い・締まらない・解が更新されない、といった症状を切り分けるには、**探索の途中経過**が見える必要がある。

そこで PySCIPOpt の求解を OpenTelemetry で計装し、分岐・LP・暫定解更新などの内部計算を **Unity Catalog（Delta）** に貯めて SQL で見るデモを Declarative Automation Bundles（DABs）で回した。

この記事も Zenn コンテスト「OpenTelemetryの知見を、記事にしよう」**OpenTelemetry部門**向けの実践メモ。バックエンドは Databricks だが、計装自体はベンダー非依存の OTel SDK である。

## 結論だけ先に

- **測る**: OpenTelemetry SDK。SCIP の Eventhdlr から span / span event を出す
- **動かす**: Databricks Job（DABs / Serverless）
- **残す**: メモリ上のスパンを Spark で UC Delta に append（前回の Spark/Delta sink と同じ考え方）
- **見えるもの**: モデル規模、最終結果（status / obj / gap）、探索イベント（最良解・LP・分岐・ノード着目）
- 実際に走らせたサンプルは多期間生産計画 MIP。`optimal`、目的関数値 `3255.5`、UC に 4 spans を書き込んだ

## やりたかったこと

数理最適化のログで欲しいのは、だいたい次の 3 層。

| レイヤ | 例 | 用途 |
|--------|----|------|
| 静的情報 | 問題名、変数数、制約数、時間制限 | 何を解いたかの識別 |
| 結果 | status、目的関数値、gap、求解時間 | 解けたか・どれくらい良いか |
| 探索の途中 | ノード数、LP、暫定解更新、分岐、bound 推移 | 遅さ・未収束の切り分け |

LLM エージェントで言うと、`invoke_agent` / `chat` / `execute_tool` に相当するのが、ここでは `build_model` / `optimize` / SCIP イベント、という対応関係になる。

```text
solve_model
  ├─ build_model          # 変数・制約を組み立てる
  ├─ optimize             # SCIP が探索する（ここに event がぶら下がる）
  │    ├─ scip.best_sol_found
  │    ├─ scip.lp_solved
  │    ├─ scip.node_branched
  │    └─ scip.node_focused（間引き）
  └─ collect_stats        # 終了後のサマリ
```

単なる `print(status)` ではなく、**トレースの木構造**として残したい。

## アーキテクチャ

```text
databricks bundle run run_scip_optimizer
        │
        ▼
Lakeflow Job（Serverless notebook + PySCIPOpt）
  build_model / optimize / collect_stats
    ├─ SCIP Eventhdlr → OpenTelemetry span events
    └─ OTel SDK（InMemory collector）
           │
           ▼ Spark append
     otel_zerobus.scip_otel.scip_spans
           │
           ▼
     analyze_scip_traces（SQL）
```

前回のエージェント記事で Zerobus（OTLP ネイティブ）も試したが、今回は **Spark/Delta sink** に寄せた。理由は単純で、

- ワークスペースが Serverless のみでも動く
- default storage の UC でも書ける
- 「計装 → Lakehouse → SQL」の本筋を短く見せられる

Zerobus 経路が必要なら、前回記事のカスタム S3 + MANAGED LOCATION 付きカタログの手順がそのまま使える。

## サンプル問題

既定で解くのは `production_planning`（ロットサイズ風の多期間生産計画）。

- 製品 A/B/C、期間 6
- 変数: 生産量（連続）・在庫（連続）・段取り（0-1）
- 目的: 生産費 + 段取り費 + 在庫費の最小化
- 制約: 需要バランス、段取り連動（Big-M）、各期能力 120

SCIP に分岐と LP をやらせるためのデモ用 MIP。別問題として 0-1 ナップサックも用意してある。

今回の実行結果:

| 項目 | 値 |
|------|-----|
| status | `optimal` |
| objective | `3255.5` |
| spans_written | `4` |
| table | `otel_zerobus.scip_otel.scip_spans` |
| trace_id | `734ebc791a58423069c8975b0bb78b75` |

## 計装のポイント

### 1. span でフェーズを切る

`solve_model` をルートに、構築・求解・統計回収を子 span にした。属性は `optimization.*` 名前空間に寄せている。

例:

- `optimization.solver.name = scip`
- `optimization.problem.name = production_planning`
- `optimization.operation = optimize`
- `optimization.solution.status`
- `optimization.solution.objective`
- `optimization.mip.gap`
- `optimization.mip.nodes`
- `optimization.mip.lp_iterations`

GenAI semconv のような公式 semconv が MIP にはまだ薄いので、自前の一貫したキーにした。ここが「OTel を使う意味」でもある。送信先が Databricks でも、属性設計は SDK 側で完結する。

### 2. SCIP Eventhdlr で途中経過を event にする

PySCIPOpt の `Eventhdlr` で、次を `optimize` span の event として付与した。

| SCIP イベント | OTel event |
|---------------|------------|
| `BESTSOLFOUND` | `scip.best_sol_found` |
| `SOLFOUND` | `scip.sol_found` |
| `FIRSTLPSOLVED` | `scip.first_lp_solved` |
| `LPSOLVED` | `scip.lp_solved`（間引き） |
| `NODEBRANCHED` / `NODEINFEASIBLE` | 間引きして記録 |
| `NODEFOCUSED` | `node_event_sample` ごと（既定 50） |

全部を生で出すと UC がイベントログの海になる。高頻度イベントは間引き、最終 span 属性にカウンタ要約を載せる、というバランスにした。

### 3. Spark で UC に落とす

`export_mode=delta` のとき、SimpleSpanProcessor → InMemory collector → Job 終了時に DataFrame append。

テーブルの主な列:

- 共通: `trace_id`, `span_id`, `parent_span_id`, `name`, `duration_ms`, `status_code`
- SCIP 向け: `problem_name`, `solution_status`, `objective_value`, `mip_gap`, `n_vars`, `n_constraints`, `n_nodes`, `n_lp_iterations`
- 詳細全部: `attributes_json`

「よく見る列はフラット、残りは JSON」は、前回の GenAI spans テーブルと同じ発想。

## DABs で用意した Job

| キー | 役割 |
|------|------|
| `setup_scip_otel_tables` | `{prefix}_spans` を UC に作成 |
| `run_scip_optimizer` | SCIP 求解 & OTel → UC |
| `analyze_scip_traces` | gap / nodes / LP / 遅延の SQL 分析 |

ワークスペース制約で classic cluster が使えなかったため、Job は **Serverless environment**。`pyscipopt` は environment dependencies で入れた。ネイティブバイナリ付きの wheel が通るかどうかは環境依存なので、ここは最初につまづきやすい点。

## 動かした手順

```powershell
databricks auth login --profile taka
databricks bundle validate -t dev --profile taka
databricks bundle deploy -t dev --profile taka

databricks bundle run -t dev --profile taka setup_scip_otel_tables
databricks bundle run -t dev --profile taka run_scip_optimizer
databricks bundle run -t dev --profile taka analyze_scip_traces `
  --params trace_id=734ebc791a58423069c8975b0bb78b75
```

分析側では、例えば次のような集計が見られる。

```sql
SELECT
  problem_name,
  COUNT(*) AS solves,
  AVG(duration_ms) AS avg_duration_ms,
  AVG(mip_gap) AS avg_gap,
  AVG(n_nodes) AS avg_nodes,
  AVG(n_lp_iterations) AS avg_lp_iters
FROM otel_zerobus.scip_otel.scip_spans
WHERE opt_operation = 'optimize'
GROUP BY problem_name;
```

データが 1 問題・1 回だけだと、Query Profile に `REDUNDANT_AGGREGATION` が出ることがある。これは「`GROUP BY problem_name` してもグループが 1 つしかない」という健全な指摘で、結果が壊れているわけではない。問題を変えて何度か回せば、この集計は本来の意味を持つ。

## 前回（AIエージェント）との対応

| AI エージェント記事 | 今回（SCIP） |
|---------------------|--------------|
| `invoke_agent` / `chat` / `execute_tool` | `solve_model` / `build_model` / `optimize` |
| GenAI semconv | `optimization.*` 自前属性 |
| トークン・ツール失敗 | gap・ノード・LP・暫定解更新 |
| Zerobus or Spark sink | 今回は Spark sink |
| `agent_genai_spans` | `scip_spans` |

同じ「OTel で計装 → Lakehouse に一次テレメトリ → SQL」の型を、ドメインだけ入れ替えた、というのが一番短い説明。

## わかったこと / 注意点

1. **ソルバの内部計算は「ログ文字列」より span event の方が後から追いやすい**  
   best sol の更新時刻と目的値を、同じ `trace_id` で `optimize` にぶら下げられる。

2. **高頻度イベントは必ず間引く**  
   `NODEFOCUSED` を全部残すと、観測データ自体が求解より重くなる。

3. **Serverless × ネイティブソルバは依存関係が本丸**  
   計装より先に `pyscipopt` が入るかで勝負が決まる。今回は environment dependencies で通った。

4. **OTel の強みはバックエンド非依存**  
   今回は UC Delta に落としたが、同じ計装のまま Collector / Zerobus / 別 APM にも向けられる。

## まとめ

SCIP の MIP 求解を OpenTelemetry で計装し、探索の途中経過ごと Databricks Unity Catalog に残すところまでを DABs で回した。

AI エージェントのトレースと同じ道具で、**最適化ソルバの「どう解いたか」** も Lakehouse の観測対象にできる、というのが今回の主眼。次にやるなら、パラメータ変更の前後比較や、gap が閉まらない実行のアラート（前回の ERROR span アラートの最適化版）が自然な続きになる。

## 参考

- [PySCIPOpt](https://github.com/scipopt/PySCIPOpt)
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/)
- [Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Databricks SQL Performance Insights: REDUNDANT_AGGREGATION](https://docs.databricks.com/sql/user/queries/performance-insights#redundant_aggregation)

---
title: "AIエージェントをOpenTelemetryで計装し、DatabricksのLakehouseにトレースを残してみた"
emoji: "🔭"
type: "tech"
topics:
  - "opentelemetry"
  - "databricks"
  - "unitycatalog"
  - "observability"
  - "llm"
published: true
published_at: "2026-08-10 21:20"
---

AI エージェントは「考えて → ツールを呼んで → また考えて → 答える」ため、失敗したときに **どこで詰まったのか** が追いにくい。
今回は倉庫オペレーション用の小さなエージェントを **OpenTelemetry（GenAI semantic conventions）** で計装し、トレースを **Databricks Unity Catalog** に貯めて SQL で見るところまでやった。

デプロイはすべて Declarative Automation Bundles（旧 Databricks Asset Bundles / DABs）で行い、Serverless Job から実行した。

この記事は Zenn コンテスト「OpenTelemetryの知見を、記事にしよう」**OpenTelemetry部門**向けの実践メモ。送信先バックエンドは Databricks だが、計装そのものはベンダー非依存の OTel SDK である。

## 結論だけ先に

- **測る**: OpenTelemetry SDK + GenAI semconv（`invoke_agent` / `chat` / `execute_tool`）
- **動かす**: Databricks Job（DABs）
- **残す**: 2 通り試した
  1. **Zerobus Ingest OTLP** → UC（公式の OTLP 取り込み。ただし **default storage の managed テーブルには書けない**）
  2. **Spark/Delta sink** → UC（スパンをメモリに溜めて Spark で append。S3 追加不要）
- Zerobus を使うなら、**独自 S3 + Storage Credential + External Location + MANAGED LOCATION 付きカタログ**が必要

## やりたかったこと

「注文の状況と在庫と ETA を教えて」のような質問に対して、エージェントがツールを連鎖呼び出しするデモを作る。

```text
質問
  → chat（次の行動を決める）
  → execute_tool lookup_order
  → chat
  → execute_tool lookup_inventory
  → chat
  → execute_tool calculate_eta
  → chat（最終回答）
```

これを単なるログ文字列ではなく、**トレース（trace）とスパン（span）の木構造**として残したい。

用語を短くすると:

| 用語 | 意味 |
|------|------|
| トレース | 1 回のエージェント実行全体 |
| スパン | その中の 1 作業（chat / tool など） |
| GenAI semconv | AI 用の属性名・operation 名の取り決め |

## アーキテクチャ

最終的に次の 2 経路を用意した。

### A. Zerobus 経路（OTLP ネイティブ）

```text
DABs Job
  → Agent（OTel 計装）
  → OTLP/gRPC
  → Zerobus Ingest
  → Unity Catalog Delta（カスタム S3 上の managed table）
  → SQL で分析
```

### B. Spark/Delta sink 経路（S3 追加なし）

```text
DABs Job
  → Agent（OTel 計装）
  → InMemory collector
  → Spark DataFrame.append
  → Unity Catalog Delta（default storage でも可）
  → SQL で分析
```

ポイントは **「測る」と「残す」を分けたこと**。
エージェント本体は常に OTel SDK で計装し、sink だけ差し替える。

Databricks と OpenTelemetry を「公式コネクタ一発」で繋いだのではなく、

1. OTel で測る
2. Job ノートブックで sink に渡す

という橋渡しになっている。

## プロジェクト構成

```text
opentelemetry_demo/
  databricks.yml
  resources/
    run_warehouse_agent.job.yml
    setup_otel_tables.job.yml
    analyze_otel_traces.job.yml
  src/
    warehouse_agent/
      agent.py          # invoke_agent
      llm.py            # chat + tool calling
      tools.py          # execute_tool
      telemetry.py      # TracerProvider / export_mode
      delta_sink.py     # Spark 書き込み
      zerobus.py        # Zerobus OAuth + OTLP exporter
    notebooks/
      run_agent.py
      setup_otel_tables.py
      analyze_traces.py
```

`export_mode` は Job パラメータで切り替える。

| 値 | 用途 |
|----|------|
| `zerobus` | OTLP → Zerobus → UC |
| `delta` | メモリ → Spark → UC |
| `console` | ローカル確認 |
| `collector` | 外部 OTLP Collector |

## 計装（ここが OpenTelemetry）

ルートは `invoke_agent`。中で turn を回し、各 turn で `chat` と必要なら `execute_tool` を子スパンにする。

```python
with self.tracer.start_as_current_span(
    f"invoke_agent {self.agent_name}",
    kind=trace.SpanKind.INTERNAL,
) as root:
    root.set_attribute("gen_ai.operation.name", "invoke_agent")
    root.set_attribute("gen_ai.agent.name", self.agent_name)
    # ...
    result = self.llm.chat(messages, self.tracer)  # chat スパン
    if result.message.tool_calls:
        for call in result.message.tool_calls:
            execute_tool(...)  # execute_tool スパン
```

LLM 側ではトークンも属性に載せる。

```python
span.set_attribute("gen_ai.operation.name", "chat")
span.set_attribute("gen_ai.provider.name", self.provider)
span.set_attribute("gen_ai.request.model", self.model)
span.set_attribute("gen_ai.usage.input_tokens", result.input_tokens)
span.set_attribute("gen_ai.usage.output_tokens", result.output_tokens)
```

ツール側。

```python
span.set_attribute("gen_ai.operation.name", "execute_tool")
span.set_attribute("gen_ai.tool.name", name)
span.set_attribute("gen_ai.tool.call.arguments", arguments_json)
```

これで「エージェント全体 / LLM 呼び出し / ツール実行」が同じ `trace_id` でつながる。

LLM は `mock` / `openai` / `databricks`（Model Serving）を切り替え可能にした。記事再現や CI では `mock` で十分。

## DABs で載せる

`databricks.yml` で catalog / schema / Zerobus 用の workspace_id・region を variables にした。

Job はノートブックタスク + serverless environment。依存パッケージは Job の `environments.spec.dependencies` に列挙し、自前パッケージは `sys.path` に `src` を足して import した（`--editable ${workspace.file_path}` だけでは Serverless で `ModuleNotFoundError` になった）。

```powershell
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run -t dev run_warehouse_agent
```

## ハマりどころ①: Zerobus は default storage に書けない

最初は Zerobus に直接送り、UC の managed テーブルへ落とそうとした。

するとこうなった。

```text
Unsupported table kind.
Tables created in default storage are not supported.
Error Code: 4024
```

公式の Zerobus 制限どおり、**metastore の default storage 上の managed テーブルには書けない**。

回避には次が必要だった。

1. ワークスペースと同じリージョン（今回は `us-west-2`）の **S3 バケット**
2. Databricks が Assume できる **IAM Role**
3. **Storage Credential**
4. **External Location**
5. その上に `MANAGED LOCATION` 付きカタログ

```sql
CREATE CATALOG otel_zerobus
  MANAGED LOCATION 's3://nextgen-databricks/uc/otel_zerobus';
```

IAM の trust は二段構え。

1. まず ExternalId=`0000` で Role 作成
2. Databricks で Storage Credential を作って本物の ExternalId を取得
3. trust を更新（Databricks UC Master Role + **self-assuming**）

ここまでやって初めて、OTLP/gRPC で Zerobus へ送ったスパンが UC に着地した。

```text
[otel] zerobus-otlp exported 1 span(s)
...
otel_zerobus.otel_demo.agent_otel_spans → 12 spans / 1 trace
```

Zerobus 用テーブルは公式の OTLP スキーマ（`otel.schemaVersion = v2`）で作成し、Service Principal には `USE CATALOG` / `USE SCHEMA` / `SELECT,MODIFY` を明示付与した（`ALL PRIVILEGES` だけでは足りない）。

## ハマりどころ②: S3 がすぐ用意できないとき

コンテスト準備中、最初は S3 が無く Zerobus を諦めかけた。
そのとき作ったのが **Spark/Delta sink**。

流れは単純。

1. `export_mode=delta` で InMemory collector にスパンを溜める
2. エージェント終了後、`ReadableSpan` を行に変換
3. `spark.createDataFrame(...).write.mode("append").saveAsTable(...)`

分析しやすいように、公式 OTLP フルスキーマではなく、デモ用の薄い表にした。

```sql
CREATE TABLE IF NOT EXISTS catalog.schema.agent_genai_spans (
  ingested_at TIMESTAMP,
  service_name STRING,
  trace_id STRING,
  span_id STRING,
  parent_span_id STRING,
  name STRING,
  kind STRING,
  status_code STRING,
  duration_ms DOUBLE,
  gen_ai_operation STRING,
  gen_ai_provider STRING,
  gen_ai_model STRING,
  gen_ai_tool_name STRING,
  input_tokens BIGINT,
  output_tokens BIGINT,
  attributes_json STRING
)
USING DELTA
```

この経路なら **追加 S3 なし**で、同じ OTel 計装の結果を Lakehouse に残せる。
「OTel で測る」ことと「Zerobus で運ぶ」ことを分離できたのが大きい。

## 実際に見えたもの

Job 実行後、SQL でこう見える（Spark sink 側の例）。

| name | gen_ai_operation | tool | input_tokens | output_tokens |
|------|------------------|------|--------------|---------------|
| invoke_agent ... | invoke_agent | | | |
| chat mock-warehouse-llm | chat | | 120 | 40 |
| execute_tool lookup_order | execute_tool | lookup_order | | |
| chat ... | chat | | 160 | 30 |
| execute_tool lookup_inventory | execute_tool | lookup_inventory | | |
| chat ... | chat | | 180 | 35 |
| execute_tool calculate_eta | execute_tool | calculate_eta | | |
| chat ... | chat | | 220 | 90 |

ここから分かること:

1. **1 回答が LLM 4 回 + ツール 3 回**でできている
2. ターンが進むと input tokens が増える（会話履歴が載る）
3. ツール自体の latency は小さい（mock のため）。本番 LLM なら chat スパンが支配的になるはず
4. 同じ `trace_id` で親子関係を辿れる

Zerobus 経路でも同様に `agent_otel_spans` へ着地し、`status.code = STATUS_CODE_OK` のスパンが並んだ。

## SQL で見る（運用イメージ）

```sql
-- 最近のエージェント実行
SELECT time, trace_id, name, status.code
FROM otel_zerobus.otel_demo.agent_otel_spans
WHERE name LIKE 'invoke_agent%'
ORDER BY time DESC
LIMIT 50;

-- ある trace を展開
SELECT name, kind, parent_span_id, span_id,
       CAST((end_time_unix_nano - start_time_unix_nano)/1e6 AS DOUBLE) AS duration_ms
FROM otel_zerobus.otel_demo.agent_otel_spans
WHERE trace_id = '<trace_id>'
ORDER BY start_time_unix_nano;
```

Spark sink 側なら `gen_ai_operation` / `input_tokens` 列が最初から分かれているので、集計がさらに簡単。

```sql
SELECT gen_ai_model, COUNT(*) AS chats,
       SUM(input_tokens) AS input_tokens,
       SUM(output_tokens) AS output_tokens,
       AVG(duration_ms) AS avg_ms
FROM catalog.schema.agent_genai_spans
WHERE gen_ai_operation = 'chat'
GROUP BY 1;
```

## 学び

1. **AI エージェントの可観測性は GenAI semconv が効く**  
   `invoke_agent` / `chat` / `execute_tool` を揃えるだけで、後から SQL しやすい。

2. **Databricks × OTel は「計装」と「搬送」を分けて考えると楽**  
   Zerobus は強力だがストレージ前提がある。Spark sink は泥臭いが再現性が高い。

3. **Zerobus Error 4024 は設計制約**  
   default storage の managed table では無理。独自バケットと UC managed location が必要。

4. **DABs + Serverless では依存解決に注意**  
   editable install が効かないケースがあった。`sys.path` と明示 dependencies が安定した。

5. **SP の GRANT は明示が必要**  
   Zerobus 書き込みは `USE CATALOG` / `USE SCHEMA` / `SELECT,MODIFY` が揃って初めて通る。

## まとめ

OpenTelemetry で AI エージェントを計装し、Databricks 上で動かして Lakehouse にトレースを残す、という一連を DABs で再現した。

- コンテスト的には「OTel で測った一次情報を、自分の環境に残して SQL で見た」体験が本体
- Zerobus は公式 OTLP バックエンドとして魅力的だが、**ストレージ前提を先に読むこと**
- すぐ試すなら `export_mode=delta`、本番寄りの OTLP 搬送なら Zerobus

続編: [OpenTelemetryでAIエージェントのLLM比較と失敗スパンアラートまでやってみた](https://zenn.dev/babysteps/articles/databricks-otel-llm-compare-alert)

:::message
生成 AI も活用しつつ、手順・エラー・着地確認は実際のワークスペースで検証した内容です。
:::

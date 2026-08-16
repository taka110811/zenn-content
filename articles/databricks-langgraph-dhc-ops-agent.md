---
title: "予測も最適化も呼ばない — LangGraph で作る地冷プラント運転支援エージェント"
emoji: "🏭"
type: "tech"
topics:
  - "databricks"
  - "langgraph"
  - "agent"
  - "unitycatalog"
  - "mlflow"
published: true
published_at: "2026-08-16 14:36"
---

地域冷暖房（地冷）の当直が欲しいのは、新しい予測モデルでもソルバーでもない。**今の数字**と**手順書**を突き合わせて、次の一手を短く出すことだ。

この記事では、Databricks の LangGraph テンプレートを使って「湾岸プラント」の運転支援エージェントを組んだ。需要予測も最適化もアプリ内では動かさない。合成した実績・予測・推奨結果を Unity Catalog から読み、Metric View の KPI と手順書チャンクを根拠に答える。

## 結論（先に）

| レイヤー | 何を担うか |
| --- | --- |
| 対話・ツール選択 | **LangGraph**（`create_agent`）+ Foundation Model（`databricks-gpt-5-2`） |
| 配信 | **MLflow AgentServer**（Responses API）+ **Databricks Apps** |
| 数字の定義 | **Unity Catalog Metric View**（ダッシュボードと同じ KPI） |
| 根拠 | 実績 / 予測 / テレメトリ / 事前計算した推奨 / 手順書チャンク |
| 権限 | **DABs**（`databricks.yml`）でテーブル・Warehouse・LLM をアプリ SP に付与 |

やらないことも先に書く。

- 需要予測モデルは呼び出さない。`demand_forecast` を読むだけ
- 最適化ソルバーは実行しない。`opt_recommendations` を読むだけ
- 数値は推測しない。ツール結果から引用する

エージェントは「推論エンジン」ではなく、**運転 copilot** である。

## なぜこの切り方か

プラント運転のデモでよくある失敗は、エージェントに予測や最適化までやらせることだ。当直の現場では、モデルはすでに別系統で動いており、欲しいのは次の 3 点だけになる。

1. **今の状態** — 外気、冷却水、COP、蓄熱残量、受電電力
2. **契約との余裕** — 8,500 kW に対して何 kW 残っているか
3. **手順書との照合** — ターボを足してよいか、蓄熱を先に使うべきか

数字の定義をエージェント側で割り算し直すと、ダッシュボードと答えがズレる。だから KPI は Metric View の `MEASURE(...)` を優先する。

## デモの物語

基準時刻は **2026-08-14 14:00 JST**（猛暑ピーク）。エージェントにとっての「今」は、この時刻に固定する。

```text
猛暑で冷熱実績が予測を上回る
  → 冷却水温度が上がり、ターボの COP が落ちる
  → 契約電力 8,500 kW の余裕が 200 kW を切る（DHC-E-2101）
  → 手順書は「ターボ追加起動より蓄熱放熱と吸収式」（DHC-E-320）
```

合成データの仕込みは次のとおり。

| 仕掛け | 中身 |
| --- | --- |
| 気象 | デモ日 11〜17 時だけ、実績気温を予報より高くする |
| 需要 | ホテル海風を 12〜17 時に 1.28 倍。イベント日の外れを再現 |
| 受電 | 14:00 ちょうどを 8,320 kW に固定（余裕 180 kW） |
| 推奨 | 13〜16 時は「蓄熱放熱を最大し、ターボ追加起動を行わない」 |

需要家は湾岸タワーA / B、ホテル海風、駅前商業施設の 4 件。熱源はターボ 2 台、吸収式、ジェネリンク、蓄熱槽。

## アーキテクチャ

```text
ブラウザ（Databricks Apps / ローカル :8000）
    │
    ▼
MLflow AgentServer（Responses API + chat proxy）
    │
    ▼
LangGraph create_agent
    │  ChatDatabricks（databricks-gpt-5-2）
    │
    ├─ get_kpis                     Metric View
    ├─ get_plant_status             plant_telemetry
    ├─ compare_demand_and_forecast  demand_actual / demand_forecast
    ├─ check_contract_power         contract_power
    ├─ get_optimization_recommendation  opt_recommendations
    └─ search_operating_procedures  sop_chunks
         │
         ▼
    SQL Warehouse（Statement Execution API）
         │
         ▼
    Unity Catalog  fs_demo.dhc_ops_demo
```

ポイントは 3 つ。

- **「今」は環境変数 `DHC_DEMO_TS`**。デモを再現可能にするため、システム時刻には依存しない
- **SQL はツールの中で書く**。モデルに SQL を生成させない
- **権限はアプリのサービスプリンシパル**。DABs の `uc_securable` で SELECT を渡す

## データと Metric View

カタログは `fs_demo.dhc_ops_demo`。テーブルは合成 CSV を Volume に上げ、`read_files` で `CREATE OR REPLACE TABLE` する。

| テーブル | 役割 |
| --- | --- |
| `plants` / `equipment` / `customers` | マスタ |
| `weather` | 外気の実績と予報 |
| `demand_actual` / `demand_forecast` | 冷熱 RT の実績と予測 |
| `plant_telemetry` | 30 分スナップショット（COP、蓄熱 SOC、受電） |
| `contract_power` | 契約 8,500 kW、警報余裕 200 kW |
| `opt_recommendations` | 事前計算した運転推奨 |
| `sop_chunks` | 手順書の条文チャンク |

KPI はテーブルを直接集計せず、Metric View に置く。予測誤差率は「行ごとの平均」ではなく **SUM の比** にする。ここをエージェント側で再計算すると、ダッシュボードとズレる。

```sql
CREATE OR REPLACE VIEW fs_demo.dhc_ops_demo.demand_kpis
WITH METRICS
LANGUAGE YAML
AS $$
  version: 1.1
  source: fs_demo.dhc_ops_demo.demand_actual
  joins:
    - name: forecast
      source: fs_demo.dhc_ops_demo.demand_forecast
      on: "source.ts = forecast.ts AND source.plant_id = forecast.plant_id AND source.customer_id = forecast.customer_id"
  measures:
    - name: forecast_error_pct
      expr: "100 * (SUM(cooling_rt) - SUM(forecast.cooling_rt)) / NULLIF(SUM(forecast.cooling_rt), 0)"
$$
```

`plant_ops_kpis` 側では、契約余裕を `MIN(contract_kw - received_power_kw)`、契約ステータスを dimension として定義する。200 kW 未満は `ALERT`、超過は `BREACH`。

エージェントの `get_kpis` は、この View を `MEASURE(...)` で読む。

```sql
SELECT
  MEASURE(received_power_kw) AS received_power_kw,
  MEASURE(contract_margin_kw) AS contract_margin_kw,
  ROUND(MEASURE(turbo_cop), 2) AS turbo_cop,
  MEASURE(storage_soc_pct) AS storage_soc_pct
FROM fs_demo.dhc_ops_demo.plant_ops_kpis
WHERE plant_id = 'PL-WANGAN'
  AND ts = TIMESTAMP('2026-08-14 14:00:00')
GROUP BY ALL
```

## エージェント実装の要点

本体は `agent_server/agent.py`。LangGraph の `create_agent` に 6 ツールを渡し、MLflow の `@invoke` / `@stream` で Responses API に載せる。

### システムプロンプト

役割と禁止事項を先に固定する。

```python
SYSTEM_PROMPT = """あなたは湾岸地域冷暖房プラントの運転支援エージェントです。
対象プラントは「湾岸プラント」（plant_id=PL-WANGAN）です。

ルール:
- 数値は必ずツール結果から引用する。推測で作らない。
- KPI は get_kpis を優先する。自分で割り算し直さない。
- 「今」「現在」はデモ基準時刻 {demo_ts}（JST）を指す。
- 回答は日本語。先に結論、次に根拠（時刻・数値・出典）。
- 手順書を使ったら、文書名と条文番号（DHC-E-xxxx など）を書く。
- 需要予測モデルや最適化ソルバーは呼び出さない。結果テーブルを読むだけ。
""".format(demo_ts=demo_as_of())
```

### ツールは SQL を内包する

`get_kpis` 以外は、テレメトリや手順書を直接読む。`as_of` が空ならデモ基準時刻にフォールバックする。

| ツール | 見ているもの |
| --- | --- |
| `get_kpis` | `plant_ops_kpis` / `demand_kpis` |
| `get_plant_status` | 外気・冷却水・COP・蓄熱・受電 |
| `compare_demand_and_forecast` | 冷熱の実績 vs 予測（顧客別） |
| `check_contract_power` | 契約 8,500 kW との余裕 |
| `get_optimization_recommendation` | 事前計算した運転推奨 |
| `search_operating_procedures` | 手順書チャンク（LIKE 検索） |

手順書検索は Vector Search ではなく、まずは条文をチャンク化したテーブルの部分一致にした。ヒットしなければ目次を返す。デモで聞きたい条文は次の 3 つに寄せてある。

| コード | 意味 |
| --- | --- |
| `DHC-E-2101` | 契約余裕 200 kW 未満のデマンド警報 |
| `DHC-E-320` | 冷却水 32℃超でのターボ追加起動禁止 |
| `DHC-F-015` | 予測誤差 15% 以上の手動補正対象 |

### サーバ

`AgentServer("ResponsesAgent", enable_chat_proxy=True)` が FastAPI アプリになる。ローカルも Apps も同じエントリポイント。

```python
from mlflow.genai.agent_server import AgentServer

agent_server = AgentServer("ResponsesAgent", enable_chat_proxy=True)
app = agent_server.app
```

トレースは `mlflow.langchain.autolog()` と、セッション ID を `mlflow.update_current_trace` に載せる。

## 権限は databricks.yml に書く

ツールを足したら、同じ PR で権限も足す。アプリのサービスプリンシパルがカタログを辿れないと、ローカルでは動いても Apps で落ちる。

```yaml
resources:
  apps:
    agent_langgraph:
      name: agent-dhc-ops
      resources:
        - name: warehouse
          sql_warehouse:
            id: ${var.warehouse_id}
            permission: CAN_USE
        - name: llm
          serving_endpoint:
            name: ${var.llm_endpoint}
            permission: CAN_QUERY
        - name: plant_ops_kpis
          uc_securable:
            securable_full_name: ${var.catalog}.${var.schema}.plant_ops_kpis
            securable_type: TABLE
            permission: SELECT
```

テーブルごとに `uc_securable` を列挙する。Metric View も TABLE として SELECT を付ける。カタログ / スキーマの `USE` が足りない場合は、デプロイ後に追加 GRANT する。

```sql
GRANT USE CATALOG ON CATALOG fs_demo TO `<app-sp>`;
GRANT USE SCHEMA ON SCHEMA fs_demo.dhc_ops_demo TO `<app-sp>`;
```

## ハンズオン手順（最短）

### 0. 前提

- Unity Catalog 有効なワークスペース
- SQL Warehouse
- Foundation Model エンドポイント（例: `databricks-gpt-5-2`）
- ローカルに Databricks CLI と `uv`

### 1. セットアップ

```powershell
copy .env.example .env
uv sync
```

`.env` にプロファイル、Warehouse ID、カタログ / スキーマ、デモ時刻、MLflow 実験 ID を入れる。実験が無ければ先に作る。

```powershell
databricks experiments create-experiment /Users/<you>/agent-dhc-ops --profile <profile>
```

返ってきた ID を `.env` と `databricks.yml` の `experiment_id` に入れる。

### 2. デモデータを載せる

```powershell
uv run generate-demo-data
uv run create-metric-views
```

`generate-demo-data` は 2026-05-17 〜 08-16 の 30 分値を作り、Volume 経由でテーブル化する。`sop_chunks` には Change Data Feed を付けてあり、後から Vector Search に載せ替えやすい。

### 3. ローカル起動

```powershell
uv run start-app
```

ブラウザは http://localhost:8000 。

### 4. 試し聞き

基準時刻は 2026-08-14 14:00。次の順で聞くと、ツールの使い分けが見える。

1. 今の KPI を教えて。予測誤差と契約電力の余裕は？
2. 今、湾岸プラントはどんな状態？
3. 冷熱需要は予測をどれだけ上回っている？どの需要家が効いている？
4. 契約電力の超過リスクは？警報コードは何？
5. このとき蓄熱槽はどう使う？ターボを足してよいか？
6. 次の一手を、数字と手順書の出典つきでまとめて。

期待するキーワードは次のとおり。

- 予測誤差 15% 前後、ホテル海風
- 契約 8,500 kW、余裕 200 kW 未満、`DHC-E-2101`
- 冷却水 32℃超のターボ追加起動禁止 `DHC-E-320`
- 推奨は蓄熱放熱＋吸収式増負荷

### 5. デプロイ

```powershell
databricks bundle validate --profile <profile>
databricks bundle deploy --profile <profile>
databricks bundle run agent_langgraph --profile <profile>
```

アプリ名は `agent-dhc-ops`。`bundle run` はデプロイ済みアプリの起動なので、コードや YAML を変えたら毎回 `deploy` してから `run` する。

## 評価

当直と当直長の 2 ペルソナで、MLflow の `ConversationSimulator` を回す。

```python
test_cases = [
    {
        "goal": "今の湾岸プラントの状態と、契約電力の余裕を把握する",
        "persona": "当直運転員。数字はツールで確認したい。",
    },
    {
        "goal": "冷熱が予測を外れている理由と、蓄熱槽の使い方を手順書付きで知る",
        "persona": "当直長。出典のない助言は信じない。",
    },
]
```

ローカルサーバ起動後に `uv run agent-evaluate`。Scorer は Completeness / Safety / ToolCallCorrectness など、MLflow 組み込みを使う。

## 設計上の注意

1. **エージェントにモデルを再実行させない**  
   予測と最適化は結果テーブルに閉じる。デモが壊れにくく、本番の責任分界にも近い。

2. **KPI は Metric View を単一の定義源にする**  
   誤差率や契約余裕をツールごとに計算し直すと、ダッシュボードと会話が食い違う。

3. **「今」を固定する**  
   運転デモは時刻で物語が決まる。`DHC_DEMO_TS` を環境変数にして、ローカルと Apps で同じスナップショットを見る。

4. **SQL はツール実装に閉じる**  
   モデルに SQL を書かせると、権限と再現性が崩れる。パラメータは時刻と検索語だけにする。

5. **権限漏れは Apps で初めて顕在化する**  
   ローカルは自分のプロファイル、Apps はサービスプリンシパル。`databricks.yml` の `resources` と `USE CATALOG` / `USE SCHEMA` をセットで見る。

6. **手順書は最初から Vector Search にしなくてよい**  
   条文数が少なければチャンクテーブル + LIKE で足りる。CDF を付けておけば、後からインデックスに載せ替えられる。

## まとめ

- 地冷の運転支援は、予測・最適化の再実行より **数字と手順書の突合** が先
- LangGraph のツールは SQL Warehouse 経由で UC を読み、KPI は Metric View に寄せる
- 配信は MLflow AgentServer + Databricks Apps、権限は DABs でアプリ SP に渡す
- 猛暑ピークの物語（契約余裕 200 kW 未満、ターボ追加起動禁止）をデータ側に仕込むと、試し聞きが安定する

## 参考

- [Databricks Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
- [MLflow Agent Server](https://mlflow.org/docs/latest/genai/serving/agent-server/)
- [Unity Catalog metric views](https://docs.databricks.com/aws/en/metric-views/)
- [Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)

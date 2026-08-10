---
title: "OpenTelemetryでAIエージェントのLLM比較と失敗スパンアラートまでやってみた"
emoji: "📡"
type: "tech"
topics:
  - "opentelemetry"
  - "databricks"
  - "observability"
  - "llm"
  - "modelserving"
published: true
published_at: "2026-08-10 22:00"
---

前回の記事（[AIエージェントをOpenTelemetryで計装し、DatabricksのLakehouseにトレースを残してみた](https://zenn.dev/babysteps/articles/databricks-opentelemetry-ai-agent-dabs)）では、倉庫オペレーション用エージェントを OpenTelemetry（GenAI semconv）で計装し、Unity Catalog にスパンを残すところまでやった。

ただ、そこまではほぼ **mock LLM** だった。mock だと「動いた」は見えるが、**コスト（トークン）とレイテンシの本番感**は隠れる。また、ERROR スパンが UC に残っても、人が SQL を見に行かない限り気づけない。

今回はその続きとして、

1. **同じ質問で mock と Databricks Model Serving を並べて比較**
2. **失敗スパンを検知して Job を落とす（アラート経路）**

までを DABs で回した。OpenTelemetry コンテストの延長線上だが、焦点は「測ったあとの運用」側。

## 結論だけ先に

- **比較**: 同一クエリを `mock` / `databricks`（Model Serving）で実行し、turns・tool_calls・tokens・latency を 1 行に要約して UC へ
- **アラート**: ERROR スパンをスキャン → `agent_span_alerts` に記録 → `fail_on_alert=true` なら Job を意図的に失敗させる
- **本番シグナル**: turns/tool_calls は一致（公平な A/B）。差は **レイテンシと実トークン**（mock ~15ms vs Model Serving ~4s）
- **認証**: secret `databricks_token` が無くても、Serverless Job の `apiToken` フォールバックで Model Serving 呼び出しが通った

## 今回つくったもの

前回の `agent_genai_spans` に加え、比較用とアラート用の表と Job を足した。

| リソース | 役割 |
|----------|------|
| Job `compare_llm_providers` | 同一クエリを複数 LLM で実行し比較行を書く |
| Job `alert_error_spans` | 直近の ERROR スパンをスキャンし、必要なら Job 失敗 |
| テーブル `agent_genai_spans` | 生スパン（前回から継続） |
| テーブル `agent_llm_comparisons` | プロバイダ横断の要約行 |
| テーブル `agent_span_alerts` | アラート発火の記録 |

カタログ／スキーマは前回と同じ `otel_zerobus.otel_demo`。

## アーキテクチャ

```text
# LLM 比較
DABs Job compare_llm_providers
  → 同一質問（ORD-7788 / SKU / ETA）
  → mock 実行（OTel → agent_genai_spans）
  → databricks Model Serving 実行（同上）
  → スパン要約 → agent_llm_comparisons（1 comparison_id に複数行）

# 失敗アラート
DABs Job alert_error_spans (seed_failure=true)
  → SIMULATE_FAILURE=tool で execute_tool を ERROR に
  → agent_genai_spans をスキャン
  → agent_span_alerts に追記
  → fail_on_alert=true なら raise RuntimeError（Job FAIL）
```

「測る」は引き続き OTel SDK。「比較する」「気づかせる」は UC + Job の状態で表現する。

## LLM 比較の実装ポイント

エージェント本体は前回と同じ。比較 Job 側でプロバイダを切り替えて回し、各実行のスパンから指標を抜く。

トークンは GenAI 属性から取る（前回どおり）。

```python
span.set_attribute("gen_ai.usage.input_tokens", result.input_tokens)
span.set_attribute("gen_ai.usage.output_tokens", result.output_tokens)
span.set_attribute("gen_ai.provider.name", self.provider)
span.set_attribute("gen_ai.request.model", self.model)
```

比較行は「1 実行 = 1 行」に畳む。

```python
# 擬似コード: in-memory OTel spans を 1 実行ぶん要約
summary = {
    "comparison_id": comparison_id,
    "provider": provider,
    "model": model,
    "status": "OK" if no_error else "ERROR",
    "turns": agent_turns,
    "tool_calls": agent_tool_calls,
    "input_tokens": sum(chat_input_tokens),
    "output_tokens": sum(chat_output_tokens),
    "invoke_duration_ms": root_duration_ms,
    "avg_chat_latency_ms": mean(chat_durations),
    "max_chat_latency_ms": max(chat_durations),
    "trace_id": trace_id,
}
# → agent_llm_comparisons へ append
```

ポイントは **同じ質問・同じツールセット・同じループ**でプロバイダだけ変えること。turns / tool_calls が揃えば、トークンと latency の差をフェアに読める。

## 実際の比較結果

`comparison_id = c67f7e08-feb3-49e1-a88a-8c6c6d777d82` の 1 ラン（同じ倉庫クエリ: ORD-7788 / SKU / ETA）。

| provider | model | status | turns | tool_calls | chat_spans | input_tokens | output_tokens | total_tokens | invoke_ms | avg_chat_ms | max_chat_ms |
|----------|-------|--------|------:|-----------:|-----------:|-------------:|--------------:|-------------:|----------:|------------:|------------:|
| mock | mock-warehouse-llm | OK | 4 | 3 | 4 | 680 | 195 | 875 | 14.8 | 0.1 | 0.2 |
| databricks | databricks-meta-llama-3-3-70b-instruct | OK | 4 | 3 | 4 | 4546 | 112 | 4658 | 4069.8 | 1014.5 | 1200.7 |

読み方:

1. **turns / tool_calls が一致**（4 turns, 3 tools）→ エージェントループ自体は同じ道を通っている
2. **mock のトークンは合成固定値**。Model Serving 側は実 usage。input が大きいのはツールスキーマ＋マルチターン履歴が載るため
3. **レイテンシ差が本番シグナル**。mock の invoke ~15ms に対し、Model Serving は ~4s（chat 平均 ~1s、max ~1.2s）
4. 認証は secret `databricks_token` が無くても、ノートブックの `apiToken` フォールバックで Serverless Job から呼べた

trace も残っている（mock: `9670a766...` / databricks: `bc19151d...`）。気になる方は `agent_genai_spans` で親子を展開できる。

## 失敗スパンのアラート

「ERROR が表にある」だけでは運用にならない。今回は **Job 失敗をアラートチャネル**にした。

流れ:

1. `seed_failure=true` で意図的に失敗スパンを仕込む（`SIMULATE_FAILURE=tool` → `execute_tool` を ERROR）
2. 直近（例: 60 分）の `agent_genai_spans` をスキャン
3. ヒットしたら `agent_span_alerts` に書く
4. `fail_on_alert=true` なら `RuntimeError` を投げて Job を FAIL

実際のメッセージ例:

```text
OTel alert: 2 ERROR span(s) in last 60m
(source=otel_zerobus.otel_demo.agent_genai_spans,
 sample_trace_ids=['943b8734f6cf2e397defa573a2294e1b', ...])
```

Job `alert_error_spans` は **意図的に FAILED**。ここがポイントで、「失敗を隠さず、失敗として通知経路に乗せる」。Databricks の Job 失敗通知（メール / webhook 等）と組み合わせれば、追加の監視基盤なしでも最低限の push になる。

## 動かしたコマンド

```powershell
databricks bundle deploy -t dev --profile taka
databricks bundle run -t dev --profile taka compare_llm_providers
databricks bundle run -t dev --profile taka alert_error_spans --params seed_failure=true
```

## SQL で見る

```sql
-- 比較ランを並べる
SELECT provider, model, status, turns, tool_calls,
       input_tokens, output_tokens, total_tokens,
       invoke_duration_ms, avg_chat_latency_ms, max_chat_latency_ms, trace_id
FROM otel_zerobus.otel_demo.agent_llm_comparisons
WHERE comparison_id = 'c67f7e08-feb3-49e1-a88a-8c6c6d777d82'
ORDER BY provider;

-- 最近のアラート
SELECT *
FROM otel_zerobus.otel_demo.agent_span_alerts
ORDER BY detected_at DESC
LIMIT 20;

-- ERROR スパンの元データ
SELECT ingested_at, trace_id, name, status_code, gen_ai_operation, gen_ai_tool_name
FROM otel_zerobus.otel_demo.agent_genai_spans
WHERE status_code = 'ERROR'
ORDER BY ingested_at DESC
LIMIT 50;
```

## 学び

1. **mock だけではコストとレイテンシが読めない**  
   turns が揃って初めて、実 LLM のトークン／latency 差が意味を持つ。

2. **比較表は「要約行」を別テーブルにすると運用しやすい**  
   生スパンは監査・深掘り用、`agent_llm_comparisons` はダッシュボード／A/B 用。

3. **アラートは「UC に書く」＋「Job を落とす」の二段が手堅い**  
   記録は残しつつ、既存の Job 失敗通知を push チャネルに流用できる。

4. **seed 付きアラート Job は検証用に必須**  
   `seed_failure=true` で意図的に FAIL させ、メッセージと `sample_trace_ids` まで確認できた。

5. **Serverless 上の Model Serving 認証はフォールバックを用意しておくと楽**  
   secret 未設定でも `apiToken` で通った。本番では secret 管理に寄せつつ、デモ再現性も確保できる。

## まとめ

前回の「OTel で測って Lakehouse に残す」に続き、**LLM 比較**と **失敗スパンのアラート**まで DABs で回した。

- 同じエージェントループなら、プロバイダ差はトークンと latency に素直に出る
- ERROR スパンは UC に溜めるだけでなく、Job FAIL まで繋ぐと運用の入口になる
- コンテスト的な「計装」から一歩進めて、比較と通知という実務寄りの使い方にした

次の候補:

- Databricks SQL Alerts / webhook への直接通知
- 複数モデルのマトリクス比較（温度・プロンプト差含む）
- 成功率・P95 latency の日次ダッシュボード

:::message
生成 AI も活用しつつ、比較結果・アラートメッセージ・Job 失敗は実際のワークスペースで検証した内容です。
:::

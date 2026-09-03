# Day 6 — Observability Revision Notes

## 1. Prometheus — Core Model
- **Pull-based metrics**: Prometheus scrapes `/metrics` HTTP endpoints on targets at a configured interval (not pushed to it, unlike some other systems). Exception: `Pushgateway` for short-lived batch jobs that can't be scraped.
- **Data model**: Time-series identified by a metric name + key-value labels. e.g. `http_requests_total{method="GET", status="200", service="backend"}`
- **Metric types**:
  - **Counter**: Only goes up (resets on restart) — e.g. total requests served.
  - **Gauge**: Goes up or down — e.g. current memory usage, active connections.
  - **Histogram**: Buckets observations (e.g. request duration) — gives you `_bucket`, `_sum`, `_count` series, lets you calculate percentiles/quantiles.
  - **Summary**: Similar to histogram but calculates quantiles client-side (less flexible for aggregation across instances — histograms are generally preferred).

## 2. PromQL — Building the Chain

### `rate()` vs `irate()`
- **`rate(metric[5m])`**: Average per-second rate of increase over the time window — smooths out spikes, good for alerting/dashboards on trends.
- **`irate(metric[5m])`**: Uses only the last two data points in the window — reacts fast to spikes, but noisier. Good for high-resolution graphs, bad for alerting (too jumpy).
- Both only work correctly on **Counters** (since they measure rate of increase — using them on a Gauge gives meaningless results).

### `histogram_quantile()`
- Used to calculate percentiles (like p95, p99 latency) from histogram bucket data.
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```
- **Breaking this down**:
  1. `http_request_duration_seconds_bucket` — the raw bucketed histogram data (has a `le` label = "less than or equal to" boundary).
  2. `rate(...[5m])` — rate of increase per bucket over 5 min (converts cumulative counts into a per-second rate, needed before quantile math).
  3. `histogram_quantile(0.95, ...)` — interpolates across the bucket boundaries to estimate the 95th percentile value.
- **Interview point**: This is THE standard way to answer "what's our p99 latency" in a Prometheus-based stack — know this formula cold, it's asked constantly.

### Common Alerting Query Pattern
```promql
# Error rate over 5%, sustained
(sum(rate(http_requests_total{status=~"5.."}[5m])) 
  / sum(rate(http_requests_total[5m]))) > 0.05
```

## 3. Grafana
- **Data source layer**: Grafana itself doesn't store metrics — it queries data sources (Prometheus, Loki, Tempo, etc.) and visualizes results.
- **Dashboards**: JSON-defined, made of panels (each panel = one or more PromQL/LogQL/TraceQL queries + visualization type).
- **Alerting**: Grafana can also define alert rules directly (newer unified alerting) that evaluate queries and fire notifications (Slack, PagerDuty, email) — overlaps with Prometheus's own Alertmanager, but centralizes alerting UI across multiple data sources.

## 4. Tempo (Distributed Tracing)
- **What it does**: Stores and queries **traces** — a trace represents one request's full journey across multiple services, made up of **spans** (one span = one unit of work, e.g. one service call or DB query).
- **Why tracing matters beyond metrics/logs**: Metrics tell you *something* is slow (e.g. p99 latency spiked). Logs tell you what happened in one service. Tracing tells you **exactly where in a multi-service request chain** the time was spent — which specific downstream call caused the slowdown.
- **TraceQL**: Tempo's query language for searching traces by span attributes, similar spirit to PromQL/LogQL but for trace data.
- **Correlation**: Modern setups link trace IDs into logs and metrics exemplars, so you can jump from "this request was slow" (metric) → "here's its full trace" (Tempo) → "here's the exact log lines from the slow span" (Loki).

## 5. LGTM Stack — What Each Piece Does
| Letter | Tool | Purpose |
|---|---|---|
| **L** | Loki | Log aggregation — stores and queries logs (LogQL), indexes only labels (not full text) for efficiency |
| **G** | Grafana | Visualization + dashboards + alerting UI layer across everything |
| **T** | Tempo | Distributed tracing — stores and queries traces/spans |
| **M** | Mimir (or Prometheus) | Metrics storage — Mimir is Grafana Labs' horizontally-scalable, long-term Prometheus-compatible metrics store |

- **Why the "L-G-T-M" grouping matters (interview angle)**: It's Grafana Labs' unified observability stack — one vendor, one set of tools, all designed to correlate with each other (trace ID in logs, exemplars linking metrics to traces). Contrast with a fragmented stack where Prometheus + ELK + Jaeger don't naturally cross-reference.

---

## Quick Self-Test (do this without looking)
1. Why does `rate()` work correctly on a Counter but give meaningless results on a Gauge?
2. Write out (or say out loud) the full `histogram_quantile` query to get p99 latency from a metric called `api_latency_seconds_bucket`, explaining each part.
3. What specific problem does distributed tracing (Tempo) solve that metrics and logs alone can't?
4. In the LGTM stack, which letter handles logs, and what does it index differently than something like Elasticsearch?

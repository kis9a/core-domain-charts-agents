# 監視・可観測性（Monitoring & Observability）

## 概要

エージェントシステムの健全性、パフォーマンス、品質を継続的に監視し、問題の早期発見と根本原因分析を可能にします。

## メトリクス設計

### 1. システムメトリクス

```yaml
system_metrics:
  # リソース使用量
  resource_usage:
    cpu_utilization:
      type: gauge
      unit: percent
      labels: [agent, environment]
      alert_threshold: 80

    memory_usage:
      type: gauge
      unit: megabytes
      labels: [agent, environment]
      alert_threshold: 450  # 512MB制限の90%

    api_quota_remaining:
      type: gauge
      unit: requests
      labels: [provider, model]
      alert_threshold: 1000

  # レイテンシ
  latency:
    request_duration:
      type: histogram
      unit: milliseconds
      buckets: [100, 500, 1000, 5000, 10000, 30000]
      labels: [agent, operation, status]

    agent_processing_time:
      type: histogram
      unit: milliseconds
      labels: [agent, module_count]
      percentiles: [0.5, 0.95, 0.99]

  # スループット
  throughput:
    requests_per_second:
      type: counter
      labels: [endpoint, method, status_code]

    modules_processed_per_minute:
      type: counter
      labels: [agent]
```

### 2. ビジネスメトリクス

```yaml
business_metrics:
  # 分析品質
  analysis_quality:
    evidence_count:
      type: histogram
      buckets: [1, 3, 5, 10, 20]
      labels: [module, classification]

    confidence_score:
      type: gauge
      range: [0, 1]
      labels: [module, agent]

    classification_distribution:
      type: counter
      labels: [classification, environment]

  # スコアリング統計
  scoring_stats:
    score_distribution:
      type: histogram
      buckets: [1, 2, 3, 4, 5]
      labels: [score_type, classification]  # score_type: S, Δ, X

    score_adjustments:
      type: counter
      labels: [reason, direction]  # direction: up, down

    default_score_usage:
      type: counter
      labels: [score_type, reason]

  # 精度メトリクス
  accuracy:
    contradiction_detected:
      type: counter
      labels: [type, severity]

    self_correction_rate:
      type: gauge
      labels: [agent]

    validation_failures:
      type: counter
      labels: [validation_type, field]
```

### 3. オペレーショナルメトリクス

```yaml
operational_metrics:
  # エラー追跡
  errors:
    error_rate:
      type: counter
      labels: [error_code, agent, level]

    retry_attempts:
      type: counter
      labels: [error_code, attempt_number]

    recovery_success:
      type: gauge
      labels: [strategy, agent]

  # キャッシュ効率
  cache:
    hit_rate:
      type: gauge
      labels: [cache_type]

    eviction_count:
      type: counter
      labels: [reason]

    size:
      type: gauge
      unit: megabytes

  # 依存サービス
  dependencies:
    availability:
      type: gauge
      labels: [service]

    response_time:
      type: histogram
      labels: [service, operation]

    error_rate:
      type: counter
      labels: [service, error_type]
```

## トレーシング実装

### 1. 分散トレーシング構造

```yaml
trace_structure:
  root_span:
    name: "ddd_mapping_request"
    attributes:
      request_id: "uuid"
      module_count: "integer"
      environment: "string"

  child_spans:
    - name: "analyzer_process"
      parent: "root"
      attributes:
        input_size: "bytes"
        subdomain_count: "integer"

    - name: "scorer_process"
      parent: "root"
      attributes:
        scores_calculated: "integer"
        default_scores_used: "integer"

    - name: "contradiction_check"
      parent: "root"
      attributes:
        contradictions_found: "integer"
        corrections_made: "integer"

    - name: "chart_formatting"
      parent: "root"
      attributes:
        chart_type: "string"
        data_points: "integer"
```

### 2. トレース実装例

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode
import time

tracer = trace.get_tracer(__name__)

class TracedAgent:
    def process(self, input_data: dict) -> dict:
        with tracer.start_as_current_span(
            "agent_process",
            attributes={
                "agent.name": self.__class__.__name__,
                "input.module_count": len(input_data.get("modules", [])),
                "request.id": input_data.get("request_id")
            }
        ) as span:
            try:
                # 入力検証フェーズ
                with tracer.start_span("validation") as validation_span:
                    validation_start = time.time()
                    self.validate_input(input_data)
                    validation_span.set_attribute(
                        "duration_ms",
                        (time.time() - validation_start) * 1000
                    )

                # 処理フェーズ
                with tracer.start_span("processing") as processing_span:
                    result = self.process_internal(input_data)
                    processing_span.set_attribute(
                        "output.items_count",
                        len(result.get("items", []))
                    )

                span.set_status(Status(StatusCode.OK))
                return result

            except Exception as e:
                span.record_exception(e)
                span.set_status(
                    Status(StatusCode.ERROR, str(e))
                )
                raise
```

## ログ集約と分析

### 1. 構造化ログ形式

```json
{
  "timestamp": "2025-01-03T10:00:00.000Z",
  "level": "INFO",
  "logger": "scorer_agent",
  "trace_id": "abc123def456",
  "span_id": "789ghi012",
  "request_id": "req-2025-0001",
  "message": "Scoring completed",
  "attributes": {
    "module": "pricing-service",
    "scores": {"S": 4, "Δ": 3, "X": 2},
    "classification": "Core",
    "evidence_count": 8,
    "processing_time_ms": 234,
    "confidence": 0.85
  },
  "context": {
    "environment": "production",
    "version": "1.0.0",
    "host": "agent-worker-01"
  }
}
```

### 2. ログ集約パイプライン

```yaml
pipeline:
  inputs:
    - type: file
      path: "/var/log/agents/*.log"
      multiline:
        pattern: '^\d{4}-\d{2}-\d{2}'

  processors:
    - type: json_parser
      field: message

    - type: add_fields
      fields:
        environment: "${ENV}"
        hostname: "${HOSTNAME}"

    - type: filter
      condition: 'level in ["ERROR", "WARN", "INFO"]'

  outputs:
    - type: elasticsearch
      hosts: ["es-cluster:9200"]
      index: "agents-%{+YYYY.MM.dd}"

    - type: s3
      bucket: "agent-logs-archive"
      prefix: "year=%{+YYYY}/month=%{+MM}/day=%{+dd}/"
      compression: gzip
```

## ダッシュボード設計

### 1. エグゼクティブダッシュボード

```yaml
executive_dashboard:
  panels:
    - title: "システムヘルス"
      type: traffic_light
      metrics:
        - success_rate
        - avg_latency
        - error_rate

    - title: "分類分布"
      type: pie_chart
      data: classification_distribution

    - title: "処理量トレンド"
      type: time_series
      metrics:
        - requests_per_minute
        - modules_processed

    - title: "品質スコア"
      type: gauge
      metrics:
        - avg_confidence
        - evidence_coverage
```

### 2. オペレーションダッシュボード

```yaml
operations_dashboard:
  panels:
    - title: "エラー率"
      type: time_series
      metrics:
        - error_rate_by_agent
        - retry_rate

    - title: "レイテンシ分布"
      type: heatmap
      data: latency_percentiles

    - title: "リソース使用状況"
      type: multi_line
      metrics:
        - cpu_usage
        - memory_usage
        - api_quota

    - title: "キャッシュ効率"
      type: stat_panel
      metrics:
        - cache_hit_rate
        - cache_size
```

### 3. 分析品質ダッシュボード

```yaml
quality_dashboard:
  panels:
    - title: "スコア分布"
      type: histogram
      data: score_distribution_by_type

    - title: "証拠カバレッジ"
      type: bar_chart
      data: evidence_count_by_module

    - title: "矛盾検出"
      type: table
      columns:
        - module
        - contradiction_type
        - correction_applied

    - title: "信頼度トレンド"
      type: area_chart
      data: confidence_over_time
```

## アラート設定

### 1. 重要度別アラートルール

```yaml
alerts:
  critical:
    - name: "API完全停止"
      condition: "up == 0"
      duration: "1m"
      channels: ["pagerduty", "slack-critical"]

    - name: "メモリ枯渇"
      condition: "memory_usage > 500"
      duration: "2m"
      channels: ["pagerduty", "email"]

  warning:
    - name: "高エラー率"
      condition: "error_rate > 0.05"
      duration: "5m"
      channels: ["slack-alerts", "email"]

    - name: "レイテンシ劣化"
      condition: "p95_latency > 10000"
      duration: "10m"
      channels: ["slack-alerts"]

  info:
    - name: "キャッシュ効率低下"
      condition: "cache_hit_rate < 0.5"
      duration: "15m"
      channels: ["slack-monitoring"]
```

### 2. アラート通知テンプレート

```markdown
## 🚨 Alert: ${alert_name}

**Severity**: ${severity}
**Time**: ${timestamp}
**Duration**: ${duration}

### Details
- **Condition**: `${condition}`
- **Current Value**: ${current_value}
- **Threshold**: ${threshold}

### Affected Components
${affected_components}

### Suggested Actions
${suggested_actions}

### Runbook
${runbook_url}
```

## SLI/SLO定義

### 1. サービスレベル指標（SLI）

```yaml
slis:
  availability:
    metric: "successful_requests / total_requests"
    measurement_window: "5m"

  latency:
    metric: "p95(request_duration) < 5000ms"
    measurement_window: "5m"

  quality:
    metric: "modules_with_evidence > 3 / total_modules"
    measurement_window: "1h"
```

### 2. サービスレベル目標（SLO）

```yaml
slos:
  availability:
    target: 99.9
    window: "30d"
    error_budget: 43.2  # minutes per month

  latency:
    target: 95  # 95% of requests < 5s
    window: "7d"

  quality:
    target: 90  # 90% of modules with sufficient evidence
    window: "7d"
```

## 監査ログ

### 1. 監査イベント

```yaml
audit_events:
  - event: "classification_changed"
    data:
      module: "string"
      old_classification: "string"
      new_classification: "string"
      reason: "string"
      agent: "string"

  - event: "score_adjusted"
    data:
      module: "string"
      score_type: "S|Δ|X"
      old_value: "integer"
      new_value: "integer"
      reason: "string"

  - event: "default_score_used"
    data:
      module: "string"
      score_type: "string"
      reason: "evidence_insufficient"
      evidence_count: "integer"
```

### 2. 監査ログ保存

```yaml
audit_storage:
  immediate:
    destination: "audit_database"
    retention: "90d"
    encryption: true

  archive:
    destination: "s3://audit-logs-archive"
    retention: "7years"
    format: "parquet"
    compression: "snappy"
```

## パフォーマンス分析

### 1. ボトルネック検出

```sql
-- 最も時間がかかっているエージェントの特定
SELECT
  agent_name,
  AVG(processing_time_ms) as avg_time,
  MAX(processing_time_ms) as max_time,
  COUNT(*) as request_count
FROM agent_metrics
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY agent_name
ORDER BY avg_time DESC;
```

### 2. 改善機会の特定

```python
def analyze_performance_trends():
    """パフォーマンストレンドを分析"""
    metrics = {
        "baseline_latency": calculate_baseline(),
        "current_latency": calculate_current(),
        "degradation_rate": calculate_degradation(),
        "bottlenecks": identify_bottlenecks(),
        "optimization_opportunities": suggest_optimizations()
    }
    return metrics
```

## 実装例

### Prometheus設定

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'agents'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'

rule_files:
  - 'alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

### Grafana設定

```json
{
  "dashboard": {
    "title": "DDD Mapping Agents",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(requests_total[5m])",
            "legendFormat": "{{agent}}"
          }
        ]
      }
    ]
  }
}
```
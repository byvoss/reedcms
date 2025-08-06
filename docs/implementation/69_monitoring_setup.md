# Monitoring Setup

## Overview

Monitoring Setup Implementation für ReedCMS. Production Monitoring mit Prometheus, Grafana und Alerting.

## Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    service: 'reedcms'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules
rule_files:
  - "alerts/*.yml"
  - "recording_rules/*.yml"

# Scrape configurations
scrape_configs:
  # ReedCMS application metrics
  - job_name: 'reedcms'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
            - staging
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
  
  # Node exporter
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
  
  # PostgreSQL exporter
  - job_name: 'postgres'
    static_configs:
      - targets:
          - postgres-exporter:9187
    params:
      db: ['reedcms']
  
  # Redis exporter
  - job_name: 'redis'
    static_configs:
      - targets:
          - redis-exporter:9121

# Alert rules
# alerts/application.yml
groups:
  - name: reedcms_application
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (instance)
            /
            sum(rate(http_requests_total[5m])) by (instance)
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          service: reedcms
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"
      
      # High response time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, instance)
          ) > 1
        for: 10m
        labels:
          severity: warning
          service: reedcms
        annotations:
          summary: "High response time on {{ $labels.instance }}"
          description: "95th percentile response time is {{ $value }}s"
      
      # Pod restart
      - alert: PodRestart
        expr: |
          increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 3
        labels:
          severity: warning
          service: reedcms
        annotations:
          summary: "Pod {{ $labels.pod }} has restarted {{ $value }} times"
          description: "Pod in namespace {{ $labels.namespace }} is restarting frequently"
      
      # Memory pressure
      - alert: HighMemoryUsage
        expr: |
          (
            container_memory_working_set_bytes{pod=~"reedcms-.*"}
            /
            container_spec_memory_limit_bytes{pod=~"reedcms-.*"}
          ) > 0.8
        for: 5m
        labels:
          severity: warning
          service: reedcms
        annotations:
          summary: "High memory usage in {{ $labels.pod }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

# alerts/database.yml
groups:
  - name: database
    rules:
      # Database connection pool exhausted
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          (
            pg_stat_database_numbackends{datname="reedcms"}
            /
            pg_settings_max_connections
          ) > 0.8
        for: 5m
        labels:
          severity: critical
          service: postgresql
        annotations:
          summary: "Database connection pool near exhaustion"
          description: "{{ $value | humanizePercentage }} of connections are in use"
      
      # Slow queries
      - alert: DatabaseSlowQueries
        expr: |
          rate(pg_stat_statements_mean_exec_time_seconds{datname="reedcms"}[5m]) > 1
        for: 10m
        labels:
          severity: warning
          service: postgresql
        annotations:
          summary: "Slow database queries detected"
          description: "Average query execution time is {{ $value }}s"
      
      # Replication lag
      - alert: DatabaseReplicationLag
        expr: |
          pg_replication_lag_seconds > 10
        for: 5m
        labels:
          severity: critical
          service: postgresql
        annotations:
          summary: "Database replication lag is high"
          description: "Replication lag is {{ $value }}s"
```

## Grafana Dashboards

```json
// grafana/dashboards/reedcms-overview.json
{
  "dashboard": {
    "title": "ReedCMS Overview",
    "uid": "reedcms-overview",
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Rate",
        "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (method, status)",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "type": "graph",
        "yaxes": [
          { "format": "reqps", "label": "Requests/sec" }
        ]
      },
      {
        "title": "Response Time",
        "gridPos": { "x": 12, "y": 0, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "p99"
          }
        ],
        "type": "graph",
        "yaxes": [
          { "format": "s", "label": "Response Time" }
        ]
      },
      {
        "title": "Error Rate",
        "gridPos": { "x": 0, "y": 8, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
            "legendFormat": "Error Rate"
          }
        ],
        "type": "graph",
        "yaxes": [
          { "format": "percentunit", "label": "Error Rate" }
        ],
        "thresholds": [
          { "value": 0.01, "color": "yellow" },
          { "value": 0.05, "color": "red" }
        ]
      },
      {
        "title": "Active Users",
        "gridPos": { "x": 12, "y": 8, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "reedcms_active_users",
            "legendFormat": "Active Users"
          }
        ],
        "type": "stat",
        "options": {
          "colorMode": "value",
          "graphMode": "area"
        }
      },
      {
        "title": "Database Connections",
        "gridPos": { "x": 0, "y": 16, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "pg_stat_database_numbackends{datname=\"reedcms\"}",
            "legendFormat": "Active Connections"
          },
          {
            "expr": "pg_settings_max_connections",
            "legendFormat": "Max Connections"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Cache Hit Rate",
        "gridPos": { "x": 12, "y": 16, "w": 12, "h": 8 },
        "targets": [
          {
            "expr": "rate(cache_hits_total[5m]) / (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))",
            "legendFormat": "Hit Rate"
          }
        ],
        "type": "gauge",
        "options": {
          "showThresholdLabels": true,
          "showThresholdMarkers": true
        },
        "fieldConfig": {
          "defaults": {
            "min": 0,
            "max": 1,
            "unit": "percentunit",
            "thresholds": {
              "steps": [
                { "value": 0, "color": "red" },
                { "value": 0.8, "color": "yellow" },
                { "value": 0.95, "color": "green" }
              ]
            }
          }
        }
      }
    ]
  }
}
```

## Alertmanager Configuration

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m
  slack_api_url: '{{ SLACK_WEBHOOK_URL }}'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Route tree
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts
    - match:
        severity: critical
      receiver: pagerduty
      continue: true
    
    # Database alerts
    - match:
        service: postgresql
      receiver: database-team
    
    # Business hours only
    - match:
        severity: warning
      receiver: slack-warnings
      active_time_intervals:
        - business-hours

# Receivers
receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: 'ReedCMS Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
        send_resolved: true
  
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '{{ PAGERDUTY_SERVICE_KEY }}'
        description: '{{ .GroupLabels.alertname }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          summary: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
  
  - name: 'database-team'
    email_configs:
      - to: 'database-team@reedcms.com'
        headers:
          Subject: 'Database Alert: {{ .GroupLabels.alertname }}'
  
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'
        title: '⚠️ Warning: {{ .GroupLabels.alertname }}'
        text: '{{ .Annotations.description }}'
        send_resolved: true
        actions:
          - type: button
            text: 'View in Grafana'
            url: 'https://grafana.reedcms.com/d/{{ .GroupLabels.dashboard_id }}'

# Inhibition rules
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']

# Active time intervals
time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '09:00'
            end_time: '18:00'
        location: 'Europe/Berlin'
```

## Logging Stack

```yaml
# fluentd/fluent.conf
<source>
  @type tail
  path /var/log/pods/*/*/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

# Enrich with Kubernetes metadata
<filter kubernetes.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
  kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV['KUBERNETES_SERVICE_HOST'] + ':' + ENV['KUBERNETES_SERVICE_PORT'] + '/api'}"
  verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
</filter>

# Parse ReedCMS logs
<filter kubernetes.var.log.containers.reedcms-**>
  @type parser
  key_name log
  reserve_data true
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S%.3NZ
  </parse>
</filter>

# Add tags
<filter kubernetes.**>
  @type record_transformer
  <record>
    cluster "#{ENV['CLUSTER_NAME']}"
    environment "#{ENV['ENVIRONMENT']}"
  </record>
</filter>

# Output to Elasticsearch
<match kubernetes.**>
  @type elasticsearch
  @id out_es
  @log_level info
  include_tag_key true
  host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
  port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
  path "#{ENV['FLUENT_ELASTICSEARCH_PATH']}"
  scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
  ssl_verify "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'] || 'true'}"
  ssl_version "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERSION'] || 'TLSv1_2'}"
  user "#{ENV['FLUENT_ELASTICSEARCH_USER']}"
  password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD']}"
  reload_connections "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'false'}"
  reconnect_on_error true
  reload_on_failure true
  log_es_400_reason false
  logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'logstash'}"
  logstash_dateformat "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_DATEFORMAT'] || '%Y.%m.%d'}"
  logstash_format true
  target_index_key stream
  type_name "_doc"
  <buffer>
    flush_thread_count "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_THREAD_COUNT'] || '8'}"
    flush_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_INTERVAL'] || '5s'}"
    chunk_limit_size "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_CHUNK_LIMIT_SIZE'] || '2M'}"
    queue_limit_length "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_QUEUE_LIMIT_LENGTH'] || '32'}"
    retry_max_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_RETRY_MAX_INTERVAL'] || '30'}"
    retry_forever true
  </buffer>
</match>
```

## Custom Metrics

```rust
// src/monitoring/custom_metrics.rs
use prometheus::{register_histogram_vec, register_counter_vec, register_gauge_vec};
use lazy_static::lazy_static;

lazy_static! {
    // Business metrics
    pub static ref CONTENT_OPERATIONS: CounterVec = register_counter_vec!(
        "reedcms_content_operations_total",
        "Total number of content operations",
        &["operation", "content_type", "status"]
    ).unwrap();
    
    pub static ref ACTIVE_SESSIONS: GaugeVec = register_gauge_vec!(
        "reedcms_active_sessions",
        "Number of active user sessions",
        &["user_type"]
    ).unwrap();
    
    pub static ref SEARCH_LATENCY: HistogramVec = register_histogram_vec!(
        "reedcms_search_duration_seconds",
        "Search query latency",
        &["index", "query_type"],
        vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
    ).unwrap();
    
    pub static ref PLUGIN_EXECUTION: HistogramVec = register_histogram_vec!(
        "reedcms_plugin_execution_seconds",
        "Plugin execution time",
        &["plugin", "hook"],
        vec![0.001, 0.01, 0.1, 1.0, 10.0]
    ).unwrap();
    
    pub static ref CACHE_SIZE: GaugeVec = register_gauge_vec!(
        "reedcms_cache_size_bytes",
        "Current cache size in bytes",
        &["cache_type"]
    ).unwrap();
}

/// Record content operation
pub fn record_content_operation(operation: &str, content_type: &str, success: bool) {
    CONTENT_OPERATIONS
        .with_label_values(&[operation, content_type, if success { "success" } else { "failure" }])
        .inc();
}

/// Update active sessions
pub fn update_active_sessions(user_type: &str, count: i64) {
    ACTIVE_SESSIONS
        .with_label_values(&[user_type])
        .set(count as f64);
}

/// Record search latency
pub fn record_search_latency(index: &str, query_type: &str, duration: Duration) {
    SEARCH_LATENCY
        .with_label_values(&[index, query_type])
        .observe(duration.as_secs_f64());
}
```

## Health Checks

```rust
// src/monitoring/health.rs
use axum::{Json, response::IntoResponse};
use serde::Serialize;

#[derive(Serialize)]
pub struct HealthResponse {
    status: String,
    version: String,
    timestamp: chrono::DateTime<chrono::Utc>,
    checks: HashMap<String, ComponentHealth>,
}

#[derive(Serialize)]
pub struct ComponentHealth {
    status: String,
    latency_ms: Option<u64>,
    message: Option<String>,
}

/// Main health check endpoint
pub async fn health_check(
    State(app_state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let mut checks = HashMap::new();
    
    // Database health
    let db_start = Instant::now();
    let db_health = match sqlx::query("SELECT 1")
        .fetch_one(&app_state.db_pool)
        .await
    {
        Ok(_) => ComponentHealth {
            status: "healthy".to_string(),
            latency_ms: Some(db_start.elapsed().as_millis() as u64),
            message: None,
        },
        Err(e) => ComponentHealth {
            status: "unhealthy".to_string(),
            latency_ms: None,
            message: Some(e.to_string()),
        },
    };
    checks.insert("database".to_string(), db_health);
    
    // Redis health
    let redis_start = Instant::now();
    let redis_health = match app_state.redis_client
        .get_async_connection()
        .await
    {
        Ok(mut conn) => {
            match redis::cmd("PING")
                .query_async::<_, String>(&mut conn)
                .await
            {
                Ok(_) => ComponentHealth {
                    status: "healthy".to_string(),
                    latency_ms: Some(redis_start.elapsed().as_millis() as u64),
                    message: None,
                },
                Err(e) => ComponentHealth {
                    status: "unhealthy".to_string(),
                    latency_ms: None,
                    message: Some(e.to_string()),
                },
            }
        }
        Err(e) => ComponentHealth {
            status: "unhealthy".to_string(),
            latency_ms: None,
            message: Some(e.to_string()),
        },
    };
    checks.insert("redis".to_string(), redis_health);
    
    // Search health
    let search_health = match app_state.search_service.health_check().await {
        Ok(latency) => ComponentHealth {
            status: "healthy".to_string(),
            latency_ms: Some(latency.as_millis() as u64),
            message: None,
        },
        Err(e) => ComponentHealth {
            status: "unhealthy".to_string(),
            latency_ms: None,
            message: Some(e.to_string()),
        },
    };
    checks.insert("search".to_string(), search_health);
    
    // Overall status
    let overall_status = if checks.values().all(|c| c.status == "healthy") {
        "healthy"
    } else if checks.values().any(|c| c.status == "unhealthy") {
        "unhealthy"
    } else {
        "degraded"
    };
    
    let response = HealthResponse {
        status: overall_status.to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        timestamp: chrono::Utc::now(),
        checks,
    };
    
    (StatusCode::OK, Json(response))
}

/// Readiness check
pub async fn readiness_check(
    State(app_state): State<Arc<AppState>>,
) -> impl IntoResponse {
    // Check if app is ready to serve traffic
    if app_state.is_ready() {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    }
}

/// Liveness check
pub async fn liveness_check() -> impl IntoResponse {
    // Simple liveness check
    StatusCode::OK
}
```

## Configuration

```yaml
# monitoring/values.yaml
prometheus:
  retention: 30d
  storage:
    size: 100Gi
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 8Gi
  serviceMonitor:
    enabled: true
    interval: 15s
    path: /metrics

grafana:
  adminPassword: ${GRAFANA_ADMIN_PASSWORD}
  persistence:
    enabled: true
    size: 10Gi
  datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      isDefault: true
    - name: Elasticsearch
      type: elasticsearch
      url: http://elasticsearch:9200
      jsonData:
        timeField: "@timestamp"
        esVersion: 7
  dashboardProviders:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      options:
        path: /var/lib/grafana/dashboards

alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
  persistence:
    enabled: true
    size: 1Gi

elasticsearch:
  replicas: 3
  minimumMasterNodes: 2
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  volumeClaimTemplate:
    resources:
      requests:
        storage: 50Gi

kibana:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```

## Summary

Diese Monitoring Setup Implementation bietet:
- **Metrics Collection** - Prometheus mit Custom Metrics
- **Visualization** - Grafana Dashboards
- **Alerting** - Multi-Channel Alert Routing
- **Log Aggregation** - ELK Stack Integration
- **Health Monitoring** - Comprehensive Health Checks
- **Performance Tracking** - Business und Technical Metrics

Complete Monitoring Infrastructure für Production Operations.
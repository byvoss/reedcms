# Monitoring

## Overview

Monitoring Implementation für ReedCMS. Comprehensive System Monitoring mit Metrics, Logging, Tracing und Alerting.

## Monitoring Core

```rust
use std::sync::Arc;
use prometheus::{Registry, Counter, Gauge, Histogram, HistogramOpts};
use opentelemetry::{trace::Tracer, metrics::Meter};

/// Main monitoring service
pub struct MonitoringService {
    metrics: Arc<MetricsCollector>,
    logger: Arc<StructuredLogger>,
    tracer: Arc<TracingService>,
    health: Arc<HealthChecker>,
    alerter: Arc<AlertManager>,
    config: MonitoringConfig,
}

impl MonitoringService {
    pub fn new(config: MonitoringConfig) -> Result<Self> {
        // Initialize Prometheus registry
        let registry = Registry::new();
        
        // Initialize OpenTelemetry
        let (tracer, meter) = init_opentelemetry(&config.otel)?;
        
        let metrics = Arc::new(MetricsCollector::new(registry, meter)?);
        let logger = Arc::new(StructuredLogger::new(&config.logging)?);
        let tracer = Arc::new(TracingService::new(tracer));
        let health = Arc::new(HealthChecker::new());
        let alerter = Arc::new(AlertManager::new(&config.alerting)?);
        
        Ok(Self {
            metrics,
            logger,
            tracer,
            health,
            alerter,
            config,
        })
    }
    
    /// Start monitoring services
    pub async fn start(&self) -> Result<()> {
        // Start metrics server
        self.start_metrics_server().await?;
        
        // Start health check server
        self.start_health_server().await?;
        
        // Start alert checker
        self.alerter.start().await?;
        
        info!("Monitoring services started");
        
        Ok(())
    }
    
    /// Record metric
    pub fn record_metric(&self, metric: Metric) {
        match metric {
            Metric::Counter { name, value, labels } => {
                self.metrics.increment_counter(&name, value, labels);
            }
            Metric::Gauge { name, value, labels } => {
                self.metrics.set_gauge(&name, value, labels);
            }
            Metric::Histogram { name, value, labels } => {
                self.metrics.observe_histogram(&name, value, labels);
            }
        }
    }
    
    /// Log structured event
    pub fn log(&self, level: LogLevel, message: &str, fields: LogFields) {
        self.logger.log(level, message, fields);
    }
    
    /// Create trace span
    pub fn span(&self, name: &str) -> Span {
        self.tracer.create_span(name)
    }
    
    /// Register health check
    pub async fn register_health_check(&self, check: HealthCheck) {
        self.health.register(check).await;
    }
    
    /// Check system health
    pub async fn check_health(&self) -> HealthStatus {
        self.health.check_all().await
    }
}

/// Metric types
#[derive(Debug, Clone)]
pub enum Metric {
    Counter {
        name: String,
        value: f64,
        labels: HashMap<String, String>,
    },
    Gauge {
        name: String,
        value: f64,
        labels: HashMap<String, String>,
    },
    Histogram {
        name: String,
        value: f64,
        labels: HashMap<String, String>,
    },
}
```

## Metrics Collection

```rust
/// Metrics collector
pub struct MetricsCollector {
    registry: Registry,
    meter: Meter,
    counters: Arc<RwLock<HashMap<String, Counter>>>,
    gauges: Arc<RwLock<HashMap<String, Gauge>>>,
    histograms: Arc<RwLock<HashMap<String, Histogram>>>,
}

impl MetricsCollector {
    pub fn new(registry: Registry, meter: Meter) -> Result<Self> {
        let collector = Self {
            registry,
            meter,
            counters: Arc::new(RwLock::new(HashMap::new())),
            gauges: Arc::new(RwLock::new(HashMap::new())),
            histograms: Arc::new(RwLock::new(HashMap::new())),
        };
        
        // Register default metrics
        collector.register_default_metrics()?;
        
        Ok(collector)
    }
    
    /// Register default system metrics
    fn register_default_metrics(&self) -> Result<()> {
        // HTTP metrics
        self.register_http_metrics()?;
        
        // Database metrics
        self.register_database_metrics()?;
        
        // Cache metrics
        self.register_cache_metrics()?;
        
        // Business metrics
        self.register_business_metrics()?;
        
        Ok(())
    }
    
    /// Register HTTP metrics
    fn register_http_metrics(&self) -> Result<()> {
        // Request counter
        let http_requests = Counter::new("http_requests_total", "Total HTTP requests")?;
        self.registry.register(Box::new(http_requests.clone()))?;
        self.counters.write().unwrap().insert("http_requests_total".to_string(), http_requests);
        
        // Response time histogram
        let response_time = Histogram::with_opts(
            HistogramOpts::new("http_request_duration_seconds", "HTTP request duration")
                .buckets(vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]),
        )?;
        self.registry.register(Box::new(response_time.clone()))?;
        self.histograms.write().unwrap().insert("http_request_duration_seconds".to_string(), response_time);
        
        // Active connections gauge
        let active_connections = Gauge::new("http_active_connections", "Active HTTP connections")?;
        self.registry.register(Box::new(active_connections.clone()))?;
        self.gauges.write().unwrap().insert("http_active_connections".to_string(), active_connections);
        
        Ok(())
    }
    
    /// Increment counter
    pub fn increment_counter(&self, name: &str, value: f64, labels: HashMap<String, String>) {
        if let Some(counter) = self.counters.read().unwrap().get(name) {
            counter.inc_by(value);
        }
        
        // Also record in OpenTelemetry
        let counter = self.meter.f64_counter(name).init();
        counter.add(value, &labels_to_attributes(labels));
    }
    
    /// Set gauge value
    pub fn set_gauge(&self, name: &str, value: f64, labels: HashMap<String, String>) {
        if let Some(gauge) = self.gauges.read().unwrap().get(name) {
            gauge.set(value);
        }
        
        // Also record in OpenTelemetry
        let gauge = self.meter.f64_gauge(name).init();
        gauge.set(value, &labels_to_attributes(labels));
    }
    
    /// Observe histogram value
    pub fn observe_histogram(&self, name: &str, value: f64, labels: HashMap<String, String>) {
        if let Some(histogram) = self.histograms.read().unwrap().get(name) {
            histogram.observe(value);
        }
        
        // Also record in OpenTelemetry
        let histogram = self.meter.f64_histogram(name).init();
        histogram.record(value, &labels_to_attributes(labels));
    }
    
    /// Export metrics
    pub fn export(&self) -> String {
        use prometheus::Encoder;
        let encoder = prometheus::TextEncoder::new();
        let metric_families = self.registry.gather();
        let mut buffer = Vec::new();
        encoder.encode(&metric_families, &mut buffer).unwrap();
        String::from_utf8(buffer).unwrap()
    }
}

/// System metrics collector
pub struct SystemMetrics {
    collector: Arc<MetricsCollector>,
}

impl SystemMetrics {
    /// Collect system metrics
    pub async fn collect(&self) {
        // CPU usage
        if let Ok(cpu) = self.get_cpu_usage().await {
            self.collector.set_gauge("system_cpu_usage_percent", cpu, HashMap::new());
        }
        
        // Memory usage
        if let Ok(mem) = self.get_memory_usage().await {
            self.collector.set_gauge("system_memory_used_bytes", mem.used as f64, HashMap::new());
            self.collector.set_gauge("system_memory_total_bytes", mem.total as f64, HashMap::new());
        }
        
        // Disk usage
        if let Ok(disk) = self.get_disk_usage().await {
            self.collector.set_gauge("system_disk_used_bytes", disk.used as f64, HashMap::new());
            self.collector.set_gauge("system_disk_total_bytes", disk.total as f64, HashMap::new());
        }
        
        // Network stats
        if let Ok(net) = self.get_network_stats().await {
            self.collector.increment_counter("system_network_rx_bytes", net.rx_bytes as f64, HashMap::new());
            self.collector.increment_counter("system_network_tx_bytes", net.tx_bytes as f64, HashMap::new());
        }
    }
    
    /// Get CPU usage
    async fn get_cpu_usage(&self) -> Result<f64> {
        use sysinfo::{SystemExt, ProcessorExt};
        let mut system = sysinfo::System::new();
        system.refresh_cpu();
        tokio::time::sleep(Duration::from_millis(100)).await;
        system.refresh_cpu();
        
        let cpu_usage = system.global_processor_info().cpu_usage();
        Ok(cpu_usage as f64)
    }
    
    /// Get memory usage
    async fn get_memory_usage(&self) -> Result<MemoryStats> {
        use sysinfo::SystemExt;
        let mut system = sysinfo::System::new();
        system.refresh_memory();
        
        Ok(MemoryStats {
            total: system.total_memory(),
            used: system.used_memory(),
            available: system.available_memory(),
        })
    }
}

#[derive(Debug)]
struct MemoryStats {
    total: u64,
    used: u64,
    available: u64,
}
```

## Structured Logging

```rust
use slog::{Logger, Drain, o, info, warn, error, debug};

/// Structured logger
pub struct StructuredLogger {
    logger: Logger,
    config: LoggingConfig,
}

impl StructuredLogger {
    pub fn new(config: &LoggingConfig) -> Result<Self> {
        let logger = match &config.output {
            LogOutput::Stdout => {
                let decorator = slog_term::TermDecorator::new().build();
                let drain = slog_term::FullFormat::new(decorator).build().fuse();
                let drain = slog_async::Async::new(drain).build().fuse();
                
                Logger::root(drain, o!("service" => "reed-cms"))
            }
            LogOutput::File { path } => {
                let file = std::fs::OpenOptions::new()
                    .create(true)
                    .write(true)
                    .append(true)
                    .open(path)?;
                
                let decorator = slog_term::PlainDecorator::new(file);
                let drain = slog_json::Json::new(decorator)
                    .add_default_keys()
                    .build()
                    .fuse();
                let drain = slog_async::Async::new(drain).build().fuse();
                
                Logger::root(drain, o!("service" => "reed-cms"))
            }
            LogOutput::Elasticsearch { url } => {
                // Elasticsearch drain
                let drain = ElasticsearchDrain::new(url)?;
                let drain = slog_async::Async::new(drain).build().fuse();
                
                Logger::root(drain, o!("service" => "reed-cms"))
            }
        };
        
        Ok(Self { logger, config })
    }
    
    /// Log message
    pub fn log(&self, level: LogLevel, message: &str, fields: LogFields) {
        let logger = self.logger.new(fields_to_slog(fields));
        
        match level {
            LogLevel::Debug => debug!(logger, "{}", message),
            LogLevel::Info => info!(logger, "{}", message),
            LogLevel::Warn => warn!(logger, "{}", message),
            LogLevel::Error => error!(logger, "{}", message),
        }
    }
    
    /// Create context logger
    pub fn with_context(&self, context: LogFields) -> ContextLogger {
        ContextLogger {
            logger: self.logger.new(fields_to_slog(context)),
        }
    }
}

/// Context logger with preset fields
pub struct ContextLogger {
    logger: Logger,
}

impl ContextLogger {
    pub fn debug(&self, message: &str, fields: LogFields) {
        debug!(self.logger.new(fields_to_slog(fields)), "{}", message);
    }
    
    pub fn info(&self, message: &str, fields: LogFields) {
        info!(self.logger.new(fields_to_slog(fields)), "{}", message);
    }
    
    pub fn warn(&self, message: &str, fields: LogFields) {
        warn!(self.logger.new(fields_to_slog(fields)), "{}", message);
    }
    
    pub fn error(&self, message: &str, fields: LogFields) {
        error!(self.logger.new(fields_to_slog(fields)), "{}", message);
    }
}

/// Log fields
pub type LogFields = HashMap<String, serde_json::Value>;

/// Convert fields to slog
fn fields_to_slog(fields: LogFields) -> slog::OwnedKVList {
    let mut kv = vec![];
    
    for (key, value) in fields {
        kv.push((key, format!("{}", value)));
    }
    
    o!(kv)
}

#[derive(Debug, Clone, Copy)]
pub enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}
```

## Distributed Tracing

```rust
use opentelemetry::{
    trace::{Tracer, SpanKind, Status},
    Context,
};
use tracing_opentelemetry::OpenTelemetryLayer;

/// Tracing service
pub struct TracingService {
    tracer: Box<dyn Tracer>,
}

impl TracingService {
    pub fn new(tracer: Box<dyn Tracer>) -> Self {
        Self { tracer }
    }
    
    /// Create new span
    pub fn create_span(&self, name: &str) -> Span {
        let span = self.tracer
            .span_builder(name)
            .with_kind(SpanKind::Internal)
            .start(&self.tracer);
        
        Span::new(span)
    }
    
    /// Create HTTP span
    pub fn create_http_span(&self, method: &str, path: &str) -> Span {
        let span = self.tracer
            .span_builder(format!("{} {}", method, path))
            .with_kind(SpanKind::Server)
            .with_attributes(vec![
                KeyValue::new("http.method", method.to_string()),
                KeyValue::new("http.path", path.to_string()),
            ])
            .start(&self.tracer);
        
        Span::new(span)
    }
    
    /// Create database span
    pub fn create_db_span(&self, operation: &str, table: &str) -> Span {
        let span = self.tracer
            .span_builder(format!("db.{}", operation))
            .with_kind(SpanKind::Client)
            .with_attributes(vec![
                KeyValue::new("db.operation", operation.to_string()),
                KeyValue::new("db.table", table.to_string()),
            ])
            .start(&self.tracer);
        
        Span::new(span)
    }
}

/// Span wrapper
pub struct Span {
    inner: Box<dyn opentelemetry::trace::Span>,
}

impl Span {
    fn new(inner: Box<dyn opentelemetry::trace::Span>) -> Self {
        Self { inner }
    }
    
    /// Add event to span
    pub fn add_event(&mut self, name: &str, attributes: Vec<KeyValue>) {
        self.inner.add_event(name, attributes);
    }
    
    /// Set span status
    pub fn set_status(&mut self, status: Status) {
        self.inner.set_status(status);
    }
    
    /// Record error
    pub fn record_error(&mut self, error: &dyn std::error::Error) {
        self.inner.record_error(error);
        self.set_status(Status::error(error.to_string()));
    }
}

/// HTTP tracing middleware
pub async fn tracing_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Result<Response, Error> {
    let tracer = global::tracer("http");
    
    let span = tracer
        .span_builder(format!("{} {}", req.method(), req.uri().path()))
        .with_kind(SpanKind::Server)
        .with_attributes(vec![
            KeyValue::new("http.method", req.method().to_string()),
            KeyValue::new("http.target", req.uri().path().to_string()),
            KeyValue::new("http.host", req.headers().get("host").unwrap_or(&HeaderValue::from_static("")).to_str().unwrap_or("")),
        ])
        .start(&tracer);
    
    let cx = Context::current_with_span(span);
    let response = with_context(cx.clone(), async {
        next.run(req).await
    }).await;
    
    cx.span().set_attribute(KeyValue::new("http.status_code", response.status().as_u16() as i64));
    
    response
}
```

## Health Checks

```rust
/// Health checker
pub struct HealthChecker {
    checks: Arc<RwLock<Vec<HealthCheck>>>,
}

impl HealthChecker {
    pub fn new() -> Self {
        Self {
            checks: Arc::new(RwLock::new(Vec::new())),
        }
    }
    
    /// Register health check
    pub async fn register(&self, check: HealthCheck) {
        self.checks.write().await.push(check);
    }
    
    /// Check all health checks
    pub async fn check_all(&self) -> HealthStatus {
        let checks = self.checks.read().await;
        let mut results = Vec::new();
        let mut overall_status = HealthState::Healthy;
        
        for check in checks.iter() {
            let result = match tokio::time::timeout(
                Duration::from_secs(5),
                (check.check_fn)(),
            ).await {
                Ok(Ok(state)) => HealthCheckResult {
                    name: check.name.clone(),
                    state,
                    message: None,
                    duration: Duration::from_secs(0), // TODO: measure
                },
                Ok(Err(e)) => HealthCheckResult {
                    name: check.name.clone(),
                    state: HealthState::Unhealthy,
                    message: Some(e.to_string()),
                    duration: Duration::from_secs(0),
                },
                Err(_) => HealthCheckResult {
                    name: check.name.clone(),
                    state: HealthState::Unhealthy,
                    message: Some("Check timed out".to_string()),
                    duration: Duration::from_secs(5),
                },
            };
            
            if result.state == HealthState::Unhealthy {
                overall_status = HealthState::Unhealthy;
            } else if result.state == HealthState::Degraded && overall_status != HealthState::Unhealthy {
                overall_status = HealthState::Degraded;
            }
            
            results.push(result);
        }
        
        HealthStatus {
            status: overall_status,
            checks: results,
            timestamp: chrono::Utc::now(),
        }
    }
}

/// Health check definition
pub struct HealthCheck {
    pub name: String,
    pub check_fn: Box<dyn Fn() -> BoxFuture<'static, Result<HealthState>> + Send + Sync>,
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum HealthState {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Debug, Serialize)]
pub struct HealthStatus {
    pub status: HealthState,
    pub checks: Vec<HealthCheckResult>,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Serialize)]
pub struct HealthCheckResult {
    pub name: String,
    pub state: HealthState,
    pub message: Option<String>,
    pub duration: Duration,
}

/// Common health checks
pub mod checks {
    use super::*;
    
    /// Database health check
    pub fn database(pool: Arc<PgPool>) -> HealthCheck {
        HealthCheck {
            name: "database".to_string(),
            check_fn: Box::new(move || {
                let pool = pool.clone();
                Box::pin(async move {
                    match sqlx::query("SELECT 1").fetch_one(&*pool).await {
                        Ok(_) => Ok(HealthState::Healthy),
                        Err(_) => Ok(HealthState::Unhealthy),
                    }
                })
            }),
        }
    }
    
    /// Redis health check
    pub fn redis(client: Arc<redis::Client>) -> HealthCheck {
        HealthCheck {
            name: "redis".to_string(),
            check_fn: Box::new(move || {
                let client = client.clone();
                Box::pin(async move {
                    match client.get_async_connection().await {
                        Ok(mut conn) => {
                            match redis::cmd("PING").query_async::<_, String>(&mut conn).await {
                                Ok(_) => Ok(HealthState::Healthy),
                                Err(_) => Ok(HealthState::Unhealthy),
                            }
                        }
                        Err(_) => Ok(HealthState::Unhealthy),
                    }
                })
            }),
        }
    }
    
    /// Disk space health check
    pub fn disk_space(threshold_percent: u8) -> HealthCheck {
        HealthCheck {
            name: "disk_space".to_string(),
            check_fn: Box::new(move || {
                Box::pin(async move {
                    let stats = sys_info::disk_info()?;
                    let used_percent = (stats.total - stats.free) * 100 / stats.total;
                    
                    if used_percent > threshold_percent as u64 {
                        Ok(HealthState::Degraded)
                    } else {
                        Ok(HealthState::Healthy)
                    }
                })
            }),
        }
    }
}
```

## Alert Manager

```rust
/// Alert management service
pub struct AlertManager {
    rules: Arc<RwLock<Vec<AlertRule>>>,
    notifiers: Arc<RwLock<Vec<Box<dyn Notifier>>>>,
    state: Arc<RwLock<AlertState>>,
    config: AlertingConfig,
}

impl AlertManager {
    pub fn new(config: &AlertingConfig) -> Result<Self> {
        let mut notifiers: Vec<Box<dyn Notifier>> = Vec::new();
        
        // Configure notifiers
        for notifier_config in &config.notifiers {
            match notifier_config {
                NotifierConfig::Email(email) => {
                    notifiers.push(Box::new(EmailNotifier::new(email)?));
                }
                NotifierConfig::Slack(slack) => {
                    notifiers.push(Box::new(SlackNotifier::new(slack)?));
                }
                NotifierConfig::Webhook(webhook) => {
                    notifiers.push(Box::new(WebhookNotifier::new(webhook)?));
                }
            }
        }
        
        Ok(Self {
            rules: Arc::new(RwLock::new(config.rules.clone())),
            notifiers: Arc::new(RwLock::new(notifiers)),
            state: Arc::new(RwLock::new(AlertState::new())),
            config: config.clone(),
        })
    }
    
    /// Start alert checking
    pub async fn start(&self) -> Result<()> {
        let rules = self.rules.clone();
        let notifiers = self.notifiers.clone();
        let state = self.state.clone();
        let interval = self.config.check_interval;
        
        tokio::spawn(async move {
            let mut ticker = tokio::time::interval(interval);
            
            loop {
                ticker.tick().await;
                
                if let Err(e) = check_alerts(&rules, &notifiers, &state).await {
                    error!("Alert check error: {}", e);
                }
            }
        });
        
        Ok(())
    }
}

/// Check alerts
async fn check_alerts(
    rules: &Arc<RwLock<Vec<AlertRule>>>,
    notifiers: &Arc<RwLock<Vec<Box<dyn Notifier>>>>,
    state: &Arc<RwLock<AlertState>>,
) -> Result<()> {
    let rules = rules.read().await;
    let mut alerts_to_send = Vec::new();
    
    for rule in rules.iter() {
        match evaluate_rule(rule).await {
            Ok(true) => {
                // Rule triggered
                let mut state = state.write().await;
                
                if !state.is_firing(&rule.name) {
                    // New alert
                    state.set_firing(&rule.name);
                    alerts_to_send.push(Alert {
                        rule: rule.name.clone(),
                        severity: rule.severity,
                        message: rule.message.clone(),
                        timestamp: chrono::Utc::now(),
                    });
                }
            }
            Ok(false) => {
                // Rule not triggered
                let mut state = state.write().await;
                
                if state.is_firing(&rule.name) {
                    // Alert resolved
                    state.set_resolved(&rule.name);
                    alerts_to_send.push(Alert {
                        rule: rule.name.clone(),
                        severity: Severity::Info,
                        message: format!("RESOLVED: {}", rule.message),
                        timestamp: chrono::Utc::now(),
                    });
                }
            }
            Err(e) => {
                error!("Error evaluating rule {}: {}", rule.name, e);
            }
        }
    }
    
    // Send alerts
    if !alerts_to_send.is_empty() {
        let notifiers = notifiers.read().await;
        
        for alert in alerts_to_send {
            for notifier in notifiers.iter() {
                if let Err(e) = notifier.notify(&alert).await {
                    error!("Failed to send alert: {}", e);
                }
            }
        }
    }
    
    Ok(())
}

/// Alert rule
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AlertRule {
    pub name: String,
    pub condition: AlertCondition,
    pub severity: Severity,
    pub message: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AlertCondition {
    MetricThreshold {
        metric: String,
        operator: ComparisonOperator,
        value: f64,
    },
    HealthCheck {
        check_name: String,
        expected_state: HealthState,
    },
    Custom {
        evaluator: String,
    },
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum Severity {
    Info,
    Warning,
    Error,
    Critical,
}

/// Notifier trait
#[async_trait]
pub trait Notifier: Send + Sync {
    async fn notify(&self, alert: &Alert) -> Result<()>;
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MonitoringConfig {
    pub metrics: MetricsConfig,
    pub logging: LoggingConfig,
    pub tracing: TracingConfig,
    pub alerting: AlertingConfig,
    pub otel: OpenTelemetryConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MetricsConfig {
    pub enabled: bool,
    pub port: u16,
    pub path: String,
    pub collection_interval: Duration,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LoggingConfig {
    pub level: LogLevel,
    pub output: LogOutput,
    pub format: LogFormat,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum LogOutput {
    Stdout,
    File { path: PathBuf },
    Elasticsearch { url: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum LogFormat {
    Json,
    Pretty,
    Compact,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TracingConfig {
    pub enabled: bool,
    pub sample_rate: f64,
    pub exporter: TracingExporter,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TracingExporter {
    Jaeger { endpoint: String },
    Zipkin { endpoint: String },
    Otlp { endpoint: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AlertingConfig {
    pub enabled: bool,
    pub check_interval: Duration,
    pub rules: Vec<AlertRule>,
    pub notifiers: Vec<NotifierConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum NotifierConfig {
    Email(EmailConfig),
    Slack(SlackConfig),
    Webhook(WebhookConfig),
}

impl Default for MonitoringConfig {
    fn default() -> Self {
        Self {
            metrics: MetricsConfig {
                enabled: true,
                port: 9090,
                path: "/metrics".to_string(),
                collection_interval: Duration::from_secs(10),
            },
            logging: LoggingConfig {
                level: LogLevel::Info,
                output: LogOutput::Stdout,
                format: LogFormat::Json,
            },
            tracing: TracingConfig {
                enabled: true,
                sample_rate: 0.1,
                exporter: TracingExporter::Jaeger {
                    endpoint: "http://localhost:14268/api/traces".to_string(),
                },
            },
            alerting: AlertingConfig {
                enabled: true,
                check_interval: Duration::from_secs(60),
                rules: Vec::new(),
                notifiers: Vec::new(),
            },
            otel: OpenTelemetryConfig::default(),
        }
    }
}
```

## Summary

Diese Monitoring Implementation bietet:
- **Metrics Collection** - Prometheus und OpenTelemetry
- **Structured Logging** - JSON Logs mit Context
- **Distributed Tracing** - Request Tracing
- **Health Checks** - System Health Monitoring
- **Alert Management** - Rule-based Alerting
- **Multiple Exporters** - Jaeger, Zipkin, OTLP

Comprehensive Monitoring für Production Systems.
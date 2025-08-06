# Axum Server Setup

## Overview

Axum Web Server Setup für ReedCMS. Minimalistic HTTP Server mit Tokio Runtime, graceful Shutdown und Health Monitoring.

## Server Core

```rust
use axum::{
    Router,
    Server,
    extract::Extension,
    middleware,
};
use tower::ServiceBuilder;
use tower_http::{
    trace::TraceLayer,
    compression::CompressionLayer,
    cors::CorsLayer,
};
use std::net::SocketAddr;
use tokio::signal;

/// Main server instance
pub struct ReedServer {
    config: ServerConfig,
    router: Router,
    context: Arc<ServerContext>,
}

impl ReedServer {
    pub fn new(config: ServerConfig) -> Result<Self> {
        let context = Arc::new(ServerContext::new(&config)?);
        let router = Router::new();
        
        Ok(Self {
            config,
            router,
            context,
        })
    }
    
    /// Build and configure server
    pub fn build(mut self) -> Result<Self> {
        // Add middleware layers
        self.router = self.router
            .layer(
                ServiceBuilder::new()
                    // Add high level tracing/logging
                    .layer(TraceLayer::new_for_http())
                    // Compression
                    .layer(CompressionLayer::new())
                    // CORS if configured
                    .layer(self.build_cors_layer()?)
                    // Shared state
                    .layer(Extension(self.context.clone()))
            );
        
        // Add routes
        self.router = self.configure_routes(self.router)?;
        
        Ok(self)
    }
    
    /// Run server with graceful shutdown
    pub async fn run(self) -> Result<()> {
        let addr: SocketAddr = format!("{}:{}", self.config.host, self.config.port)
            .parse()?;
        
        info!("Starting ReedCMS server on {}", addr);
        
        // Create server
        let server = Server::bind(&addr)
            .serve(self.router.into_make_service())
            .with_graceful_shutdown(shutdown_signal());
        
        // Run server
        if let Err(e) = server.await {
            error!("Server error: {}", e);
            return Err(ReedError::ServerError(e.to_string()));
        }
        
        info!("Server shutdown complete");
        Ok(())
    }
}

/// Graceful shutdown signal handler
async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("Failed to install Ctrl+C handler");
    };
    
    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Failed to install SIGTERM handler")
            .recv()
            .await;
    };
    
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();
    
    tokio::select! {
        _ = ctrl_c => {
            info!("Received Ctrl+C, starting graceful shutdown");
        },
        _ = terminate => {
            info!("Received SIGTERM, starting graceful shutdown");
        },
    }
}
```

## Server Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct ServerConfig {
    /// Host to bind to
    #[serde(default = "default_host")]
    pub host: String,
    
    /// Port to bind to
    #[serde(default = "default_port")]
    pub port: u16,
    
    /// Request timeout in seconds
    #[serde(default = "default_timeout")]
    pub request_timeout: u64,
    
    /// Max request body size in bytes
    #[serde(default = "default_body_limit")]
    pub body_limit: usize,
    
    /// Enable CORS
    #[serde(default)]
    pub cors_enabled: bool,
    
    /// Allowed CORS origins
    #[serde(default)]
    pub cors_origins: Vec<String>,
    
    /// Enable compression
    #[serde(default = "default_compression")]
    pub compression_enabled: bool,
    
    /// Worker threads (0 = CPU count)
    #[serde(default)]
    pub worker_threads: usize,
    
    /// Keep-alive timeout in seconds
    #[serde(default = "default_keepalive")]
    pub keepalive_timeout: u64,
}

fn default_host() -> String {
    "0.0.0.0".to_string()
}

fn default_port() -> u16 {
    3000
}

fn default_timeout() -> u64 {
    30
}

fn default_body_limit() -> usize {
    10 * 1024 * 1024 // 10MB
}

fn default_compression() -> bool {
    true
}

fn default_keepalive() -> u64 {
    75
}
```

## Server Context

```rust
/// Shared server context
pub struct ServerContext {
    pub config: Arc<Config>,
    pub db_pool: Arc<DatabasePool>,
    pub redis: Arc<RedisConnection>,
    pub ucg_manager: Arc<UcgManager>,
    pub snippet_manager: Arc<SnippetManager>,
    pub template_engine: Arc<TeraEngine>,
    pub theme_manager: Arc<ThemeManager>,
    pub cache: Arc<ResponseCache>,
    pub metrics: Arc<Metrics>,
}

impl ServerContext {
    pub fn new(config: &ServerConfig) -> Result<Self> {
        // Load main config
        let config = Arc::new(Config::load()?);
        
        // Initialize database
        let db_pool = Arc::new(DatabasePool::new(&config.database).await?);
        
        // Initialize Redis
        let redis = Arc::new(RedisConnection::new(&config.redis).await?);
        
        // Initialize managers
        let ucg_manager = Arc::new(UcgManager::new(
            db_pool.clone(),
            redis.clone(),
        ));
        
        let snippet_manager = Arc::new(SnippetManager::new(
            ucg_manager.clone(),
            redis.clone(),
        ));
        
        let template_engine = Arc::new(TeraEngine::new(&config.templates)?);
        
        let theme_manager = Arc::new(ThemeManager::new(
            &config.themes_path,
            redis.clone(),
        ));
        
        let cache = Arc::new(ResponseCache::new(
            redis.clone(),
            config.cache.response_ttl,
        ));
        
        let metrics = Arc::new(Metrics::new());
        
        Ok(Self {
            config,
            db_pool,
            redis,
            ucg_manager,
            snippet_manager,
            template_engine,
            theme_manager,
            cache,
            metrics,
        })
    }
}
```

## CORS Configuration

```rust
impl ReedServer {
    /// Build CORS layer
    fn build_cors_layer(&self) -> Result<CorsLayer> {
        let mut cors = CorsLayer::new();
        
        if self.config.cors_enabled {
            // Configure allowed origins
            if self.config.cors_origins.is_empty() {
                cors = cors.allow_origin(tower_http::cors::Any);
            } else {
                for origin in &self.config.cors_origins {
                    cors = cors.allow_origin(
                        origin.parse::<axum::http::HeaderValue>()
                            .map_err(|_| ReedError::InvalidConfig(
                                format!("Invalid CORS origin: {}", origin)
                            ))?
                    );
                }
            }
            
            // Configure methods
            cors = cors
                .allow_methods([
                    axum::http::Method::GET,
                    axum::http::Method::POST,
                    axum::http::Method::PUT,
                    axum::http::Method::DELETE,
                    axum::http::Method::OPTIONS,
                ])
                .allow_headers([
                    axum::http::header::CONTENT_TYPE,
                    axum::http::header::AUTHORIZATION,
                    axum::http::header::ACCEPT,
                ])
                .expose_headers([
                    axum::http::header::CONTENT_LENGTH,
                    axum::http::header::ETAG,
                ])
                .max_age(std::time::Duration::from_secs(3600));
        }
        
        Ok(cors)
    }
}
```

## Health Check

```rust
use axum::{
    response::Json,
    extract::State,
};
use serde_json::json;

/// Health check endpoint
pub async fn health_check(
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Json<serde_json::Value>, StatusCode> {
    let mut health = json!({
        "status": "healthy",
        "timestamp": chrono::Utc::now().to_rfc3339(),
        "version": env!("CARGO_PKG_VERSION"),
    });
    
    // Check database
    let db_healthy = sqlx::query("SELECT 1")
        .fetch_optional(&*ctx.db_pool.pool)
        .await
        .is_ok();
    
    health["database"] = json!({
        "status": if db_healthy { "healthy" } else { "unhealthy" },
    });
    
    // Check Redis
    let redis_healthy = ctx.redis.ping().await.is_ok();
    
    health["redis"] = json!({
        "status": if redis_healthy { "healthy" } else { "unhealthy" },
    });
    
    // Overall status
    if !db_healthy || !redis_healthy {
        health["status"] = json!("degraded");
        return Ok(Json(health));
    }
    
    Ok(Json(health))
}

/// Readiness check
pub async fn readiness_check(
    State(ctx): State<Arc<ServerContext>>,
) -> StatusCode {
    // Check if all services are ready
    let db_ready = sqlx::query("SELECT 1")
        .fetch_optional(&*ctx.db_pool.pool)
        .await
        .is_ok();
    
    let redis_ready = ctx.redis.ping().await.is_ok();
    
    if db_ready && redis_ready {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    }
}
```

## Metrics Collection

```rust
use prometheus::{IntCounter, Histogram, Registry};

/// Server metrics
pub struct Metrics {
    pub requests_total: IntCounter,
    pub request_duration: Histogram,
    pub active_connections: IntGauge,
    pub response_size: Histogram,
    registry: Registry,
}

impl Metrics {
    pub fn new() -> Self {
        let registry = Registry::new();
        
        let requests_total = IntCounter::new(
            "reed_http_requests_total",
            "Total number of HTTP requests"
        ).unwrap();
        registry.register(Box::new(requests_total.clone())).unwrap();
        
        let request_duration = Histogram::new(
            "reed_http_request_duration_seconds",
            "HTTP request duration in seconds"
        ).unwrap();
        registry.register(Box::new(request_duration.clone())).unwrap();
        
        let active_connections = IntGauge::new(
            "reed_http_connections_active",
            "Number of active HTTP connections"
        ).unwrap();
        registry.register(Box::new(active_connections.clone())).unwrap();
        
        let response_size = Histogram::new(
            "reed_http_response_size_bytes",
            "HTTP response size in bytes"
        ).unwrap();
        registry.register(Box::new(response_size.clone())).unwrap();
        
        Self {
            requests_total,
            request_duration,
            active_connections,
            response_size,
            registry,
        }
    }
    
    /// Export metrics in Prometheus format
    pub fn export(&self) -> String {
        use prometheus::Encoder;
        let encoder = prometheus::TextEncoder::new();
        let metric_families = self.registry.gather();
        let mut buffer = Vec::new();
        encoder.encode(&metric_families, &mut buffer).unwrap();
        String::from_utf8(buffer).unwrap()
    }
}

/// Metrics endpoint
pub async fn metrics_handler(
    State(ctx): State<Arc<ServerContext>>,
) -> impl IntoResponse {
    let metrics = ctx.metrics.export();
    Response::builder()
        .header("Content-Type", "text/plain; version=0.0.4")
        .body(metrics)
        .unwrap()
}
```

## Server Builder

```rust
/// Convenient server builder
pub struct ServerBuilder {
    config: ServerConfig,
    routes: Vec<Router>,
}

impl ServerBuilder {
    pub fn new() -> Self {
        Self {
            config: ServerConfig::default(),
            routes: Vec::new(),
        }
    }
    
    /// Set host
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.config.host = host.into();
        self
    }
    
    /// Set port
    pub fn port(mut self, port: u16) -> Self {
        self.config.port = port;
        self
    }
    
    /// Enable CORS
    pub fn cors(mut self, origins: Vec<String>) -> Self {
        self.config.cors_enabled = true;
        self.config.cors_origins = origins;
        self
    }
    
    /// Add routes
    pub fn routes(mut self, routes: Router) -> Self {
        self.routes.push(routes);
        self
    }
    
    /// Build server
    pub fn build(self) -> Result<ReedServer> {
        let mut server = ReedServer::new(self.config)?;
        
        // Merge all routes
        for routes in self.routes {
            server.router = server.router.merge(routes);
        }
        
        server.build()
    }
}

/// Default implementation
impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            host: default_host(),
            port: default_port(),
            request_timeout: default_timeout(),
            body_limit: default_body_limit(),
            cors_enabled: false,
            cors_origins: Vec::new(),
            compression_enabled: default_compression(),
            worker_threads: 0,
            keepalive_timeout: default_keepalive(),
        }
    }
}
```

## Summary

Dieser Server Setup bietet:
- **Axum Framework** - Modern Rust Web Framework
- **Graceful Shutdown** - Clean Server Termination
- **Health Checks** - Readiness und Liveness Probes
- **CORS Support** - Configurable Cross-Origin Settings
- **Metrics** - Prometheus-compatible Metrics
- **Compression** - Automatic Response Compression

Minimalistic aber production-ready Server Foundation.
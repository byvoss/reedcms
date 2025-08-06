# Database Connection Setup

## Overview

SQLx-basierte Datenbankverbindung mit Connection Pooling für PostgreSQL Main und UCG Datenbanken.

## Connection Manager

```rust
use sqlx::{postgres::PgPoolOptions, PgPool, Postgres, Transaction};
use std::time::Duration;

/// Manages database connections for both main and UCG databases
pub struct DatabaseManager {
    main_pool: PgPool,
    ucg_pool: PgPool,
    config: DatabaseConfig,
}

impl DatabaseManager {
    /// Create new database manager with configured pools
    pub async fn new(config: DatabaseConfig) -> Result<Self> {
        // Main content database pool
        let main_pool = Self::create_pool(&config.main_url, &config).await?;
        
        // UCG backup database pool (fewer connections needed)
        let mut ucg_config = config.clone();
        ucg_config.max_connections = config.max_connections / 2;
        let ucg_pool = Self::create_pool(&config.ucg_url, &ucg_config).await?;
        
        Ok(Self {
            main_pool,
            ucg_pool,
            config,
        })
    }
    
    /// Create connection pool with configuration
    async fn create_pool(url: &str, config: &DatabaseConfig) -> Result<PgPool> {
        let pool = PgPoolOptions::new()
            .max_connections(config.max_connections)
            .min_connections(config.min_connections)
            .acquire_timeout(Duration::from_secs(config.connection_timeout_seconds))
            .idle_timeout(Duration::from_secs(config.idle_timeout_seconds))
            .test_before_acquire(true)
            .connect(url)
            .await
            .map_err(|e| ReedError::Database(e))?;
        
        // Verify connection
        sqlx::query("SELECT 1")
            .fetch_one(&pool)
            .await
            .map_err(|e| ReedError::Database(e))?;
        
        Ok(pool)
    }
    
    /// Get reference to main pool
    pub fn main_pool(&self) -> &PgPool {
        &self.main_pool
    }
    
    /// Get reference to UCG pool
    pub fn ucg_pool(&self) -> &PgPool {
        &self.ucg_pool
    }
}
```

## Transaction Support

```rust
impl DatabaseManager {
    /// Execute operation in transaction on main database
    pub async fn with_transaction<F, R>(&self, f: F) -> Result<R>
    where
        F: for<'c> FnOnce(&'c mut Transaction<'_, Postgres>) -> BoxFuture<'c, Result<R>>,
    {
        let mut tx = self.main_pool.begin().await?;
        
        match f(&mut tx).await {
            Ok(result) => {
                tx.commit().await?;
                Ok(result)
            }
            Err(e) => {
                // Transaction automatically rolls back on drop
                Err(e)
            }
        }
    }
    
    /// Execute operation in transaction with retry logic
    pub async fn with_retry_transaction<F, R>(&self, f: F, max_retries: u32) -> Result<R>
    where
        F: for<'c> Fn(&'c mut Transaction<'_, Postgres>) -> BoxFuture<'c, Result<R>> + Clone,
    {
        let mut retries = 0;
        loop {
            match self.with_transaction(f.clone()).await {
                Ok(result) => return Ok(result),
                Err(ReedError::Database(e)) if retries < max_retries => {
                    // Check for retryable errors
                    let error_str = e.to_string();
                    if error_str.contains("deadlock") || 
                       error_str.contains("could not serialize") {
                        retries += 1;
                        let backoff = Duration::from_millis(100 * 2_u64.pow(retries));
                        tokio::time::sleep(backoff).await;
                        continue;
                    }
                    return Err(ReedError::Database(e));
                }
                Err(e) => return Err(e),
            }
        }
    }
}
```

## Health Checks

```rust
impl DatabaseManager {
    /// Check database health
    pub async fn health_check(&self) -> Result<DatabaseHealth> {
        let main_health = self.check_pool_health(&self.main_pool, "main").await?;
        let ucg_health = self.check_pool_health(&self.ucg_pool, "ucg").await?;
        
        Ok(DatabaseHealth {
            main: main_health,
            ucg: ucg_health,
            overall_status: if main_health.is_healthy && ucg_health.is_healthy {
                HealthStatus::Healthy
            } else {
                HealthStatus::Degraded
            },
        })
    }
    
    async fn check_pool_health(&self, pool: &PgPool, name: &str) -> Result<PoolHealth> {
        let start = std::time::Instant::now();
        
        // Test query
        let result = sqlx::query("SELECT 1")
            .fetch_one(pool)
            .await;
        
        let latency = start.elapsed();
        let is_healthy = result.is_ok();
        
        Ok(PoolHealth {
            name: name.to_string(),
            is_healthy,
            latency,
            active_connections: pool.size() as u32,
            idle_connections: pool.num_idle() as u32,
            max_connections: self.config.max_connections,
        })
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct DatabaseHealth {
    pub main: PoolHealth,
    pub ucg: PoolHealth,
    pub overall_status: HealthStatus,
}

#[derive(Debug, Clone, Serialize)]
pub struct PoolHealth {
    pub name: String,
    pub is_healthy: bool,
    pub latency: Duration,
    pub active_connections: u32,
    pub idle_connections: u32,
    pub max_connections: u32,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}
```

## Connection Monitoring

```rust
/// Monitor connection pool metrics
pub struct ConnectionMonitor {
    manager: Arc<DatabaseManager>,
    metrics: Arc<RwLock<ConnectionMetrics>>,
}

#[derive(Debug, Default, Clone)]
pub struct ConnectionMetrics {
    pub total_acquired: u64,
    pub total_released: u64,
    pub total_errors: u64,
    pub slow_queries: u64,
    pub average_wait_time: Duration,
}

impl ConnectionMonitor {
    pub fn new(manager: Arc<DatabaseManager>) -> Self {
        Self {
            manager,
            metrics: Arc::new(RwLock::new(ConnectionMetrics::default())),
        }
    }
    
    /// Start monitoring in background
    pub fn start_monitoring(self) {
        tokio::spawn(async move {
            let mut interval = tokio::time::interval(Duration::from_secs(30));
            
            loop {
                interval.tick().await;
                
                if let Ok(health) = self.manager.health_check().await {
                    self.update_metrics(&health).await;
                    
                    // Log if issues detected
                    if health.overall_status != HealthStatus::Healthy {
                        tracing::warn!(
                            "Database health degraded: main={:?}, ucg={:?}",
                            health.main.is_healthy,
                            health.ucg.is_healthy
                        );
                    }
                }
            }
        });
    }
    
    async fn update_metrics(&self, health: &DatabaseHealth) {
        let mut metrics = self.metrics.write().await;
        
        // Detect anomalies for intelligent monitoring
        if health.main.latency > Duration::from_millis(100) {
            metrics.slow_queries += 1;
            tracing::warn!(
                "Slow database query detected: {:?}",
                health.main.latency
            );
        }
    }
}
```

## Query Builder Helpers

```rust
/// Helper functions for common query patterns
impl DatabaseManager {
    /// Execute query with proper error handling
    pub async fn query<T>(&self, query: &str, params: &[&(dyn ToSql + Sync)]) -> Result<Vec<T>>
    where
        T: for<'r> FromRow<'r, PgRow> + Send + Unpin,
    {
        sqlx::query_as::<_, T>(query)
            .fetch_all(&self.main_pool)
            .await
            .map_err(|e| ReedError::Database(e))
    }
    
    /// Execute single row query
    pub async fn query_one<T>(&self, query: &str, params: &[&(dyn ToSql + Sync)]) -> Result<T>
    where
        T: for<'r> FromRow<'r, PgRow> + Send + Unpin,
    {
        sqlx::query_as::<_, T>(query)
            .fetch_one(&self.main_pool)
            .await
            .map_err(|e| ReedError::Database(e))
    }
    
    /// Execute query that returns optional result
    pub async fn query_optional<T>(&self, query: &str, params: &[&(dyn ToSql + Sync)]) -> Result<Option<T>>
    where
        T: for<'r> FromRow<'r, PgRow> + Send + Unpin,
    {
        sqlx::query_as::<_, T>(query)
            .fetch_optional(&self.main_pool)
            .await
            .map_err(|e| ReedError::Database(e))
    }
}
```

## Graceful Shutdown

```rust
impl DatabaseManager {
    /// Close all connections gracefully
    pub async fn shutdown(&self) -> Result<()> {
        tracing::info!("Shutting down database connections");
        
        // Close main pool
        self.main_pool.close().await;
        
        // Close UCG pool
        self.ucg_pool.close().await;
        
        tracing::info!("Database connections closed");
        Ok(())
    }
}

/// Ensure cleanup on drop
impl Drop for DatabaseManager {
    fn drop(&mut self) {
        // Pools close automatically on drop, but we log it
        tracing::debug!("DatabaseManager dropped, pools will close");
    }
}
```

## Summary

Dieses Modul bietet:
- **Connection Pooling** mit SQLx für beide Datenbanken
- **Transaction Support** mit automatischem Rollback
- **Health Monitoring** für proaktive Wartung
- **Retry Logic** für transiente Fehler
- **Clean Shutdown** für graceful termination

Die Implementierung folgt dem KISS-Prinzip und nutzt bewährte SQLx-Patterns.
# Redis Connection Setup

## Overview

Redis-Verbindung für Performance-Cache und UCG-Struktur. Single Redis mit TTL-basiertem Memory Management.

## Redis Client Manager

```rust
use redis::{Client, Connection, AsyncCommands, RedisError};
use redis::aio::{ConnectionManager, MultiplexedConnection};
use std::time::Duration;

/// Manages Redis connection with automatic reconnection
pub struct RedisManager {
    client: Client,
    connection: Arc<Mutex<ConnectionManager>>,
    config: RedisConfig,
    key_prefix: String,
}

impl RedisManager {
    /// Create new Redis manager
    pub async fn new(config: RedisConfig) -> Result<Self> {
        let client = Client::open(config.url.as_str())
            .map_err(|e| ReedError::Redis(e))?;
        
        let connection = client.get_tokio_connection_manager().await
            .map_err(|e| ReedError::Redis(e))?;
        
        let manager = Self {
            client,
            connection: Arc::new(Mutex::new(connection)),
            key_prefix: config.key_prefix.clone(),
            config,
        };
        
        // Configure Redis settings
        manager.configure_redis().await?;
        
        Ok(manager)
    }
    
    /// Configure Redis with memory policy
    async fn configure_redis(&self) -> Result<()> {
        let mut conn = self.connection.lock().await;
        
        // Set max memory
        redis::cmd("CONFIG")
            .arg("SET")
            .arg("maxmemory")
            .arg(&self.config.max_memory)
            .query_async(&mut *conn)
            .await?;
        
        // Set eviction policy
        let policy = match self.config.max_memory_policy {
            EvictionPolicy::VolatileLru => "volatile-lru",
            EvictionPolicy::AllkeysLru => "allkeys-lru",
            EvictionPolicy::NoEviction => "noeviction",
        };
        
        redis::cmd("CONFIG")
            .arg("SET")
            .arg("maxmemory-policy")
            .arg(policy)
            .query_async(&mut *conn)
            .await?;
        
        // Select database
        redis::cmd("SELECT")
            .arg(self.config.database)
            .query_async(&mut *conn)
            .await?;
        
        tracing::info!("Redis configured: memory={}, policy={}", 
            self.config.max_memory, policy);
        
        Ok(())
    }
    
    /// Build key with prefix
    fn build_key(&self, key: &str) -> String {
        format!("{}{}", self.key_prefix, key)
    }
}
```

## Key Operations

```rust
impl RedisManager {
    /// Set value with optional TTL
    pub async fn set(&self, key: &str, value: &str, ttl: Option<Duration>) -> Result<()> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        match ttl {
            Some(duration) => {
                conn.set_ex(&full_key, value, duration.as_secs() as usize).await?;
            }
            None => {
                conn.set(&full_key, value).await?;
                // Remove TTL to mark as protected
                conn.persist(&full_key).await?;
            }
        }
        
        Ok(())
    }
    
    /// Get value by key
    pub async fn get(&self, key: &str) -> Result<Option<String>> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let value: Option<String> = conn.get(&full_key).await?;
        Ok(value)
    }
    
    /// Delete key(s)
    pub async fn del(&self, keys: &[&str]) -> Result<u32> {
        if keys.is_empty() {
            return Ok(0);
        }
        
        let mut conn = self.connection.lock().await;
        let full_keys: Vec<String> = keys.iter()
            .map(|k| self.build_key(k))
            .collect();
        
        let deleted: u32 = conn.del(&full_keys).await?;
        Ok(deleted)
    }
    
    /// Check if key exists
    pub async fn exists(&self, key: &str) -> Result<bool> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let exists: bool = conn.exists(&full_key).await?;
        Ok(exists)
    }
}
```

## Hash Operations (for UCG Entities)

```rust
impl RedisManager {
    /// Set hash field
    pub async fn hset(&self, key: &str, field: &str, value: &str) -> Result<()> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        conn.hset(&full_key, field, value).await?;
        // UCG entities have no TTL (protected)
        conn.persist(&full_key).await?;
        
        Ok(())
    }
    
    /// Get hash field
    pub async fn hget(&self, key: &str, field: &str) -> Result<Option<String>> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let value: Option<String> = conn.hget(&full_key, field).await?;
        Ok(value)
    }
    
    /// Get all hash fields
    pub async fn hgetall(&self, key: &str) -> Result<HashMap<String, String>> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let map: HashMap<String, String> = conn.hgetall(&full_key).await?;
        Ok(map)
    }
    
    /// Delete hash field
    pub async fn hdel(&self, key: &str, fields: &[&str]) -> Result<u32> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let deleted: u32 = conn.hdel(&full_key, fields).await?;
        Ok(deleted)
    }
}
```

## Set Operations (for Search Index)

```rust
impl RedisManager {
    /// Add to set
    pub async fn sadd(&self, key: &str, members: &[&str]) -> Result<u32> {
        if members.is_empty() {
            return Ok(0);
        }
        
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let added: u32 = conn.sadd(&full_key, members).await?;
        // Search indexes have no TTL (protected)
        conn.persist(&full_key).await?;
        
        Ok(added)
    }
    
    /// Remove from set
    pub async fn srem(&self, key: &str, members: &[&str]) -> Result<u32> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let removed: u32 = conn.srem(&full_key, members).await?;
        Ok(removed)
    }
    
    /// Get set members
    pub async fn smembers(&self, key: &str) -> Result<Vec<String>> {
        let mut conn = self.connection.lock().await;
        let full_key = self.build_key(key);
        
        let members: Vec<String> = conn.smembers(&full_key).await?;
        Ok(members)
    }
    
    /// Set intersection (for search)
    pub async fn sinter(&self, keys: &[&str]) -> Result<Vec<String>> {
        if keys.is_empty() {
            return Ok(vec![]);
        }
        
        let mut conn = self.connection.lock().await;
        let full_keys: Vec<String> = keys.iter()
            .map(|k| self.build_key(k))
            .collect();
        
        let results: Vec<String> = conn.sinter(&full_keys).await?;
        Ok(results)
    }
}
```

## Memory Management

```rust
impl RedisManager {
    /// Get memory usage statistics
    pub async fn memory_stats(&self) -> Result<MemoryStats> {
        let mut conn = self.connection.lock().await;
        
        // Get memory info
        let info: String = redis::cmd("INFO")
            .arg("memory")
            .query_async(&mut *conn)
            .await?;
        
        // Parse memory stats
        let used_memory = self.parse_info_value(&info, "used_memory")
            .and_then(|v| v.parse::<u64>().ok())
            .unwrap_or(0);
        
        let max_memory = self.parse_info_value(&info, "maxmemory")
            .and_then(|v| v.parse::<u64>().ok())
            .unwrap_or(0);
        
        // Get key count by pattern
        let ucg_keys: u64 = self.count_keys("entity:*").await?;
        let cache_keys: u64 = self.count_keys("page:cache:*").await?;
        let session_keys: u64 = self.count_keys("session:*").await?;
        
        Ok(MemoryStats {
            used_bytes: used_memory,
            max_bytes: max_memory,
            usage_percent: if max_memory > 0 {
                (used_memory as f64 / max_memory as f64) * 100.0
            } else {
                0.0
            },
            ucg_keys,
            cache_keys,
            session_keys,
        })
    }
    
    /// Count keys matching pattern
    async fn count_keys(&self, pattern: &str) -> Result<u64> {
        let mut conn = self.connection.lock().await;
        let full_pattern = self.build_key(pattern);
        
        // Use SCAN for efficient counting
        let mut count = 0u64;
        let mut cursor = 0u64;
        
        loop {
            let (new_cursor, _keys): (u64, Vec<String>) = redis::cmd("SCAN")
                .arg(cursor)
                .arg("MATCH")
                .arg(&full_pattern)
                .arg("COUNT")
                .arg(100)
                .query_async(&mut *conn)
                .await?;
            
            count += _keys.len() as u64;
            cursor = new_cursor;
            
            if cursor == 0 {
                break;
            }
        }
        
        Ok(count)
    }
    
    fn parse_info_value<'a>(&self, info: &'a str, key: &str) -> Option<&'a str> {
        info.lines()
            .find(|line| line.starts_with(&format!("{}:", key)))
            .and_then(|line| line.split(':').nth(1))
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct MemoryStats {
    pub used_bytes: u64,
    pub max_bytes: u64,
    pub usage_percent: f64,
    pub ucg_keys: u64,
    pub cache_keys: u64,
    pub session_keys: u64,
}
```

## Health Monitoring

```rust
impl RedisManager {
    /// Check Redis health
    pub async fn health_check(&self) -> Result<RedisHealth> {
        let start = std::time::Instant::now();
        
        // Test connection with PING
        let mut conn = self.connection.lock().await;
        let pong: String = redis::cmd("PING")
            .query_async(&mut *conn)
            .await?;
        
        let latency = start.elapsed();
        let healthy = pong == "PONG";
        
        // Get memory stats
        let memory = self.memory_stats().await?;
        
        Ok(RedisHealth {
            healthy,
            latency,
            memory_usage_percent: memory.usage_percent,
            connected_clients: self.get_connected_clients().await?,
        })
    }
    
    async fn get_connected_clients(&self) -> Result<u32> {
        let mut conn = self.connection.lock().await;
        
        let info: String = redis::cmd("INFO")
            .arg("clients")
            .query_async(&mut *conn)
            .await?;
        
        self.parse_info_value(&info, "connected_clients")
            .and_then(|v| v.parse::<u32>().ok())
            .ok_or_else(|| ReedError::Internal("Failed to parse client count".to_string()))
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct RedisHealth {
    pub healthy: bool,
    pub latency: Duration,
    pub memory_usage_percent: f64,
    pub connected_clients: u32,
}
```

## Error Recovery

```rust
impl RedisManager {
    /// Execute with automatic retry
    pub async fn with_retry<T, F>(&self, operation: F) -> Result<T>
    where
        F: Fn() -> BoxFuture<'static, Result<T, RedisError>>,
    {
        let mut retries = 0;
        const MAX_RETRIES: u32 = 3;
        
        loop {
            match operation().await {
                Ok(result) => return Ok(result),
                Err(e) if retries < MAX_RETRIES => {
                    retries += 1;
                    let backoff = Duration::from_millis(100 * 2_u64.pow(retries));
                    
                    tracing::warn!(
                        "Redis operation failed, retry {} after {:?}: {}",
                        retries, backoff, e
                    );
                    
                    tokio::time::sleep(backoff).await;
                    continue;
                }
                Err(e) => return Err(ReedError::Redis(e)),
            }
        }
    }
    
    /// Graceful shutdown
    pub async fn shutdown(&self) -> Result<()> {
        // Connection closes automatically on drop
        tracing::info!("Redis connection closing");
        Ok(())
    }
}
```

## Summary

Dieses Modul bietet:
- **Single Redis Instance** mit automatischer Reconnection
- **TTL-basierte Memory Protection** für UCG vs Cache
- **Key Prefix System** für namespace isolation
- **Memory Monitoring** mit statistiken
- **Health Checks** für proaktive Wartung

Alles folgt dem KISS-Prinzip mit klaren Verantwortlichkeiten.
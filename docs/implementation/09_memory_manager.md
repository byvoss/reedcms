# Memory Manager - TTL-based Protection

## Overview

Single Redis mit intelligentem TTL-Management. UCG-Struktur wird geschützt, Cache-Daten sind evictable.

## Memory Manager Core

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

/// Manages Redis memory with TTL-based protection
pub struct MemoryManager {
    redis: Arc<RedisManager>,
    csv_loader: Arc<CsvLoader>,
    monitor: Arc<MemoryMonitor>,
    config: MemoryConfig,
}

#[derive(Debug, Clone)]
pub struct MemoryConfig {
    pub warning_threshold: f64,  // 0.85 = 85%
    pub critical_threshold: f64, // 0.95 = 95%
    pub check_interval: Duration,
}

impl MemoryManager {
    pub fn new(
        redis: Arc<RedisManager>,
        csv_loader: Arc<CsvLoader>,
        config: MemoryConfig,
    ) -> Self {
        let monitor = Arc::new(MemoryMonitor::new(redis.clone(), config.clone()));
        
        Self {
            redis,
            csv_loader,
            monitor,
            config,
        }
    }
    
    /// Start memory monitoring
    pub fn start_monitoring(&self) {
        let monitor = self.monitor.clone();
        tokio::spawn(async move {
            monitor.run().await;
        });
    }
}
```

## UCG Structure Storage (Protected)

```rust
impl MemoryManager {
    /// Store UCG entity - NO TTL (protected from eviction)
    pub async fn store_ucg_entity(&self, entity: &Entity) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let entity_key = keys.entity(&entity.entity_type.to_string(), &entity.id);
        
        // Store as hash with no TTL
        self.redis.hset(&entity_key, "type", &entity.entity_type.to_string()).await?;
        self.redis.hset(&entity_key, "data", &serde_json::to_string(&entity.data)?).await?;
        
        if let Some(name) = &entity.semantic_name {
            self.redis.hset(&entity_key, "semantic_name", name.as_str()).await?;
            
            // Also store semantic mapping
            let semantic_key = keys.semantic(&entity.entity_type.to_string(), name.as_str());
            self.redis.set(&semantic_key, &entity.id.to_string(), None).await?;
        }
        
        // Ensure no TTL (protected)
        self.redis.persist(&entity_key).await?;
        
        tracing::debug!("Stored UCG entity: {} (protected)", entity.id);
        Ok(())
    }
    
    /// Store UCG association - NO TTL
    pub async fn store_ucg_association(&self, assoc: &Association) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let assoc_key = keys.association(&assoc.path);
        
        // Store association data
        self.redis.hset(&assoc_key, "parent_id", &assoc.parent_id.to_string()).await?;
        self.redis.hset(&assoc_key, "child_id", &assoc.child_id.to_string()).await?;
        self.redis.hset(&assoc_key, "type", &assoc.association_type.to_string()).await?;
        self.redis.hset(&assoc_key, "weight", &assoc.weight.to_string()).await?;
        
        // Update parent's children set
        let children_key = keys.children(&assoc.parent_id);
        self.redis.sadd(&children_key, &[&assoc.child_id.to_string()]).await?;
        
        // Store child's parent
        let parent_key = keys.parent(&assoc.child_id);
        self.redis.set(&parent_key, &assoc.parent_id.to_string(), None).await?;
        
        tracing::debug!("Stored UCG association: {} (protected)", assoc.path);
        Ok(())
    }
    
    /// Store search index - NO TTL
    pub async fn store_search_index(&self, word: &str, entity_ids: &[Uuid]) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let word_key = keys.word_index(word);
        
        let ids: Vec<&str> = entity_ids.iter()
            .map(|id| id.as_str())
            .collect();
        
        self.redis.sadd(&word_key, &ids).await?;
        
        tracing::debug!("Updated search index for '{}' (protected)", word);
        Ok(())
    }
}
```

## Cache Storage (Evictable)

```rust
impl MemoryManager {
    /// Store page cache - WITH TTL (can be evicted)
    pub async fn store_page_cache(
        &self, 
        path: &str, 
        locale: &str, 
        theme: &str, 
        content: &str
    ) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let cache_key = keys.page_cache(path, locale, theme);
        
        // Store with TTL - can be evicted by LRU
        self.redis.set(&cache_key, content, Some(RedisTTL::PAGE_CACHE)).await?;
        
        tracing::debug!("Cached page: {} (TTL: {:?})", path, RedisTTL::PAGE_CACHE);
        Ok(())
    }
    
    /// Store session - WITH TTL
    pub async fn store_session(&self, session_id: &str, data: &str) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let session_key = keys.session(session_id);
        
        self.redis.set(&session_key, data, Some(RedisTTL::SESSION)).await?;
        
        tracing::debug!("Stored session: {} (TTL: {:?})", session_id, RedisTTL::SESSION);
        Ok(())
    }
    
    /// Store template cache - WITH TTL
    pub async fn store_template_cache(
        &self,
        name: &str,
        theme: &str,
        compiled: &str
    ) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let cache_key = keys.template_cache(name, theme);
        
        self.redis.set(&cache_key, compiled, Some(RedisTTL::TEMPLATE_CACHE)).await?;
        
        tracing::debug!("Cached template: {} (TTL: {:?})", name, RedisTTL::TEMPLATE_CACHE);
        Ok(())
    }
}
```

## Memory Pressure Handling

```rust
impl MemoryManager {
    /// Handle memory pressure situations
    pub async fn handle_memory_pressure(&self) -> Result<()> {
        let stats = self.redis.memory_stats().await?;
        
        if stats.usage_percent > self.config.critical_threshold * 100.0 {
            tracing::error!("Memory critical: {:.1}%", stats.usage_percent);
            self.emergency_cleanup().await?;
        } else if stats.usage_percent > self.config.warning_threshold * 100.0 {
            tracing::warn!("Memory warning: {:.1}%", stats.usage_percent);
            self.clear_expendable_cache().await?;
        }
        
        Ok(())
    }
    
    /// Clear expendable cache (keep UCG structure)
    async fn clear_expendable_cache(&self) -> Result<()> {
        tracing::info!("Clearing expendable cache data");
        
        // Delete cache keys (have TTL)
        let cache_patterns = vec![
            RedisKeyPatterns::all_cache(),
            RedisKeyPatterns::all_sessions(),
            "template:cache:*".to_string(),
        ];
        
        let mut deleted = 0;
        for pattern in cache_patterns {
            deleted += self.delete_by_pattern(&pattern).await?;
        }
        
        tracing::info!("Cleared {} cache keys", deleted);
        Ok(())
    }
    
    /// Emergency cleanup - nuclear option
    async fn emergency_cleanup(&self) -> Result<()> {
        tracing::error!("Emergency memory cleanup initiated");
        
        // Step 1: Clear all cache
        self.clear_expendable_cache().await?;
        
        // Step 2: Check if enough
        let stats = self.redis.memory_stats().await?;
        if stats.usage_percent < self.config.critical_threshold * 100.0 {
            return Ok(());
        }
        
        // Step 3: Nuclear - rebuild from CSV
        tracing::error!("Memory still critical, rebuilding from CSV");
        self.rebuild_from_csv().await?;
        
        Ok(())
    }
    
    /// Delete keys by pattern
    async fn delete_by_pattern(&self, pattern: &str) -> Result<u32> {
        let mut deleted = 0;
        let mut cursor = 0;
        
        loop {
            let (new_cursor, keys): (u64, Vec<String>) = 
                self.redis.scan_match(pattern, cursor, 100).await?;
            
            if !keys.is_empty() {
                let key_refs: Vec<&str> = keys.iter().map(|s| s.as_str()).collect();
                deleted += self.redis.del(&key_refs).await?;
            }
            
            cursor = new_cursor;
            if cursor == 0 {
                break;
            }
        }
        
        Ok(deleted)
    }
}
```

## CSV Recovery System

```rust
impl MemoryManager {
    /// Rebuild UCG from CSV (nuclear option)
    async fn rebuild_from_csv(&self) -> Result<()> {
        tracing::warn!("Rebuilding UCG structure from CSV");
        
        // Clear everything
        self.redis.flushdb().await?;
        
        // Reload structure
        let loaded = self.csv_loader.load_all().await?;
        
        // Store entities
        for entity in &loaded.entities {
            self.store_ucg_entity(entity).await?;
        }
        
        // Store associations
        for assoc in &loaded.associations {
            self.store_ucg_association(assoc).await?;
        }
        
        // Rebuild search index
        for (word, entities) in &loaded.search_index {
            self.store_search_index(word, entities).await?;
        }
        
        tracing::info!(
            "UCG rebuilt: {} entities, {} associations", 
            loaded.entities.len(),
            loaded.associations.len()
        );
        
        Ok(())
    }
}

#[derive(Debug)]
pub struct CsvLoadResult {
    pub entities: Vec<Entity>,
    pub associations: Vec<Association>,
    pub search_index: HashMap<String, Vec<Uuid>>,
}
```

## Memory Monitor

```rust
/// Background memory monitor
pub struct MemoryMonitor {
    redis: Arc<RedisManager>,
    config: MemoryConfig,
    last_alert: Arc<RwLock<Option<Instant>>>,
}

impl MemoryMonitor {
    pub fn new(redis: Arc<RedisManager>, config: MemoryConfig) -> Self {
        Self {
            redis,
            config,
            last_alert: Arc::new(RwLock::new(None)),
        }
    }
    
    /// Run monitoring loop
    pub async fn run(&self) {
        let mut interval = tokio::time::interval(self.config.check_interval);
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.check_memory().await {
                tracing::error!("Memory check failed: {}", e);
            }
        }
    }
    
    async fn check_memory(&self) -> Result<()> {
        let stats = self.redis.memory_stats().await?;
        let usage_percent = stats.usage_percent;
        
        // Detect anomalies
        if usage_percent > self.config.critical_threshold * 100.0 {
            self.alert_critical(usage_percent).await?;
        } else if usage_percent > self.config.warning_threshold * 100.0 {
            self.alert_warning(usage_percent).await?;
        }
        
        // Log metrics periodically
        if usage_percent > 50.0 {
            tracing::info!(
                "Memory usage: {:.1}% ({} UCG keys, {} cache keys)",
                usage_percent,
                stats.ucg_keys,
                stats.cache_keys
            );
        }
        
        Ok(())
    }
    
    async fn alert_critical(&self, usage: f64) -> Result<()> {
        let mut last = self.last_alert.write().await;
        
        // Rate limit alerts (max 1 per minute)
        if let Some(last_time) = *last {
            if last_time.elapsed() < Duration::from_secs(60) {
                return Ok(());
            }
        }
        
        tracing::error!("CRITICAL: Redis memory at {:.1}%", usage);
        *last = Some(Instant::now());
        
        // Could trigger automated cleanup here
        Ok(())
    }
}
```

## Memory Statistics

```rust
/// Get detailed memory statistics
impl MemoryManager {
    pub async fn detailed_stats(&self) -> Result<DetailedMemoryStats> {
        let basic = self.redis.memory_stats().await?;
        
        // Count keys by type
        let ucg_entities = self.redis.count_keys("entity:*").await?;
        let ucg_assocs = self.redis.count_keys("assoc:*").await?;
        let search_words = self.redis.count_keys("word:*").await?;
        let page_cache = self.redis.count_keys("page:cache:*").await?;
        let sessions = self.redis.count_keys("session:*").await?;
        
        Ok(DetailedMemoryStats {
            total_memory_mb: basic.used_bytes / 1_048_576,
            max_memory_mb: basic.max_bytes / 1_048_576,
            usage_percent: basic.usage_percent,
            protected_keys: ucg_entities + ucg_assocs + search_words,
            evictable_keys: page_cache + sessions + basic.cache_keys,
            breakdown: MemoryBreakdown {
                ucg_entities,
                ucg_associations: ucg_assocs,
                search_index: search_words,
                page_cache,
                sessions,
                other_cache: basic.cache_keys,
            },
        })
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct DetailedMemoryStats {
    pub total_memory_mb: u64,
    pub max_memory_mb: u64,
    pub usage_percent: f64,
    pub protected_keys: u64,
    pub evictable_keys: u64,
    pub breakdown: MemoryBreakdown,
}

#[derive(Debug, Clone, Serialize)]
pub struct MemoryBreakdown {
    pub ucg_entities: u64,
    pub ucg_associations: u64,
    pub search_index: u64,
    pub page_cache: u64,
    pub sessions: u64,
    pub other_cache: u64,
}
```

## Summary

Dieses Memory Management bietet:
- **TTL-basierte Protection** für UCG vs Cache
- **Automatisches Monitoring** mit Alerts
- **Graceful Degradation** bei Memory Pressure
- **CSV Recovery** als Nuclear Option
- **Detailed Statistics** für Debugging

Simple und robust - genau wie ReedCMS es braucht.
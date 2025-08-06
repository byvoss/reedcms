# Snippet Cache System

## Overview

Redis-basiertes Caching für Snippets. Intelligente Cache-Invalidierung mit TTL Management und Hierarchie-aware Updates.

## Cache Manager Core

```rust
use redis::{AsyncCommands, pipe};
use std::time::Duration;
use serde::{Serialize, Deserialize};

/// Redis cache for snippets
pub struct SnippetCache {
    redis: Arc<RedisManager>,
    memory: Arc<MemoryManager>,
    default_ttl: Duration,
    intelligence: Option<Arc<CacheIntelligence>>,
}

impl SnippetCache {
    pub fn new(
        redis: Arc<RedisManager>,
        memory: Arc<MemoryManager>,
        default_ttl: Duration,
    ) -> Self {
        Self {
            redis,
            memory,
            default_ttl,
            intelligence: None,
        }
    }
    
    /// Enable intelligence features
    pub fn with_intelligence(mut self, intelligence: Arc<CacheIntelligence>) -> Self {
        self.intelligence = Some(intelligence);
        self
    }
    
    /// Get snippet from cache
    pub async fn get_snippet(&self, id: Uuid) -> Result<Option<Entity>> {
        let key = self.snippet_key(&id);
        
        // Try memory protection first
        if let Some(data) = self.memory.get(&key).await? {
            let entity: Entity = serde_json::from_slice(&data)?;
            
            // Track cache hit
            self.track_access(&id, true).await?;
            
            return Ok(Some(entity));
        }
        
        // Track cache miss
        self.track_access(&id, false).await?;
        
        Ok(None)
    }
    
    /// Store snippet in cache
    pub async fn store_snippet(&self, entity: &Entity) -> Result<()> {
        let key = self.snippet_key(&entity.id);
        let data = serde_json::to_vec(entity)?;
        
        // Determine TTL
        let ttl = self.calculate_ttl(entity).await?;
        
        // Store with memory protection
        self.memory.set(&key, data, ttl).await?;
        
        // Update indices
        self.update_indices(entity).await?;
        
        Ok(())
    }
    
    /// Invalidate snippet cache
    pub async fn invalidate_snippet(&self, id: &Uuid) -> Result<()> {
        let key = self.snippet_key(id);
        
        // Remove from cache
        self.memory.delete(&key).await?;
        
        // Remove from indices
        self.remove_from_indices(id).await?;
        
        // Invalidate related caches
        self.invalidate_related(id).await?;
        
        Ok(())
    }
}
```

## Cache Key Structure

```rust
impl SnippetCache {
    /// Generate cache key for snippet
    fn snippet_key(&self, id: &Uuid) -> String {
        format!("snippet:{}", id)
    }
    
    /// Generate cache key for snippet list
    fn list_key(&self, snippet_type: &str, page: usize) -> String {
        format!("snippet_list:{}:{}", snippet_type, page)
    }
    
    /// Generate cache key for children
    fn children_key(&self, parent_id: &Uuid) -> String {
        format!("snippet_children:{}", parent_id)
    }
    
    /// Generate cache key for search results
    fn search_key(&self, query: &str, snippet_type: Option<&str>) -> String {
        let type_part = snippet_type.unwrap_or("all");
        let query_hash = self.hash_query(query);
        format!("snippet_search:{}:{}", type_part, query_hash)
    }
    
    /// Hash search query for consistent keys
    fn hash_query(&self, query: &str) -> String {
        use sha2::{Sha256, Digest};
        let mut hasher = Sha256::new();
        hasher.update(query.as_bytes());
        format!("{:x}", hasher.finalize())[..16].to_string()
    }
}
```

## Intelligent TTL Calculation

```rust
impl SnippetCache {
    /// Calculate TTL based on snippet characteristics
    async fn calculate_ttl(&self, entity: &Entity) -> Result<Duration> {
        // Get snippet type
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown");
        
        // Base TTL from snippet type configuration
        let base_ttl = match snippet_type {
            "navigation" => Duration::from_secs(3600),    // 1 hour
            "hero-banner" => Duration::from_secs(1800),   // 30 minutes
            "article" => Duration::from_secs(300),        // 5 minutes
            "comment" => Duration::from_secs(60),         // 1 minute
            _ => self.default_ttl,
        };
        
        // Apply intelligence adjustments
        if let Some(intel) = &self.intelligence {
            let access_pattern = intel.get_access_pattern(&entity.id).await?;
            
            match access_pattern {
                AccessPattern::Hot => base_ttl * 2,        // Double TTL for hot content
                AccessPattern::Warm => base_ttl,           // Normal TTL
                AccessPattern::Cold => base_ttl / 2,       // Half TTL for cold content
                AccessPattern::Stale => Duration::from_secs(30), // Minimal TTL
            }
        } else {
            base_ttl
        }
    }
}

/// Access patterns for cache intelligence
#[derive(Debug, Clone, Copy)]
pub enum AccessPattern {
    Hot,    // Accessed frequently
    Warm,   // Normal access
    Cold,   // Rarely accessed
    Stale,  // Not accessed recently
}
```

## Cache Indices

```rust
impl SnippetCache {
    /// Update cache indices
    async fn update_indices(&self, entity: &Entity) -> Result<()> {
        let mut conn = self.redis.get_connection().await?;
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown");
        
        // Add to type index
        let type_index = format!("idx:snippet_type:{}", snippet_type);
        conn.sadd(&type_index, entity.id.to_string()).await?;
        
        // Add to time index
        let time_index = "idx:snippet_time";
        let score = entity.updated_at.timestamp() as f64;
        conn.zadd(time_index, entity.id.to_string(), score).await?;
        
        // Add to parent index if has parent
        if let Some(parent_id) = self.get_parent_id(entity).await? {
            let parent_index = format!("idx:snippet_parent:{}", parent_id);
            conn.sadd(&parent_index, entity.id.to_string()).await?;
        }
        
        Ok(())
    }
    
    /// Remove from cache indices
    async fn remove_from_indices(&self, id: &Uuid) -> Result<()> {
        let mut conn = self.redis.get_connection().await?;
        
        // Get snippet to find its type
        if let Some(entity) = self.get_snippet(*id).await? {
            let snippet_type = entity.data.get("snippet_type")
                .and_then(|v| v.as_str())
                .unwrap_or("unknown");
            
            // Remove from type index
            let type_index = format!("idx:snippet_type:{}", snippet_type);
            conn.srem(&type_index, id.to_string()).await?;
        }
        
        // Remove from time index
        let time_index = "idx:snippet_time";
        conn.zrem(time_index, id.to_string()).await?;
        
        // Remove from parent indices (scan pattern)
        let parent_pattern = format!("idx:snippet_parent:*");
        let parent_indices: Vec<String> = conn.keys(parent_pattern).await?;
        
        for index in parent_indices {
            conn.srem(&index, id.to_string()).await?;
        }
        
        Ok(())
    }
}
```

## Hierarchical Cache Invalidation

```rust
impl SnippetCache {
    /// Invalidate related caches
    async fn invalidate_related(&self, id: &Uuid) -> Result<()> {
        // Invalidate parent's children cache
        if let Some(parent_id) = self.get_parent_id_for(*id).await? {
            self.invalidate_children(parent_id).await?;
        }
        
        // Invalidate all children recursively
        self.invalidate_descendants(id).await?;
        
        // Invalidate type lists
        if let Some(entity) = self.get_snippet(*id).await? {
            if let Some(snippet_type) = entity.data.get("snippet_type").and_then(|v| v.as_str()) {
                self.invalidate_type_lists(snippet_type).await?;
            }
        }
        
        Ok(())
    }
    
    /// Invalidate children cache
    pub async fn invalidate_children(&self, parent_id: Uuid) -> Result<()> {
        let key = self.children_key(&parent_id);
        self.memory.delete(&key).await?;
        Ok(())
    }
    
    /// Invalidate all descendants recursively
    async fn invalidate_descendants(&self, id: &Uuid) -> Result<()> {
        let mut conn = self.redis.get_connection().await?;
        let parent_index = format!("idx:snippet_parent:{}", id);
        
        // Get all children
        let children: Vec<String> = conn.smembers(&parent_index).await?;
        
        for child_id_str in children {
            if let Ok(child_id) = Uuid::parse_str(&child_id_str) {
                // Invalidate child
                self.invalidate_snippet(&child_id).await?;
                
                // Recurse
                Box::pin(self.invalidate_descendants(&child_id)).await?;
            }
        }
        
        Ok(())
    }
    
    /// Invalidate type list caches
    async fn invalidate_type_lists(&self, snippet_type: &str) -> Result<()> {
        let pattern = format!("snippet_list:{}:*", snippet_type);
        let mut conn = self.redis.get_connection().await?;
        
        // Find all list keys
        let keys: Vec<String> = conn.keys(&pattern).await?;
        
        // Delete all matching keys
        if !keys.is_empty() {
            let _: () = redis::pipe()
                .atomic()
                .del(keys)
                .query_async(&mut conn)
                .await?;
        }
        
        Ok(())
    }
}
```

## Batch Operations

```rust
impl SnippetCache {
    /// Get multiple snippets in batch
    pub async fn get_snippets(&self, ids: &[Uuid]) -> Result<Vec<Option<Entity>>> {
        if ids.is_empty() {
            return Ok(vec![]);
        }
        
        let keys: Vec<String> = ids.iter()
            .map(|id| self.snippet_key(id))
            .collect();
        
        let mut conn = self.redis.get_connection().await?;
        let values: Vec<Option<Vec<u8>>> = conn.get(keys).await?;
        
        let mut results = Vec::new();
        for (i, value) in values.into_iter().enumerate() {
            let entity = if let Some(data) = value {
                Some(serde_json::from_slice(&data)?)
            } else {
                None
            };
            
            // Track access
            self.track_access(&ids[i], entity.is_some()).await?;
            
            results.push(entity);
        }
        
        Ok(results)
    }
    
    /// Store multiple snippets in batch
    pub async fn store_snippets(&self, entities: &[Entity]) -> Result<()> {
        if entities.is_empty() {
            return Ok(());
        }
        
        let mut pipe = redis::pipe();
        pipe.atomic();
        
        for entity in entities {
            let key = self.snippet_key(&entity.id);
            let data = serde_json::to_vec(entity)?;
            let ttl = self.calculate_ttl(entity).await?;
            
            pipe.set_ex(&key, data, ttl.as_secs() as usize);
        }
        
        let mut conn = self.redis.get_connection().await?;
        pipe.query_async(&mut conn).await?;
        
        // Update indices for all
        for entity in entities {
            self.update_indices(entity).await?;
        }
        
        Ok(())
    }
}
```

## Cache Warming

```rust
/// Warm cache with frequently accessed content
pub struct CacheWarmer {
    cache: Arc<SnippetCache>,
    storage: Arc<SnippetStorage>,
}

impl CacheWarmer {
    /// Warm cache on startup
    pub async fn warm_cache(&self) -> Result<()> {
        tracing::info!("Starting cache warming");
        
        // Warm navigation snippets
        self.warm_by_type("navigation", 100).await?;
        
        // Warm recent content
        self.warm_recent(50).await?;
        
        // Warm frequently accessed
        self.warm_frequent(100).await?;
        
        tracing::info!("Cache warming completed");
        Ok(())
    }
    
    /// Warm snippets by type
    async fn warm_by_type(&self, snippet_type: &str, limit: usize) -> Result<()> {
        let snippets = self.storage
            .query()
            .with_type(snippet_type)
            .limit(limit)
            .execute()
            .await?;
        
        self.cache.store_snippets(&snippets).await?;
        
        tracing::debug!("Warmed {} {} snippets", snippets.len(), snippet_type);
        Ok(())
    }
    
    /// Warm recently updated snippets
    async fn warm_recent(&self, limit: usize) -> Result<()> {
        let snippets = self.storage
            .query()
            .order_by("updated_at", true)
            .limit(limit)
            .execute()
            .await?;
        
        self.cache.store_snippets(&snippets).await?;
        
        tracing::debug!("Warmed {} recent snippets", snippets.len());
        Ok(())
    }
    
    /// Warm frequently accessed snippets
    async fn warm_frequent(&self, limit: usize) -> Result<()> {
        // Get access statistics
        let stats = self.cache.get_access_stats(limit).await?;
        
        let ids: Vec<Uuid> = stats.into_iter()
            .map(|(id, _)| id)
            .collect();
        
        // Load from storage
        let snippets = self.storage.get_by_ids(&ids).await?;
        
        // Store in cache
        self.cache.store_snippets(&snippets).await?;
        
        tracing::debug!("Warmed {} frequently accessed snippets", snippets.len());
        Ok(())
    }
}
```

## Access Tracking

```rust
impl SnippetCache {
    /// Track cache access for intelligence
    async fn track_access(&self, id: &Uuid, hit: bool) -> Result<()> {
        if let Some(intel) = &self.intelligence {
            intel.track_access(id, hit).await?;
        }
        
        // Update access counter
        let mut conn = self.redis.get_connection().await?;
        let counter_key = format!("stats:snippet_access:{}", id);
        
        conn.hincrby(&counter_key, "total", 1).await?;
        if hit {
            conn.hincrby(&counter_key, "hits", 1).await?;
        } else {
            conn.hincrby(&counter_key, "misses", 1).await?;
        }
        
        // Update last access time
        conn.hset(&counter_key, "last_access", Utc::now().timestamp()).await?;
        
        // Set expiry
        conn.expire(&counter_key, 86400).await?; // 24 hours
        
        Ok(())
    }
    
    /// Get access statistics
    pub async fn get_access_stats(&self, limit: usize) -> Result<Vec<(Uuid, AccessStats)>> {
        let mut conn = self.redis.get_connection().await?;
        
        // Get all access keys
        let pattern = "stats:snippet_access:*";
        let keys: Vec<String> = conn.keys(pattern).await?;
        
        let mut stats = Vec::new();
        
        for key in keys {
            // Extract UUID from key
            if let Some(id_str) = key.strip_prefix("stats:snippet_access:") {
                if let Ok(id) = Uuid::parse_str(id_str) {
                    // Get stats
                    let data: HashMap<String, i64> = conn.hgetall(&key).await?;
                    
                    let access_stats = AccessStats {
                        total: data.get("total").copied().unwrap_or(0),
                        hits: data.get("hits").copied().unwrap_or(0),
                        misses: data.get("misses").copied().unwrap_or(0),
                        last_access: data.get("last_access")
                            .and_then(|ts| chrono::DateTime::from_timestamp(*ts, 0))
                            .map(|dt| dt.with_timezone(&Utc)),
                    };
                    
                    stats.push((id, access_stats));
                }
            }
        }
        
        // Sort by total accesses
        stats.sort_by(|a, b| b.1.total.cmp(&a.1.total));
        stats.truncate(limit);
        
        Ok(stats)
    }
}

#[derive(Debug, Clone)]
pub struct AccessStats {
    pub total: i64,
    pub hits: i64,
    pub misses: i64,
    pub last_access: Option<chrono::DateTime<Utc>>,
}

impl AccessStats {
    pub fn hit_rate(&self) -> f64 {
        if self.total > 0 {
            self.hits as f64 / self.total as f64
        } else {
            0.0
        }
    }
}
```

## Cache Intelligence

```rust
/// Intelligence for cache optimization
pub struct CacheIntelligence {
    redis: Arc<RedisManager>,
}

impl CacheIntelligence {
    /// Get access pattern for entity
    pub async fn get_access_pattern(&self, id: &Uuid) -> Result<AccessPattern> {
        let mut conn = self.redis.get_connection().await?;
        let counter_key = format!("stats:snippet_access:{}", id);
        
        let data: HashMap<String, i64> = conn.hgetall(&counter_key).await?;
        
        let total = data.get("total").copied().unwrap_or(0);
        let last_access = data.get("last_access")
            .and_then(|ts| chrono::DateTime::from_timestamp(*ts, 0))
            .map(|dt| dt.with_timezone(&Utc))
            .unwrap_or_else(|| Utc::now() - chrono::Duration::days(30));
        
        let age = Utc::now() - last_access;
        
        match (total, age.num_hours()) {
            (t, _) if t > 100 => AccessPattern::Hot,
            (t, h) if t > 10 && h < 24 => AccessPattern::Warm,
            (_, h) if h > 168 => AccessPattern::Stale, // 7 days
            _ => AccessPattern::Cold,
        }
    }
    
    /// Track access for learning
    pub async fn track_access(&self, id: &Uuid, hit: bool) -> Result<()> {
        // This could feed into ML model for predictive caching
        Ok(())
    }
}
```

## Summary

Dieses Cache System bietet:
- **Hierarchical Invalidation** - Parent-Child aware Cache Updates
- **Intelligent TTL** - Access Pattern basierte TTL Berechnung
- **Batch Operations** - Effiziente Multi-Snippet Operations
- **Cache Warming** - Proaktives Laden häufiger Content
- **Access Tracking** - Statistiken für Cache Optimization

Redis Cache macht ReedCMS blitzschnell.
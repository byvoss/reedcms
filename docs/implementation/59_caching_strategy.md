# Caching Strategy

## Overview

Caching Strategy Implementation für ReedCMS. Multi-Layer Caching mit Redis, In-Memory und CDN Integration.

## Cache Manager

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use moka::future::Cache as MokaCache;

/// Main cache management service
pub struct CacheManager {
    layers: Vec<Arc<dyn CacheLayer>>,
    invalidation: Arc<InvalidationManager>,
    metrics: Arc<CacheMetrics>,
    config: CacheConfig,
}

impl CacheManager {
    pub fn new(config: CacheConfig) -> Result<Self> {
        let mut layers = Vec::new();
        
        // L1: In-memory cache
        if config.enable_memory_cache {
            layers.push(Arc::new(MemoryCache::new(config.memory.clone())) as Arc<dyn CacheLayer>);
        }
        
        // L2: Redis cache
        if config.enable_redis_cache {
            layers.push(Arc::new(RedisCache::new(config.redis.clone())?) as Arc<dyn CacheLayer>);
        }
        
        // L3: CDN cache
        if config.enable_cdn_cache {
            layers.push(Arc::new(CdnCache::new(config.cdn.clone())?) as Arc<dyn CacheLayer>);
        }
        
        let invalidation = Arc::new(InvalidationManager::new());
        let metrics = Arc::new(CacheMetrics::new());
        
        Ok(Self {
            layers,
            invalidation,
            metrics,
            config,
        })
    }
    
    /// Get value from cache
    pub async fn get<T>(&self, key: &str) -> Result<Option<T>>
    where
        T: CacheValue,
    {
        let start = std::time::Instant::now();
        
        // Try each layer in order
        for (index, layer) in self.layers.iter().enumerate() {
            if let Some(value) = layer.get::<T>(key).await? {
                // Cache hit
                self.metrics.record_hit(index, start.elapsed()).await;
                
                // Promote to higher layers
                self.promote_value(key, &value, index).await?;
                
                return Ok(Some(value));
            }
        }
        
        // Cache miss
        self.metrics.record_miss(start.elapsed()).await;
        Ok(None)
    }
    
    /// Set value in cache
    pub async fn set<T>(&self, key: &str, value: &T, ttl: Option<u64>) -> Result<()>
    where
        T: CacheValue,
    {
        // Determine cache strategy
        let strategy = self.get_cache_strategy(key, value);
        
        // Write to appropriate layers
        for (index, layer) in self.layers.iter().enumerate() {
            if strategy.should_cache_in_layer(index) {
                layer.set(key, value, ttl).await?;
            }
        }
        
        // Track for invalidation
        self.invalidation.track_key(key, &strategy.tags).await;
        
        Ok(())
    }
    
    /// Delete value from cache
    pub async fn delete(&self, key: &str) -> Result<()> {
        for layer in &self.layers {
            layer.delete(key).await?;
        }
        
        self.invalidation.remove_key(key).await;
        
        Ok(())
    }
    
    /// Invalidate by tags
    pub async fn invalidate_by_tags(&self, tags: &[String]) -> Result<u64> {
        let keys = self.invalidation.get_keys_by_tags(tags).await?;
        let mut count = 0;
        
        for key in keys {
            self.delete(&key).await?;
            count += 1;
        }
        
        Ok(count)
    }
    
    /// Get or compute value
    pub async fn get_or_compute<T, F, Fut>(
        &self,
        key: &str,
        compute_fn: F,
        ttl: Option<u64>,
    ) -> Result<T>
    where
        T: CacheValue,
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<T>>,
    {
        // Try cache first
        if let Some(value) = self.get::<T>(key).await? {
            return Ok(value);
        }
        
        // Compute value
        let value = compute_fn().await?;
        
        // Store in cache
        self.set(key, &value, ttl).await?;
        
        Ok(value)
    }
    
    /// Promote value to higher cache layers
    async fn promote_value<T>(&self, key: &str, value: &T, found_at: usize) -> Result<()>
    where
        T: CacheValue,
    {
        // Promote to all layers above where it was found
        for i in 0..found_at {
            self.layers[i].set(key, value, None).await?;
        }
        
        Ok(())
    }
    
    /// Get cache strategy for key/value
    fn get_cache_strategy<T>(&self, key: &str, value: &T) -> CacheStrategy
    where
        T: CacheValue,
    {
        // Determine strategy based on key pattern and value size
        let size = value.size_hint();
        
        if key.starts_with("session:") {
            CacheStrategy {
                layers: vec![0, 1], // Memory and Redis only
                tags: vec!["session".to_string()],
                ttl_override: Some(3600), // 1 hour
            }
        } else if key.starts_with("content:") {
            CacheStrategy {
                layers: vec![0, 1, 2], // All layers
                tags: vec!["content".to_string()],
                ttl_override: None,
            }
        } else if size > 1_000_000 {
            // Large objects go to Redis/CDN only
            CacheStrategy {
                layers: vec![1, 2],
                tags: vec!["large".to_string()],
                ttl_override: None,
            }
        } else {
            // Default strategy
            CacheStrategy {
                layers: vec![0, 1],
                tags: vec!["default".to_string()],
                ttl_override: None,
            }
        }
    }
}

/// Cache layer trait
#[async_trait]
pub trait CacheLayer: Send + Sync {
    async fn get<T>(&self, key: &str) -> Result<Option<T>>
    where
        T: CacheValue;
    
    async fn set<T>(&self, key: &str, value: &T, ttl: Option<u64>) -> Result<()>
    where
        T: CacheValue;
    
    async fn delete(&self, key: &str) -> Result<()>;
    
    async fn clear(&self) -> Result<()>;
}

/// Cache value trait
pub trait CacheValue: Serialize + DeserializeOwned + Send + Sync {
    fn size_hint(&self) -> usize {
        // Default implementation using serialization
        serde_json::to_vec(self).map(|v| v.len()).unwrap_or(0)
    }
}

/// Cache strategy
struct CacheStrategy {
    layers: Vec<usize>,
    tags: Vec<String>,
    ttl_override: Option<u64>,
}

impl CacheStrategy {
    fn should_cache_in_layer(&self, layer_index: usize) -> bool {
        self.layers.contains(&layer_index)
    }
}
```

## Memory Cache Layer

```rust
/// In-memory cache using Moka
pub struct MemoryCache {
    cache: MokaCache<String, Vec<u8>>,
    config: MemoryCacheConfig,
}

impl MemoryCache {
    pub fn new(config: MemoryCacheConfig) -> Self {
        let cache = MokaCache::builder()
            .max_capacity(config.max_capacity)
            .time_to_live(Duration::from_secs(config.default_ttl))
            .time_to_idle(Duration::from_secs(config.time_to_idle))
            .eviction_listener(|k, v, cause| {
                debug!("Evicted key {} due to {:?}", k, cause);
            })
            .build();
        
        Self { cache, config }
    }
}

#[async_trait]
impl CacheLayer for MemoryCache {
    async fn get<T>(&self, key: &str) -> Result<Option<T>>
    where
        T: CacheValue,
    {
        if let Some(bytes) = self.cache.get(key).await {
            let value: T = bincode::deserialize(&bytes)?;
            Ok(Some(value))
        } else {
            Ok(None)
        }
    }
    
    async fn set<T>(&self, key: &str, value: &T, ttl: Option<u64>) -> Result<()>
    where
        T: CacheValue,
    {
        let bytes = bincode::serialize(value)?;
        
        if let Some(ttl) = ttl {
            self.cache
                .insert_with_ttl(key.to_string(), bytes, Duration::from_secs(ttl))
                .await;
        } else {
            self.cache.insert(key.to_string(), bytes).await;
        }
        
        Ok(())
    }
    
    async fn delete(&self, key: &str) -> Result<()> {
        self.cache.remove(key).await;
        Ok(())
    }
    
    async fn clear(&self) -> Result<()> {
        self.cache.invalidate_all().await;
        Ok(())
    }
}
```

## Redis Cache Layer

```rust
/// Redis cache layer
pub struct RedisCache {
    pool: Arc<RedisPool>,
    config: RedisCacheConfig,
}

impl RedisCache {
    pub fn new(config: RedisCacheConfig) -> Result<Self> {
        let pool = RedisPool::new(&config.connection)?;
        
        Ok(Self {
            pool: Arc::new(pool),
            config,
        })
    }
}

#[async_trait]
impl CacheLayer for RedisCache {
    async fn get<T>(&self, key: &str) -> Result<Option<T>>
    where
        T: CacheValue,
    {
        let mut conn = self.pool.get().await?;
        let prefixed_key = format!("{}:{}", self.config.key_prefix, key);
        
        let bytes: Option<Vec<u8>> = conn.get(&prefixed_key).await?;
        
        if let Some(bytes) = bytes {
            let value: T = bincode::deserialize(&bytes)?;
            Ok(Some(value))
        } else {
            Ok(None)
        }
    }
    
    async fn set<T>(&self, key: &str, value: &T, ttl: Option<u64>) -> Result<()>
    where
        T: CacheValue,
    {
        let mut conn = self.pool.get().await?;
        let prefixed_key = format!("{}:{}", self.config.key_prefix, key);
        let bytes = bincode::serialize(value)?;
        
        if let Some(ttl) = ttl {
            conn.set_ex(&prefixed_key, bytes, ttl as usize).await?;
        } else {
            conn.set(&prefixed_key, bytes).await?;
        }
        
        Ok(())
    }
    
    async fn delete(&self, key: &str) -> Result<()> {
        let mut conn = self.pool.get().await?;
        let prefixed_key = format!("{}:{}", self.config.key_prefix, key);
        
        conn.del(&prefixed_key).await?;
        
        Ok(())
    }
    
    async fn clear(&self) -> Result<()> {
        let mut conn = self.pool.get().await?;
        let pattern = format!("{}:*", self.config.key_prefix);
        
        let keys: Vec<String> = conn.keys(&pattern).await?;
        
        if !keys.is_empty() {
            conn.del(keys).await?;
        }
        
        Ok(())
    }
}
```

## CDN Cache Integration

```rust
/// CDN cache layer
pub struct CdnCache {
    client: Arc<CdnClient>,
    config: CdnCacheConfig,
}

impl CdnCache {
    pub fn new(config: CdnCacheConfig) -> Result<Self> {
        let client = match config.provider {
            CdnProvider::Cloudflare => {
                Arc::new(CloudflareClient::new(&config)?) as Arc<dyn CdnClient>
            }
            CdnProvider::Fastly => {
                Arc::new(FastlyClient::new(&config)?) as Arc<dyn CdnClient>
            }
            CdnProvider::CloudFront => {
                Arc::new(CloudFrontClient::new(&config)?) as Arc<dyn CdnClient>
            }
        };
        
        Ok(Self { client, config })
    }
}

#[async_trait]
impl CacheLayer for CdnCache {
    async fn get<T>(&self, key: &str) -> Result<Option<T>>
    where
        T: CacheValue,
    {
        // CDN typically doesn't support direct key-value get
        // This is mainly for cache warming and invalidation
        Ok(None)
    }
    
    async fn set<T>(&self, key: &str, value: &T, ttl: Option<u64>) -> Result<()>
    where
        T: CacheValue,
    {
        // Warm CDN cache by making request
        if self.config.enable_warming {
            let url = self.build_cdn_url(key);
            self.client.warm_cache(&url).await?;
        }
        
        Ok(())
    }
    
    async fn delete(&self, key: &str) -> Result<()> {
        // Purge from CDN
        let url = self.build_cdn_url(key);
        self.client.purge(&url).await?;
        
        Ok(())
    }
    
    async fn clear(&self) -> Result<()> {
        // Purge all CDN cache
        self.client.purge_all().await?;
        
        Ok(())
    }
}

/// CDN client trait
#[async_trait]
trait CdnClient: Send + Sync {
    async fn warm_cache(&self, url: &str) -> Result<()>;
    async fn purge(&self, url: &str) -> Result<()>;
    async fn purge_all(&self) -> Result<()>;
}
```

## Cache Invalidation

```rust
/// Cache invalidation manager
pub struct InvalidationManager {
    tag_index: Arc<RwLock<HashMap<String, HashSet<String>>>>,
    key_tags: Arc<RwLock<HashMap<String, HashSet<String>>>>,
}

impl InvalidationManager {
    pub fn new() -> Self {
        Self {
            tag_index: Arc::new(RwLock::new(HashMap::new())),
            key_tags: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Track key with tags
    pub async fn track_key(&self, key: &str, tags: &[String]) {
        let mut tag_index = self.tag_index.write().await;
        let mut key_tags = self.key_tags.write().await;
        
        // Update tag index
        for tag in tags {
            tag_index
                .entry(tag.clone())
                .or_insert_with(HashSet::new)
                .insert(key.to_string());
        }
        
        // Update key tags
        key_tags.insert(
            key.to_string(),
            tags.iter().cloned().collect(),
        );
    }
    
    /// Get keys by tags
    pub async fn get_keys_by_tags(&self, tags: &[String]) -> Result<Vec<String>> {
        let tag_index = self.tag_index.read().await;
        let mut keys = HashSet::new();
        
        for tag in tags {
            if let Some(tag_keys) = tag_index.get(tag) {
                keys.extend(tag_keys.clone());
            }
        }
        
        Ok(keys.into_iter().collect())
    }
    
    /// Remove key tracking
    pub async fn remove_key(&self, key: &str) {
        let mut tag_index = self.tag_index.write().await;
        let mut key_tags = self.key_tags.write().await;
        
        // Get tags for key
        if let Some(tags) = key_tags.remove(key) {
            // Remove from tag index
            for tag in tags {
                if let Some(tag_keys) = tag_index.get_mut(&tag) {
                    tag_keys.remove(key);
                    
                    // Remove tag if no more keys
                    if tag_keys.is_empty() {
                        tag_index.remove(&tag);
                    }
                }
            }
        }
    }
}
```

## Cache Warming

```rust
/// Cache warming service
pub struct CacheWarmer {
    cache_manager: Arc<CacheManager>,
    warming_queue: Arc<WarmingQueue>,
    config: WarmingConfig,
}

impl CacheWarmer {
    /// Start warming process
    pub fn start(self: Arc<Self>) {
        tokio::spawn(async move {
            let mut interval = tokio::time::interval(self.config.interval);
            
            loop {
                interval.tick().await;
                
                if let Err(e) = self.warm_cache().await {
                    error!("Cache warming error: {}", e);
                }
            }
        });
    }
    
    /// Warm cache
    async fn warm_cache(&self) -> Result<()> {
        let tasks = self.warming_queue.get_tasks().await?;
        
        for task in tasks {
            match task {
                WarmingTask::Content { content_type } => {
                    self.warm_content(&content_type).await?;
                }
                WarmingTask::Query { query } => {
                    self.warm_query(&query).await?;
                }
                WarmingTask::Custom { handler } => {
                    handler.warm(&self.cache_manager).await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Warm content cache
    async fn warm_content(&self, content_type: &str) -> Result<()> {
        // Get frequently accessed content
        let items = self.get_popular_content(content_type).await?;
        
        for item in items {
            let key = format!("content:{}:{}", content_type, item.id);
            
            // Compute and cache
            self.cache_manager
                .get_or_compute(
                    &key,
                    || async { self.load_content(&item.id).await },
                    Some(3600),
                )
                .await?;
        }
        
        Ok(())
    }
}

/// Warming task
enum WarmingTask {
    Content { content_type: String },
    Query { query: String },
    Custom { handler: Box<dyn WarmingHandler> },
}

/// Warming handler trait
#[async_trait]
trait WarmingHandler: Send + Sync {
    async fn warm(&self, cache: &CacheManager) -> Result<()>;
}
```

## Cache Metrics

```rust
/// Cache metrics collector
pub struct CacheMetrics {
    hits: Arc<AtomicU64>,
    misses: Arc<AtomicU64>,
    layer_hits: Arc<RwLock<Vec<LayerMetrics>>>,
    response_times: Arc<RwLock<Histogram>>,
}

impl CacheMetrics {
    pub fn new() -> Self {
        Self {
            hits: Arc::new(AtomicU64::new(0)),
            misses: Arc::new(AtomicU64::new(0)),
            layer_hits: Arc::new(RwLock::new(vec![
                LayerMetrics::new("memory"),
                LayerMetrics::new("redis"),
                LayerMetrics::new("cdn"),
            ])),
            response_times: Arc::new(RwLock::new(Histogram::new())),
        }
    }
    
    /// Record cache hit
    pub async fn record_hit(&self, layer: usize, duration: Duration) {
        self.hits.fetch_add(1, Ordering::Relaxed);
        
        let mut layer_hits = self.layer_hits.write().await;
        if let Some(metrics) = layer_hits.get_mut(layer) {
            metrics.record_hit();
        }
        
        self.response_times.write().await.record(duration.as_micros() as u64);
    }
    
    /// Record cache miss
    pub async fn record_miss(&self, duration: Duration) {
        self.misses.fetch_add(1, Ordering::Relaxed);
        self.response_times.write().await.record(duration.as_micros() as u64);
    }
    
    /// Get hit ratio
    pub fn hit_ratio(&self) -> f64 {
        let hits = self.hits.load(Ordering::Relaxed);
        let misses = self.misses.load(Ordering::Relaxed);
        let total = hits + misses;
        
        if total == 0 {
            0.0
        } else {
            hits as f64 / total as f64
        }
    }
    
    /// Export metrics
    pub async fn export(&self) -> CacheStats {
        let layer_hits = self.layer_hits.read().await;
        let response_times = self.response_times.read().await;
        
        CacheStats {
            total_hits: self.hits.load(Ordering::Relaxed),
            total_misses: self.misses.load(Ordering::Relaxed),
            hit_ratio: self.hit_ratio(),
            layer_stats: layer_hits.iter().map(|m| m.export()).collect(),
            avg_response_time_us: response_times.mean(),
            p95_response_time_us: response_times.percentile(0.95),
            p99_response_time_us: response_times.percentile(0.99),
        }
    }
}

/// Layer metrics
struct LayerMetrics {
    name: String,
    hits: AtomicU64,
    hit_ratio: AtomicU64,
}

impl LayerMetrics {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            hits: AtomicU64::new(0),
            hit_ratio: AtomicU64::new(0),
        }
    }
    
    fn record_hit(&self) {
        self.hits.fetch_add(1, Ordering::Relaxed);
    }
    
    fn export(&self) -> LayerStats {
        LayerStats {
            name: self.name.clone(),
            hits: self.hits.load(Ordering::Relaxed),
        }
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CacheConfig {
    pub enable_memory_cache: bool,
    pub enable_redis_cache: bool,
    pub enable_cdn_cache: bool,
    pub memory: MemoryCacheConfig,
    pub redis: RedisCacheConfig,
    pub cdn: CdnCacheConfig,
    pub invalidation: InvalidationConfig,
    pub warming: WarmingConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MemoryCacheConfig {
    pub max_capacity: u64,
    pub default_ttl: u64,
    pub time_to_idle: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RedisCacheConfig {
    pub connection: RedisConnectionConfig,
    pub key_prefix: String,
    pub default_ttl: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CdnCacheConfig {
    pub provider: CdnProvider,
    pub api_key: String,
    pub zone_id: String,
    pub base_url: String,
    pub enable_warming: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum CdnProvider {
    Cloudflare,
    Fastly,
    CloudFront,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct InvalidationConfig {
    pub track_tags: bool,
    pub max_tags_per_key: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WarmingConfig {
    pub enabled: bool,
    pub interval: Duration,
    pub batch_size: usize,
}

impl Default for CacheConfig {
    fn default() -> Self {
        Self {
            enable_memory_cache: true,
            enable_redis_cache: true,
            enable_cdn_cache: false,
            memory: MemoryCacheConfig {
                max_capacity: 1000,
                default_ttl: 300,
                time_to_idle: 60,
            },
            redis: RedisCacheConfig {
                connection: RedisConnectionConfig::default(),
                key_prefix: "reed:cache".to_string(),
                default_ttl: 3600,
            },
            cdn: CdnCacheConfig {
                provider: CdnProvider::Cloudflare,
                api_key: String::new(),
                zone_id: String::new(),
                base_url: String::new(),
                enable_warming: false,
            },
            invalidation: InvalidationConfig {
                track_tags: true,
                max_tags_per_key: 10,
            },
            warming: WarmingConfig {
                enabled: false,
                interval: Duration::from_secs(300),
                batch_size: 100,
            },
        }
    }
}
```

## Summary

Diese Caching Strategy bietet:
- **Multi-Layer Caching** - Memory, Redis, CDN
- **Smart Invalidation** - Tag-based Invalidation
- **Cache Warming** - Proactive Cache Population
- **Flexible Strategies** - Per-Key Cache Rules
- **Performance Metrics** - Hit Ratios, Response Times
- **CDN Integration** - Cloudflare, Fastly, CloudFront

Comprehensive Caching für optimal Performance.
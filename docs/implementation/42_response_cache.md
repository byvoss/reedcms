# Response Caching

## Overview

Response Caching System für ReedCMS Server. Intelligent Caching mit Context-awareness, Vary Headers und automatischer Invalidation.

## Cache Core

```rust
use axum::{
    response::{Response, IntoResponse},
    http::{StatusCode, HeaderMap, HeaderValue},
    body::Bytes,
};
use sha2::{Sha256, Digest};

/// Response cache manager
pub struct ResponseCache {
    redis: Arc<RedisConnection>,
    default_ttl: u64,
    cache_config: CacheConfig,
    invalidator: Arc<CacheInvalidator>,
}

impl ResponseCache {
    pub fn new(
        redis: Arc<RedisConnection>,
        default_ttl: u64,
    ) -> Self {
        let invalidator = Arc::new(CacheInvalidator::new(redis.clone()));
        
        Self {
            redis,
            default_ttl,
            cache_config: CacheConfig::default(),
            invalidator,
        }
    }
    
    /// Get cached response
    pub async fn get(
        &self,
        key: &str,
    ) -> Result<Option<CachedResponse>> {
        let data: Option<Vec<u8>> = self.redis
            .get(&format!("response:{}", key))
            .await?;
        
        if let Some(data) = data {
            let cached: CachedResponse = bincode::deserialize(&data)?;
            
            // Check if still valid
            if cached.is_valid() {
                // Update hit counter
                self.redis
                    .incr(&format!("response:hits:{}", key))
                    .await?;
                
                return Ok(Some(cached));
            } else {
                // Remove expired entry
                self.redis
                    .del(&format!("response:{}", key))
                    .await?;
            }
        }
        
        Ok(None)
    }
    
    /// Store response in cache
    pub async fn set(
        &self,
        key: &str,
        response: &Response,
        ttl: Option<u64>,
    ) -> Result<()> {
        // Check if response is cacheable
        if !self.is_cacheable(response) {
            return Ok(());
        }
        
        // Extract response data
        let cached = CachedResponse::from_response(response).await?;
        
        // Serialize
        let data = bincode::serialize(&cached)?;
        
        // Store with TTL
        let ttl = ttl.unwrap_or(self.default_ttl);
        self.redis
            .setex(
                &format!("response:{}", key),
                data,
                ttl,
            )
            .await?;
        
        // Track cache entry for invalidation
        self.invalidator
            .track_entry(key, &cached.dependencies)
            .await?;
        
        Ok(())
    }
    
    /// Check if response is cacheable
    fn is_cacheable(&self, response: &Response) -> bool {
        // Only cache successful responses
        if !response.status().is_success() {
            return false;
        }
        
        // Check Cache-Control headers
        if let Some(cache_control) = response.headers().get("Cache-Control") {
            if let Ok(value) = cache_control.to_str() {
                if value.contains("no-cache") || 
                   value.contains("no-store") ||
                   value.contains("private") {
                    return false;
                }
            }
        }
        
        true
    }
}

/// Cached response data
#[derive(Debug, Serialize, Deserialize)]
pub struct CachedResponse {
    pub status: u16,
    pub headers: Vec<(String, String)>,
    pub body: Vec<u8>,
    pub created_at: i64,
    pub expires_at: Option<i64>,
    pub etag: String,
    pub vary: Vec<String>,
    pub dependencies: Vec<String>,
}

impl CachedResponse {
    /// Create from Axum response
    pub async fn from_response(response: &Response) -> Result<Self> {
        let status = response.status().as_u16();
        
        // Extract headers
        let headers: Vec<(String, String)> = response.headers()
            .iter()
            .map(|(k, v)| {
                (
                    k.to_string(),
                    v.to_str().unwrap_or("").to_string(),
                )
            })
            .collect();
        
        // Extract body
        let body = hyper::body::to_bytes(response.body())
            .await?
            .to_vec();
        
        // Generate ETag
        let etag = Self::generate_etag(&body);
        
        // Extract Vary headers
        let vary = Self::extract_vary_headers(response.headers());
        
        // Extract dependencies from custom header
        let dependencies = Self::extract_dependencies(response.headers());
        
        Ok(Self {
            status,
            headers,
            body,
            created_at: chrono::Utc::now().timestamp(),
            expires_at: Self::calculate_expiry(response.headers()),
            etag,
            vary,
            dependencies,
        })
    }
    
    /// Check if cache entry is still valid
    pub fn is_valid(&self) -> bool {
        if let Some(expires) = self.expires_at {
            chrono::Utc::now().timestamp() < expires
        } else {
            true
        }
    }
    
    /// Convert back to Axum response
    pub fn to_response(self) -> Response {
        let mut response = Response::builder()
            .status(self.status);
        
        // Add headers
        for (key, value) in self.headers {
            response = response.header(key, value);
        }
        
        // Add cache headers
        response = response
            .header("X-Cache", "HIT")
            .header("ETag", self.etag)
            .header("Age", self.age().to_string());
        
        response
            .body(self.body.into())
            .unwrap()
    }
    
    /// Calculate age of cache entry
    fn age(&self) -> i64 {
        chrono::Utc::now().timestamp() - self.created_at
    }
    
    /// Generate ETag for content
    fn generate_etag(content: &[u8]) -> String {
        let mut hasher = Sha256::new();
        hasher.update(content);
        format!("\"{:x}\"", hasher.finalize())
    }
}
```

## Cache Key Generation

```rust
/// Generate cache keys with context awareness
pub struct CacheKeyGenerator;

impl CacheKeyGenerator {
    /// Generate cache key for request
    pub fn generate(
        uri: &Uri,
        method: &Method,
        headers: &HeaderMap,
        vary: &[String],
    ) -> String {
        let mut hasher = Sha256::new();
        
        // Base key from method and URI
        hasher.update(method.as_str());
        hasher.update(uri.to_string());
        
        // Add vary headers
        for header_name in vary {
            if let Some(value) = headers.get(header_name) {
                hasher.update(header_name);
                hasher.update(value.as_bytes());
            }
        }
        
        format!("{:x}", hasher.finalize())
    }
    
    /// Generate key with context
    pub fn generate_with_context(
        uri: &Uri,
        method: &Method,
        context: &RequestContext,
    ) -> String {
        let mut hasher = Sha256::new();
        
        // Base key
        hasher.update(method.as_str());
        hasher.update(uri.to_string());
        
        // Add context elements
        hasher.update(&context.locale);
        hasher.update(&context.theme);
        
        // Add device type for responsive caching
        hasher.update(format!("{:?}", context.device.device_type));
        
        // Add user role if authenticated
        if let Some(ref user) = context.user {
            hasher.update("auth");
            for role in &user.roles {
                hasher.update(role);
            }
        } else {
            hasher.update("anon");
        }
        
        format!("{:x}", hasher.finalize())
    }
}
```

## Cache Invalidation

```rust
/// Cache invalidation manager
pub struct CacheInvalidator {
    redis: Arc<RedisConnection>,
}

impl CacheInvalidator {
    pub fn new(redis: Arc<RedisConnection>) -> Self {
        Self { redis }
    }
    
    /// Track cache entry with dependencies
    pub async fn track_entry(
        &self,
        cache_key: &str,
        dependencies: &[String],
    ) -> Result<()> {
        // Store reverse mapping for each dependency
        for dep in dependencies {
            self.redis
                .sadd(
                    &format!("cache:dep:{}", dep),
                    cache_key,
                )
                .await?;
        }
        
        Ok(())
    }
    
    /// Invalidate by dependency
    pub async fn invalidate_by_dependency(
        &self,
        dependency: &str,
    ) -> Result<usize> {
        let key = format!("cache:dep:{}", dependency);
        
        // Get all cache keys with this dependency
        let cache_keys: Vec<String> = self.redis
            .smembers(&key)
            .await?;
        
        let mut invalidated = 0;
        
        // Delete each cache entry
        for cache_key in &cache_keys {
            if self.redis.del(&format!("response:{}", cache_key)).await? {
                invalidated += 1;
            }
        }
        
        // Clean up dependency set
        self.redis.del(&key).await?;
        
        Ok(invalidated)
    }
    
    /// Invalidate by pattern
    pub async fn invalidate_by_pattern(
        &self,
        pattern: &str,
    ) -> Result<usize> {
        let keys: Vec<String> = self.redis
            .keys(&format!("response:{}", pattern))
            .await?;
        
        let mut invalidated = 0;
        
        for key in keys {
            if self.redis.del(&key).await? {
                invalidated += 1;
            }
        }
        
        Ok(invalidated)
    }
    
    /// Invalidate by entity change
    pub async fn invalidate_by_entity(
        &self,
        entity_id: Uuid,
        entity_type: &str,
    ) -> Result<()> {
        // Invalidate direct entity cache
        self.invalidate_by_dependency(&format!("entity:{}", entity_id)).await?;
        
        // Invalidate type-based caches
        self.invalidate_by_dependency(&format!("type:{}", entity_type)).await?;
        
        // Invalidate parent/child relationships
        self.invalidate_by_dependency(&format!("children:{}", entity_id)).await?;
        
        Ok(())
    }
}
```

## Cache Middleware

```rust
/// Response caching middleware
pub async fn cache_middleware(
    State(ctx): State<Arc<ServerContext>>,
    uri: Uri,
    method: Method,
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Only cache GET requests
    if method != Method::GET {
        return Ok(next.run(request).await);
    }
    
    // Get request context
    let req_context = request.extensions()
        .get::<RequestContext>()
        .cloned()
        .ok_or(StatusCode::INTERNAL_SERVER_ERROR)?;
    
    // Generate cache key
    let cache_key = CacheKeyGenerator::generate_with_context(
        &uri,
        &method,
        &req_context,
    );
    
    // Try to get from cache
    if let Ok(Some(cached)) = ctx.cache.get(&cache_key).await {
        // Check conditional headers
        if let Some(if_none_match) = headers.get("If-None-Match") {
            if if_none_match.to_str().ok() == Some(&cached.etag) {
                return Ok(StatusCode::NOT_MODIFIED.into_response());
            }
        }
        
        return Ok(cached.to_response());
    }
    
    // Process request
    let response = next.run(request).await;
    
    // Cache successful responses
    if response.status().is_success() {
        // Determine TTL from response headers or config
        let ttl = extract_cache_ttl(response.headers())
            .or_else(|| determine_ttl_by_path(&uri.path()));
        
        // Store in cache
        if let Err(e) = ctx.cache.set(&cache_key, &response, ttl).await {
            warn!("Failed to cache response: {}", e);
        }
    }
    
    Ok(response)
}

/// Extract TTL from Cache-Control header
fn extract_cache_ttl(headers: &HeaderMap) -> Option<u64> {
    headers
        .get("Cache-Control")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| {
            // Parse max-age=seconds
            v.split(',')
                .find(|part| part.trim().starts_with("max-age="))
                .and_then(|part| {
                    part.trim()
                        .strip_prefix("max-age=")
                        .and_then(|age| age.parse().ok())
                })
        })
}

/// Determine TTL based on path patterns
fn determine_ttl_by_path(path: &str) -> Option<u64> {
    if path.starts_with("/api/") {
        Some(60) // 1 minute for API
    } else if path.starts_with("/assets/") {
        Some(86400) // 1 day for assets
    } else if path.starts_with("/content/") {
        Some(300) // 5 minutes for content
    } else {
        None
    }
}
```

## Cache Warming

```rust
/// Cache warming service
pub struct CacheWarmer {
    cache: Arc<ResponseCache>,
    client: reqwest::Client,
    base_url: String,
}

impl CacheWarmer {
    pub fn new(
        cache: Arc<ResponseCache>,
        base_url: String,
    ) -> Self {
        Self {
            cache,
            client: reqwest::Client::new(),
            base_url,
        }
    }
    
    /// Warm cache for specific paths
    pub async fn warm_paths(&self, paths: Vec<String>) -> Result<()> {
        let tasks: Vec<_> = paths.into_iter()
            .map(|path| self.warm_single_path(path))
            .collect();
        
        futures::future::try_join_all(tasks).await?;
        Ok(())
    }
    
    /// Warm single path
    async fn warm_single_path(&self, path: String) -> Result<()> {
        let url = format!("{}{}", self.base_url, path);
        
        // Make request with cache warming header
        let response = self.client
            .get(&url)
            .header("X-Cache-Warm", "true")
            .send()
            .await?;
        
        if response.status().is_success() {
            info!("Warmed cache for: {}", path);
        } else {
            warn!("Failed to warm cache for: {} ({})", path, response.status());
        }
        
        Ok(())
    }
    
    /// Warm navigation pages
    pub async fn warm_navigation(
        &self,
        snippet_manager: &SnippetManager,
    ) -> Result<()> {
        let nav_items = snippet_manager
            .get_navigation()
            .await?;
        
        let paths: Vec<String> = nav_items.into_iter()
            .filter_map(|item| item.url)
            .collect();
        
        self.warm_paths(paths).await
    }
}
```

## Cache Configuration

```rust
#[derive(Debug, Clone)]
pub struct CacheConfig {
    /// Enable response caching
    pub enabled: bool,
    
    /// Default TTL in seconds
    pub default_ttl: u64,
    
    /// Max cacheable response size
    pub max_size: usize,
    
    /// Cache key vary headers
    pub vary_headers: Vec<String>,
    
    /// Paths to never cache
    pub exclude_paths: Vec<String>,
    
    /// Paths to always cache
    pub include_paths: Vec<String>,
}

impl Default for CacheConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            default_ttl: 300, // 5 minutes
            max_size: 10 * 1024 * 1024, // 10MB
            vary_headers: vec![
                "Accept".to_string(),
                "Accept-Language".to_string(),
                "Accept-Encoding".to_string(),
            ],
            exclude_paths: vec![
                "/admin".to_string(),
                "/api/auth".to_string(),
            ],
            include_paths: vec![
                "/".to_string(),
                "/content".to_string(),
            ],
        }
    }
}
```

## Summary

Dieses Response Cache System bietet:
- **Smart Caching** - Context-aware Cache Keys
- **Cache Invalidation** - Dependency-based Invalidation
- **Conditional Requests** - ETag und If-None-Match Support
- **Cache Warming** - Proactive Cache Population
- **Flexible Configuration** - Path-based Rules
- **Performance Metrics** - Hit/Miss Tracking

Intelligent Caching für optimale Performance.
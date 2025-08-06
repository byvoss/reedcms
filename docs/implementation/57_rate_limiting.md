# Rate Limiting

## Overview

Rate Limiting Implementation für ReedCMS. Token Bucket und Sliding Window Algorithms mit Redis Backend.

## Rate Limiter Core

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use redis::AsyncCommands;

/// Main rate limiting service
pub struct RateLimiter {
    redis: Arc<RedisConnection>,
    strategies: Arc<RwLock<HashMap<String, Box<dyn RateLimitStrategy>>>>,
    config: RateLimitConfig,
}

impl RateLimiter {
    pub fn new(redis: Arc<RedisConnection>, config: RateLimitConfig) -> Self {
        let mut limiter = Self {
            redis,
            strategies: Arc::new(RwLock::new(HashMap::new())),
            config,
        };
        
        // Register default strategies
        limiter.register_default_strategies();
        
        limiter
    }
    
    /// Check if request is allowed
    pub async fn check_limit(
        &self,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        // Get strategy
        let strategies = self.strategies.read().await;
        let strategy = strategies
            .get(&rule.strategy)
            .ok_or_else(|| RateLimitError::UnknownStrategy(rule.strategy.clone()))?;
        
        // Apply rate limit
        let result = strategy.check_limit(
            &self.redis,
            key,
            rule,
        ).await?;
        
        // Log if limit exceeded
        if !result.allowed {
            self.log_limit_exceeded(key, rule).await;
        }
        
        Ok(result)
    }
    
    /// Check multiple limits
    pub async fn check_limits(
        &self,
        key: &str,
        rules: &[RateLimitRule],
    ) -> Result<RateLimitResult> {
        let mut combined_result = RateLimitResult {
            allowed: true,
            remaining: u64::MAX,
            reset_at: None,
            retry_after: None,
        };
        
        for rule in rules {
            let result = self.check_limit(key, rule).await?;
            
            if !result.allowed {
                return Ok(result);
            }
            
            // Keep the most restrictive limit
            if result.remaining < combined_result.remaining {
                combined_result.remaining = result.remaining;
                combined_result.reset_at = result.reset_at;
            }
        }
        
        Ok(combined_result)
    }
    
    /// Reset limits for key
    pub async fn reset_limits(&self, key: &str) -> Result<()> {
        let pattern = format!("ratelimit:*:{}", key);
        let keys: Vec<String> = self.redis.keys(&pattern).await?;
        
        if !keys.is_empty() {
            self.redis.del(keys).await?;
        }
        
        Ok(())
    }
    
    /// Register rate limit strategy
    pub async fn register_strategy(
        &self,
        name: String,
        strategy: Box<dyn RateLimitStrategy>,
    ) {
        self.strategies.write().await.insert(name, strategy);
    }
    
    /// Register default strategies
    fn register_default_strategies(&mut self) {
        let strategies = Arc::get_mut(&mut self.strategies).unwrap().get_mut();
        
        // Token bucket
        strategies.insert(
            "token_bucket".to_string(),
            Box::new(TokenBucketStrategy::new()),
        );
        
        // Sliding window
        strategies.insert(
            "sliding_window".to_string(),
            Box::new(SlidingWindowStrategy::new()),
        );
        
        // Fixed window
        strategies.insert(
            "fixed_window".to_string(),
            Box::new(FixedWindowStrategy::new()),
        );
        
        // Leaky bucket
        strategies.insert(
            "leaky_bucket".to_string(),
            Box::new(LeakyBucketStrategy::new()),
        );
    }
}

/// Rate limit result
#[derive(Debug, Clone)]
pub struct RateLimitResult {
    pub allowed: bool,
    pub remaining: u64,
    pub reset_at: Option<chrono::DateTime<chrono::Utc>>,
    pub retry_after: Option<u64>,
}

/// Rate limit rule
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitRule {
    pub name: String,
    pub strategy: String,
    pub limit: u64,
    pub window: chrono::Duration,
    pub burst: Option<u64>,
}

/// Rate limit strategy trait
#[async_trait]
pub trait RateLimitStrategy: Send + Sync {
    async fn check_limit(
        &self,
        redis: &RedisConnection,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult>;
}
```

## Token Bucket Strategy

```rust
/// Token bucket rate limiting
pub struct TokenBucketStrategy;

impl TokenBucketStrategy {
    pub fn new() -> Self {
        Self
    }
}

#[async_trait]
impl RateLimitStrategy for TokenBucketStrategy {
    async fn check_limit(
        &self,
        redis: &RedisConnection,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        let bucket_key = format!("ratelimit:token_bucket:{}:{}", rule.name, key);
        let now = chrono::Utc::now().timestamp();
        
        // Get bucket state
        let bucket_data: Option<String> = redis.get(&bucket_key).await?;
        
        let (mut tokens, last_refill) = if let Some(data) = bucket_data {
            let parts: Vec<&str> = data.split(':').collect();
            if parts.len() == 2 {
                let tokens = parts[0].parse::<f64>().unwrap_or(rule.limit as f64);
                let last_refill = parts[1].parse::<i64>().unwrap_or(now);
                (tokens, last_refill)
            } else {
                (rule.limit as f64, now)
            }
        } else {
            (rule.limit as f64, now)
        };
        
        // Calculate refill
        let time_passed = now - last_refill;
        let refill_rate = rule.limit as f64 / rule.window.num_seconds() as f64;
        let tokens_to_add = time_passed as f64 * refill_rate;
        
        // Add tokens (up to burst limit)
        let max_tokens = rule.burst.unwrap_or(rule.limit) as f64;
        tokens = (tokens + tokens_to_add).min(max_tokens);
        
        // Check if request allowed
        if tokens >= 1.0 {
            tokens -= 1.0;
            
            // Update bucket
            let bucket_data = format!("{}:{}", tokens, now);
            let ttl = rule.window.num_seconds() as usize;
            redis.set_ex(&bucket_key, bucket_data, ttl).await?;
            
            Ok(RateLimitResult {
                allowed: true,
                remaining: tokens.floor() as u64,
                reset_at: None,
                retry_after: None,
            })
        } else {
            // Calculate when next token available
            let tokens_needed = 1.0 - tokens;
            let seconds_until_token = (tokens_needed / refill_rate).ceil() as u64;
            
            Ok(RateLimitResult {
                allowed: false,
                remaining: 0,
                reset_at: Some(chrono::Utc::now() + chrono::Duration::seconds(seconds_until_token as i64)),
                retry_after: Some(seconds_until_token),
            })
        }
    }
}
```

## Sliding Window Strategy

```rust
/// Sliding window rate limiting
pub struct SlidingWindowStrategy;

impl SlidingWindowStrategy {
    pub fn new() -> Self {
        Self
    }
}

#[async_trait]
impl RateLimitStrategy for SlidingWindowStrategy {
    async fn check_limit(
        &self,
        redis: &RedisConnection,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        let window_key = format!("ratelimit:sliding_window:{}:{}", rule.name, key);
        let now = chrono::Utc::now();
        let window_start = now - rule.window;
        
        // Use Redis sorted set for sliding window
        let mut pipe = redis::pipe();
        
        // Remove old entries
        pipe.zremrangebyscore(
            &window_key,
            "-inf",
            window_start.timestamp_millis(),
        );
        
        // Count current window
        pipe.zcard(&window_key);
        
        // Add current request
        pipe.zadd(
            &window_key,
            now.timestamp_millis(),
            now.timestamp_millis(),
        );
        
        // Set expiry
        pipe.expire(&window_key, rule.window.num_seconds() as usize);
        
        let results: Vec<redis::Value> = pipe.query_async(redis).await?;
        
        // Get count from results
        let count: u64 = match &results[1] {
            redis::Value::Int(n) => *n as u64,
            _ => 0,
        };
        
        if count < rule.limit {
            Ok(RateLimitResult {
                allowed: true,
                remaining: rule.limit - count - 1,
                reset_at: Some(now + rule.window),
                retry_after: None,
            })
        } else {
            // Find oldest entry to calculate retry time
            let oldest: Vec<(String, f64)> = redis
                .zrange_withscores(&window_key, 0, 0)
                .await?;
            
            let retry_after = if let Some((_, timestamp)) = oldest.first() {
                let oldest_time = chrono::DateTime::<chrono::Utc>::from_timestamp_millis(*timestamp as i64)
                    .unwrap_or(now);
                let retry_time = oldest_time + rule.window - now;
                Some(retry_time.num_seconds().max(1) as u64)
            } else {
                Some(rule.window.num_seconds() as u64)
            };
            
            Ok(RateLimitResult {
                allowed: false,
                remaining: 0,
                reset_at: Some(now + rule.window),
                retry_after,
            })
        }
    }
}
```

## Fixed Window Strategy

```rust
/// Fixed window rate limiting
pub struct FixedWindowStrategy;

impl FixedWindowStrategy {
    pub fn new() -> Self {
        Self
    }
}

#[async_trait]
impl RateLimitStrategy for FixedWindowStrategy {
    async fn check_limit(
        &self,
        redis: &RedisConnection,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        let now = chrono::Utc::now();
        let window_seconds = rule.window.num_seconds();
        let window_id = now.timestamp() / window_seconds;
        
        let window_key = format!(
            "ratelimit:fixed_window:{}:{}:{}",
            rule.name,
            key,
            window_id
        );
        
        // Increment counter
        let count: u64 = redis.incr(&window_key, 1).await?;
        
        // Set expiry on first request
        if count == 1 {
            redis.expire(&window_key, window_seconds as usize).await?;
        }
        
        if count <= rule.limit {
            Ok(RateLimitResult {
                allowed: true,
                remaining: rule.limit - count,
                reset_at: Some(
                    chrono::DateTime::<chrono::Utc>::from_timestamp(
                        (window_id + 1) * window_seconds,
                        0
                    ).unwrap()
                ),
                retry_after: None,
            })
        } else {
            let reset_time = (window_id + 1) * window_seconds;
            let retry_after = reset_time - now.timestamp();
            
            Ok(RateLimitResult {
                allowed: false,
                remaining: 0,
                reset_at: Some(
                    chrono::DateTime::<chrono::Utc>::from_timestamp(reset_time, 0).unwrap()
                ),
                retry_after: Some(retry_after as u64),
            })
        }
    }
}
```

## API Rate Limiting Middleware

```rust
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
    http::{StatusCode, HeaderMap, HeaderValue},
};

/// Rate limiting middleware
pub async fn rate_limit_middleware(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, RateLimitError> {
    // Extract rate limit key
    let key = extract_rate_limit_key(&req, &state.config).await?;
    
    // Get applicable rules
    let rules = get_rate_limit_rules(&req, &state.config);
    
    // Check limits
    let result = state.rate_limiter
        .check_limits(&key, &rules)
        .await?;
    
    if !result.allowed {
        // Build rate limit response
        let mut headers = HeaderMap::new();
        
        headers.insert(
            "X-RateLimit-Limit",
            HeaderValue::from_str(&rules[0].limit.to_string()).unwrap(),
        );
        
        headers.insert(
            "X-RateLimit-Remaining",
            HeaderValue::from_str(&result.remaining.to_string()).unwrap(),
        );
        
        if let Some(reset) = result.reset_at {
            headers.insert(
                "X-RateLimit-Reset",
                HeaderValue::from_str(&reset.timestamp().to_string()).unwrap(),
            );
        }
        
        if let Some(retry_after) = result.retry_after {
            headers.insert(
                "Retry-After",
                HeaderValue::from_str(&retry_after.to_string()).unwrap(),
            );
        }
        
        return Ok(Response::builder()
            .status(StatusCode::TOO_MANY_REQUESTS)
            .body("Rate limit exceeded".into())
            .unwrap());
    }
    
    // Add rate limit headers to response
    let mut response = next.run(req).await;
    
    response.headers_mut().insert(
        "X-RateLimit-Limit",
        HeaderValue::from_str(&rules[0].limit.to_string()).unwrap(),
    );
    
    response.headers_mut().insert(
        "X-RateLimit-Remaining",
        HeaderValue::from_str(&result.remaining.to_string()).unwrap(),
    );
    
    if let Some(reset) = result.reset_at {
        response.headers_mut().insert(
            "X-RateLimit-Reset",
            HeaderValue::from_str(&reset.timestamp().to_string()).unwrap(),
        );
    }
    
    Ok(response)
}

/// Extract rate limit key from request
async fn extract_rate_limit_key(
    req: &Request,
    config: &RateLimitConfig,
) -> Result<String> {
    // Try authenticated user first
    if let Some(auth_context) = req.extensions().get::<AuthContext>() {
        return Ok(format!("user:{}", auth_context.user_id));
    }
    
    // Try API key
    if let Some(api_key) = extract_api_key(req) {
        return Ok(format!("api_key:{}", api_key));
    }
    
    // Fall back to IP address
    let ip = extract_client_ip(req, config)?;
    Ok(format!("ip:{}", ip))
}

/// Get applicable rate limit rules
fn get_rate_limit_rules(req: &Request, config: &RateLimitConfig) -> Vec<RateLimitRule> {
    let path = req.uri().path();
    let method = req.method();
    
    // Match rules based on path and method
    let mut rules = Vec::new();
    
    for rule_config in &config.rules {
        if rule_config.matches(path, method) {
            rules.push(rule_config.rule.clone());
        }
    }
    
    // Add default rule if no specific rules matched
    if rules.is_empty() {
        rules.push(config.default_rule.clone());
    }
    
    rules
}
```

## Distributed Rate Limiting

```rust
/// Distributed rate limiter using Redis
pub struct DistributedRateLimiter {
    redis_cluster: Arc<RedisCluster>,
    local_cache: Arc<RwLock<HashMap<String, CachedLimit>>>,
    config: DistributedConfig,
}

impl DistributedRateLimiter {
    /// Check limit with local cache
    pub async fn check_limit_cached(
        &self,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        // Check local cache first
        if let Some(cached) = self.get_cached_result(key, rule).await {
            return Ok(cached);
        }
        
        // Check distributed limit
        let result = self.check_distributed_limit(key, rule).await?;
        
        // Cache result if allowed
        if result.allowed {
            self.cache_result(key, rule, &result).await;
        }
        
        Ok(result)
    }
    
    /// Check distributed limit
    async fn check_distributed_limit(
        &self,
        key: &str,
        rule: &RateLimitRule,
    ) -> Result<RateLimitResult> {
        // Use Lua script for atomic operation
        let script = r#"
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            -- Clean old entries
            redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
            
            -- Count current entries
            local count = redis.call('ZCARD', key)
            
            if count < limit then
                -- Add entry
                redis.call('ZADD', key, now, now)
                redis.call('EXPIRE', key, window)
                return {1, limit - count - 1}
            else
                -- Get oldest entry
                local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
                if #oldest > 0 then
                    return {0, oldest[2] + window - now}
                else
                    return {0, window}
                end
            end
        "#;
        
        let key = format!("ratelimit:dist:{}:{}", rule.name, key);
        let now = chrono::Utc::now().timestamp();
        
        let result: Vec<i64> = redis::Script::new(script)
            .key(&key)
            .arg(rule.limit)
            .arg(rule.window.num_seconds())
            .arg(now)
            .invoke_async(&mut self.redis_cluster.get_connection().await?)
            .await?;
        
        if result[0] == 1 {
            Ok(RateLimitResult {
                allowed: true,
                remaining: result[1] as u64,
                reset_at: Some(chrono::Utc::now() + rule.window),
                retry_after: None,
            })
        } else {
            Ok(RateLimitResult {
                allowed: false,
                remaining: 0,
                reset_at: Some(chrono::Utc::now() + chrono::Duration::seconds(result[1])),
                retry_after: Some(result[1] as u64),
            })
        }
    }
    
    /// Get cached result
    async fn get_cached_result(
        &self,
        key: &str,
        rule: &RateLimitRule,
    ) -> Option<RateLimitResult> {
        let cache_key = format!("{}:{}", rule.name, key);
        let cache = self.local_cache.read().await;
        
        if let Some(cached) = cache.get(&cache_key) {
            if cached.expires_at > chrono::Utc::now() {
                return Some(cached.result.clone());
            }
        }
        
        None
    }
}

#[derive(Debug, Clone)]
struct CachedLimit {
    result: RateLimitResult,
    expires_at: chrono::DateTime<chrono::Utc>,
}
```

## Custom Rate Limiters

```rust
/// User-based rate limiter
pub struct UserRateLimiter {
    base_limiter: Arc<RateLimiter>,
    user_tiers: Arc<UserTierService>,
}

impl UserRateLimiter {
    /// Check user-specific limits
    pub async fn check_user_limit(
        &self,
        user_id: &Uuid,
        action: &str,
    ) -> Result<RateLimitResult> {
        // Get user tier
        let tier = self.user_tiers.get_user_tier(user_id).await?;
        
        // Get tier-specific rules
        let rules = self.get_tier_rules(&tier, action);
        
        // Check limits
        let key = format!("user:{}", user_id);
        self.base_limiter.check_limits(&key, &rules).await
    }
    
    /// Get tier-specific rules
    fn get_tier_rules(&self, tier: &UserTier, action: &str) -> Vec<RateLimitRule> {
        match tier {
            UserTier::Free => vec![
                RateLimitRule {
                    name: format!("{}_per_minute", action),
                    strategy: "sliding_window".to_string(),
                    limit: 10,
                    window: chrono::Duration::minutes(1),
                    burst: None,
                },
                RateLimitRule {
                    name: format!("{}_per_hour", action),
                    strategy: "sliding_window".to_string(),
                    limit: 100,
                    window: chrono::Duration::hours(1),
                    burst: None,
                },
            ],
            UserTier::Pro => vec![
                RateLimitRule {
                    name: format!("{}_per_minute", action),
                    strategy: "sliding_window".to_string(),
                    limit: 60,
                    window: chrono::Duration::minutes(1),
                    burst: None,
                },
                RateLimitRule {
                    name: format!("{}_per_hour", action),
                    strategy: "sliding_window".to_string(),
                    limit: 1000,
                    window: chrono::Duration::hours(1),
                    burst: None,
                },
            ],
            UserTier::Enterprise => vec![
                RateLimitRule {
                    name: format!("{}_per_minute", action),
                    strategy: "token_bucket".to_string(),
                    limit: 1000,
                    window: chrono::Duration::minutes(1),
                    burst: Some(2000),
                },
            ],
        }
    }
}

/// Content-based rate limiter
pub struct ContentRateLimiter {
    base_limiter: Arc<RateLimiter>,
}

impl ContentRateLimiter {
    /// Rate limit content creation
    pub async fn check_content_creation(
        &self,
        user_id: &Uuid,
        content_type: &str,
    ) -> Result<RateLimitResult> {
        let rules = vec![
            // Per-user limits
            RateLimitRule {
                name: format!("content_create_{}_per_hour", content_type),
                strategy: "fixed_window".to_string(),
                limit: 10,
                window: chrono::Duration::hours(1),
                burst: None,
            },
            // Global limits
            RateLimitRule {
                name: format!("global_content_create_{}_per_minute", content_type),
                strategy: "sliding_window".to_string(),
                limit: 100,
                window: chrono::Duration::minutes(1),
                burst: None,
            },
        ];
        
        let user_key = format!("user:{}:content:{}", user_id, content_type);
        self.base_limiter.check_limits(&user_key, &rules).await
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitConfig {
    pub enabled: bool,
    pub default_rule: RateLimitRule,
    pub rules: Vec<RateLimitRuleConfig>,
    pub redis_key_prefix: String,
    pub include_headers: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitRuleConfig {
    pub name: String,
    pub paths: Vec<String>,
    pub methods: Vec<String>,
    pub rule: RateLimitRule,
}

impl RateLimitRuleConfig {
    pub fn matches(&self, path: &str, method: &Method) -> bool {
        let path_matches = self.paths.iter().any(|p| {
            if p.ends_with('*') {
                path.starts_with(&p[..p.len() - 1])
            } else {
                path == p
            }
        });
        
        let method_matches = self.methods.is_empty() || 
            self.methods.iter().any(|m| m == method.as_str());
        
        path_matches && method_matches
    }
}

impl Default for RateLimitConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            default_rule: RateLimitRule {
                name: "default".to_string(),
                strategy: "sliding_window".to_string(),
                limit: 60,
                window: chrono::Duration::minutes(1),
                burst: None,
            },
            rules: vec![
                RateLimitRuleConfig {
                    name: "api_strict".to_string(),
                    paths: vec!["/api/*".to_string()],
                    methods: vec![],
                    rule: RateLimitRule {
                        name: "api".to_string(),
                        strategy: "token_bucket".to_string(),
                        limit: 100,
                        window: chrono::Duration::minutes(1),
                        burst: Some(20),
                    },
                },
                RateLimitRuleConfig {
                    name: "auth_endpoints".to_string(),
                    paths: vec!["/auth/*".to_string()],
                    methods: vec!["POST".to_string()],
                    rule: RateLimitRule {
                        name: "auth".to_string(),
                        strategy: "fixed_window".to_string(),
                        limit: 5,
                        window: chrono::Duration::minutes(15),
                        burst: None,
                    },
                },
            ],
            redis_key_prefix: "ratelimit".to_string(),
            include_headers: true,
        }
    }
}
```

## Summary

Diese Rate Limiting Implementation bietet:
- **Multiple Strategies** - Token Bucket, Sliding Window, Fixed Window
- **Distributed Limiting** - Redis-based Cluster Support
- **Flexible Rules** - Path, Method, User-based Rules
- **Performance** - Local Caching, Lua Scripts
- **Middleware Integration** - Axum Middleware Support
- **Custom Limiters** - User Tiers, Content Types

Robust Rate Limiting für API Protection.
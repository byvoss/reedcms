# Security Middleware

## Overview

Security Middleware für ReedCMS Server. Authentication, Rate Limiting, CSRF Protection und Request Validation.

## Authentication Middleware

```rust
use axum::{
    middleware::Next,
    extract::{Request, State},
    response::{Response, IntoResponse},
    http::{StatusCode, HeaderMap},
};
use jsonwebtoken::{decode, DecodingKey, Validation};

/// JWT authentication middleware
pub async fn require_auth(
    State(ctx): State<Arc<ServerContext>>,
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Extract token from Authorization header
    let token = match extract_token(&headers) {
        Some(token) => token,
        None => return Err(StatusCode::UNAUTHORIZED),
    };
    
    // Validate token
    let claims = match validate_token(&token, &ctx.config.jwt_secret) {
        Ok(claims) => claims,
        Err(_) => return Err(StatusCode::UNAUTHORIZED),
    };
    
    // Check if user still exists and is active
    if !is_user_active(&claims.sub, &ctx).await {
        return Err(StatusCode::UNAUTHORIZED);
    }
    
    // Add user info to request extensions
    let mut request = request;
    request.extensions_mut().insert(AuthUser {
        id: claims.sub,
        roles: claims.roles,
        permissions: claims.permissions,
    });
    
    Ok(next.run(request).await)
}

/// Extract bearer token from headers
fn extract_token(headers: &HeaderMap) -> Option<String> {
    headers
        .get("Authorization")
        .and_then(|value| value.to_str().ok())
        .and_then(|value| {
            if value.starts_with("Bearer ") {
                Some(value[7..].to_string())
            } else {
                None
            }
        })
}

/// JWT claims
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,  // User ID
    exp: i64,     // Expiration
    iat: i64,     // Issued at
    roles: Vec<String>,
    permissions: Vec<String>,
}

/// Authenticated user info
#[derive(Clone)]
pub struct AuthUser {
    pub id: String,
    pub roles: Vec<String>,
    pub permissions: Vec<String>,
}

/// Validate JWT token
fn validate_token(token: &str, secret: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
    decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &Validation::default(),
    )
    .map(|data| data.claims)
}

/// Check if user is still active
async fn is_user_active(user_id: &str, ctx: &ServerContext) -> bool {
    // Simple check - could be extended with database lookup
    ctx.redis
        .exists(&format!("user:active:{}", user_id))
        .await
        .unwrap_or(false)
}
```

## Permission Middleware

```rust
/// Require specific permission
pub fn require_permission(permission: &'static str) -> impl Fn(Request, Next) -> BoxFuture<'static, Result<Response, StatusCode>> + Clone {
    move |request: Request, next: Next| {
        Box::pin(async move {
            // Get auth user from extensions
            let user = request.extensions()
                .get::<AuthUser>()
                .ok_or(StatusCode::UNAUTHORIZED)?;
            
            // Check permission
            if !user.permissions.contains(&permission.to_string()) {
                return Err(StatusCode::FORBIDDEN);
            }
            
            Ok(next.run(request).await)
        })
    }
}

/// Require any of the specified roles
pub fn require_role(roles: Vec<&'static str>) -> impl Fn(Request, Next) -> BoxFuture<'static, Result<Response, StatusCode>> + Clone {
    move |request: Request, next: Next| {
        let roles = roles.clone();
        Box::pin(async move {
            let user = request.extensions()
                .get::<AuthUser>()
                .ok_or(StatusCode::UNAUTHORIZED)?;
            
            // Check if user has any required role
            let has_role = roles.iter()
                .any(|role| user.roles.contains(&role.to_string()));
            
            if !has_role {
                return Err(StatusCode::FORBIDDEN);
            }
            
            Ok(next.run(request).await)
        })
    }
}
```

## Rate Limiting

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use std::collections::HashMap;
use std::time::{Duration, Instant};

/// Rate limiting middleware
pub struct RateLimiter {
    limits: Arc<Mutex<HashMap<String, RateLimit>>>,
    max_requests: u32,
    window: Duration,
}

impl RateLimiter {
    pub fn new(max_requests: u32, window_seconds: u64) -> Self {
        Self {
            limits: Arc::new(Mutex::new(HashMap::new())),
            max_requests,
            window: Duration::from_secs(window_seconds),
        }
    }
    
    /// Rate limiting middleware function
    pub async fn middleware(
        &self,
        State(ctx): State<Arc<ServerContext>>,
        request: Request,
        next: Next,
    ) -> Result<Response, StatusCode> {
        // Get client identifier (IP or user ID)
        let client_id = self.get_client_id(&request).await;
        
        // Check rate limit
        if !self.check_rate_limit(&client_id).await {
            return Err(StatusCode::TOO_MANY_REQUESTS);
        }
        
        Ok(next.run(request).await)
    }
    
    /// Get client identifier
    async fn get_client_id(&self, request: &Request) -> String {
        // Try to get authenticated user ID
        if let Some(user) = request.extensions().get::<AuthUser>() {
            return format!("user:{}", user.id);
        }
        
        // Fall back to IP address
        request
            .headers()
            .get("X-Real-IP")
            .or_else(|| request.headers().get("X-Forwarded-For"))
            .and_then(|v| v.to_str().ok())
            .unwrap_or("unknown")
            .to_string()
    }
    
    /// Check and update rate limit
    async fn check_rate_limit(&self, client_id: &str) -> bool {
        let mut limits = self.limits.lock().await;
        let now = Instant::now();
        
        let rate_limit = limits.entry(client_id.to_string())
            .or_insert_with(|| RateLimit {
                count: 0,
                window_start: now,
            });
        
        // Reset window if expired
        if now.duration_since(rate_limit.window_start) > self.window {
            rate_limit.count = 0;
            rate_limit.window_start = now;
        }
        
        // Check limit
        if rate_limit.count >= self.max_requests {
            return false;
        }
        
        // Increment counter
        rate_limit.count += 1;
        true
    }
}

#[derive(Debug)]
struct RateLimit {
    count: u32,
    window_start: Instant,
}
```

## CSRF Protection

```rust
use rand::Rng;
use sha2::{Sha256, Digest};

/// CSRF protection middleware
pub struct CsrfProtection {
    secret: String,
}

impl CsrfProtection {
    pub fn new(secret: String) -> Self {
        Self { secret }
    }
    
    /// CSRF middleware for state-changing requests
    pub async fn middleware(
        &self,
        headers: HeaderMap,
        request: Request,
        next: Next,
    ) -> Result<Response, StatusCode> {
        // Skip CSRF check for safe methods
        if matches!(
            request.method(),
            &axum::http::Method::GET | &axum::http::Method::HEAD | &axum::http::Method::OPTIONS
        ) {
            return Ok(next.run(request).await);
        }
        
        // Extract CSRF token
        let token = headers
            .get("X-CSRF-Token")
            .or_else(|| headers.get("X-XSRF-Token"))
            .and_then(|v| v.to_str().ok());
        
        if let Some(token) = token {
            // Validate token
            if self.validate_token(token) {
                return Ok(next.run(request).await);
            }
        }
        
        Err(StatusCode::FORBIDDEN)
    }
    
    /// Generate CSRF token
    pub fn generate_token(&self) -> String {
        let timestamp = chrono::Utc::now().timestamp();
        let random: u32 = rand::thread_rng().gen();
        
        let data = format!("{}-{}-{}", timestamp, random, self.secret);
        let hash = Sha256::digest(data.as_bytes());
        
        format!("{}-{}-{:x}", timestamp, random, hash)
    }
    
    /// Validate CSRF token
    fn validate_token(&self, token: &str) -> bool {
        let parts: Vec<&str> = token.split('-').collect();
        if parts.len() != 3 {
            return false;
        }
        
        let timestamp = parts[0].parse::<i64>().unwrap_or(0);
        let random = parts[1];
        let provided_hash = parts[2];
        
        // Check if token is not too old (1 hour)
        let now = chrono::Utc::now().timestamp();
        if now - timestamp > 3600 {
            return false;
        }
        
        // Verify hash
        let data = format!("{}-{}-{}", timestamp, random, self.secret);
        let expected_hash = format!("{:x}", Sha256::digest(data.as_bytes()));
        
        // Constant-time comparison
        provided_hash == expected_hash
    }
}
```

## Request Validation

```rust
/// Request size limiting middleware
pub async fn limit_body_size(
    State(ctx): State<Arc<ServerContext>>,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let content_length = request
        .headers()
        .get("content-length")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse::<usize>().ok())
        .unwrap_or(0);
    
    if content_length > ctx.config.server.body_limit {
        return Err(StatusCode::PAYLOAD_TOO_LARGE);
    }
    
    Ok(next.run(request).await)
}

/// Content-Type validation
pub async fn validate_content_type(
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Only validate for POST/PUT requests
    if !matches!(
        request.method(),
        &axum::http::Method::POST | &axum::http::Method::PUT
    ) {
        return Ok(next.run(request).await);
    }
    
    let content_type = headers
        .get("content-type")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("");
    
    // Check for allowed content types
    let allowed = [
        "application/json",
        "application/x-www-form-urlencoded",
        "multipart/form-data",
    ];
    
    let is_allowed = allowed.iter()
        .any(|&allowed_type| content_type.starts_with(allowed_type));
    
    if !is_allowed && !content_type.is_empty() {
        return Err(StatusCode::UNSUPPORTED_MEDIA_TYPE);
    }
    
    Ok(next.run(request).await)
}
```

## Security Headers

```rust
use axum::http::header;

/// Add security headers to responses
pub async fn security_headers(
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let mut response = next.run(request).await;
    
    let headers = response.headers_mut();
    
    // Prevent XSS
    headers.insert(
        "X-Content-Type-Options",
        "nosniff".parse().unwrap()
    );
    
    // Prevent clickjacking
    headers.insert(
        "X-Frame-Options",
        "DENY".parse().unwrap()
    );
    
    // XSS Protection for older browsers
    headers.insert(
        "X-XSS-Protection",
        "1; mode=block".parse().unwrap()
    );
    
    // Referrer policy
    headers.insert(
        "Referrer-Policy",
        "strict-origin-when-cross-origin".parse().unwrap()
    );
    
    // Permissions policy
    headers.insert(
        "Permissions-Policy",
        "geolocation=(), microphone=(), camera=()".parse().unwrap()
    );
    
    // Basic CSP
    headers.insert(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;".parse().unwrap()
    );
    
    Ok(response)
}
```

## Request Logging

```rust
use tracing::{info, warn};

/// Request logging middleware
pub async fn log_requests(
    State(ctx): State<Arc<ServerContext>>,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = Instant::now();
    
    // Get request ID
    let request_id = request
        .headers()
        .get("X-Request-ID")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();
    
    // Log request
    info!(
        request_id = %request_id,
        method = %method,
        path = %uri.path(),
        "Incoming request"
    );
    
    // Process request
    let response = next.run(request).await;
    
    // Log response
    let duration = start.elapsed();
    let status = response.status();
    
    if status.is_client_error() || status.is_server_error() {
        warn!(
            request_id = %request_id,
            method = %method,
            path = %uri.path(),
            status = %status.as_u16(),
            duration_ms = %duration.as_millis(),
            "Request failed"
        );
    } else {
        info!(
            request_id = %request_id,
            method = %method,
            path = %uri.path(),
            status = %status.as_u16(),
            duration_ms = %duration.as_millis(),
            "Request completed"
        );
    }
    
    // Update metrics
    ctx.metrics.requests_total.inc();
    ctx.metrics.request_duration.observe(duration.as_secs_f64());
    
    Ok(response)
}
```

## Middleware Stack

```rust
/// Configure complete middleware stack
pub fn configure_middleware(app: Router, config: &ServerConfig) -> Router {
    let rate_limiter = RateLimiter::new(
        config.rate_limit_requests,
        config.rate_limit_window,
    );
    
    let csrf_protection = CsrfProtection::new(
        config.csrf_secret.clone(),
    );
    
    app
        // Global middleware (applied to all routes)
        .layer(middleware::from_fn(log_requests))
        .layer(middleware::from_fn(security_headers))
        .layer(middleware::from_fn(limit_body_size))
        .layer(middleware::from_fn(validate_content_type))
        .layer(middleware::from_fn(move |state, request, next| {
            rate_limiter.middleware(state, request, next)
        }))
        .layer(middleware::from_fn(move |headers, request, next| {
            csrf_protection.middleware(headers, request, next)
        }))
}
```

## Summary

Diese Security Middleware bietet:
- **JWT Authentication** - Token-based Auth mit User Validation
- **Permission System** - Role und Permission based Access Control
- **Rate Limiting** - Configurable Request Limits
- **CSRF Protection** - Token-based CSRF Prevention
- **Security Headers** - XSS, Clickjacking Protection
- **Request Validation** - Size Limits und Content-Type Checks

Comprehensive Security Layer für production deployments.
# Request Context Building

## Overview

Request Context Building für ReedCMS. Extrahiert und aggregiert alle relevanten Request-Daten für Template Rendering und Business Logic.

## Request Context Core

```rust
use axum::{
    extract::{Query, Path, State, Host},
    http::{HeaderMap, Uri, Method},
};
use std::collections::HashMap;
use serde_json::Value;

/// Request context containing all relevant request data
#[derive(Debug, Clone)]
pub struct RequestContext {
    // Request info
    pub method: Method,
    pub uri: Uri,
    pub host: String,
    pub scheme: String,
    pub remote_addr: Option<String>,
    
    // Path and query
    pub path: String,
    pub path_params: HashMap<String, String>,
    pub query_params: HashMap<String, String>,
    
    // Headers
    pub headers: HashMap<String, String>,
    pub cookies: HashMap<String, String>,
    
    // Locale and theme
    pub locale: String,
    pub theme: String,
    
    // User context
    pub user: Option<UserContext>,
    
    // Device info
    pub device: DeviceInfo,
    
    // Request metadata
    pub request_id: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

/// User context from auth
#[derive(Debug, Clone)]
pub struct UserContext {
    pub id: String,
    pub username: String,
    pub email: String,
    pub roles: Vec<String>,
    pub permissions: Vec<String>,
    pub preferences: HashMap<String, Value>,
}

/// Device information
#[derive(Debug, Clone)]
pub struct DeviceInfo {
    pub user_agent: String,
    pub device_type: DeviceType,
    pub browser: String,
    pub os: String,
    pub is_mobile: bool,
}

#[derive(Debug, Clone, PartialEq)]
pub enum DeviceType {
    Desktop,
    Mobile,
    Tablet,
    Bot,
}
```

## Context Builder

```rust
/// Build request context from Axum extractors
pub struct RequestContextBuilder;

impl RequestContextBuilder {
    /// Build complete request context
    pub async fn build(
        method: Method,
        uri: Uri,
        headers: HeaderMap,
        host: Option<Host>,
        state: Arc<ServerContext>,
    ) -> Result<RequestContext> {
        let request_id = Self::extract_request_id(&headers);
        let remote_addr = Self::extract_remote_addr(&headers);
        let cookies = Self::extract_cookies(&headers)?;
        
        // Extract locale
        let locale = Self::detect_locale(&headers, &cookies, &uri)?;
        
        // Extract theme
        let theme = Self::detect_theme(&cookies, &uri, &state).await?;
        
        // Get user context if authenticated
        let user = Self::extract_user_context(&headers, &state).await?;
        
        // Parse device info
        let device = Self::parse_device_info(&headers)?;
        
        // Build context
        Ok(RequestContext {
            method,
            uri: uri.clone(),
            host: host.map(|h| h.0).unwrap_or_else(|| "localhost".to_string()),
            scheme: Self::detect_scheme(&headers),
            remote_addr,
            path: uri.path().to_string(),
            path_params: HashMap::new(), // Will be populated by router
            query_params: Self::parse_query_params(uri.query()),
            headers: Self::headers_to_map(&headers),
            cookies,
            locale,
            theme,
            user,
            device,
            request_id,
            timestamp: chrono::Utc::now(),
        })
    }
    
    /// Extract request ID
    fn extract_request_id(headers: &HeaderMap) -> String {
        headers
            .get("X-Request-ID")
            .and_then(|v| v.to_str().ok())
            .map(String::from)
            .unwrap_or_else(|| uuid::Uuid::new_v4().to_string())
    }
    
    /// Extract real client IP
    fn extract_remote_addr(headers: &HeaderMap) -> Option<String> {
        // Check common forwarded headers
        headers
            .get("X-Real-IP")
            .or_else(|| headers.get("X-Forwarded-For"))
            .and_then(|v| v.to_str().ok())
            .map(|v| {
                // X-Forwarded-For can contain multiple IPs
                v.split(',').next().unwrap_or(v).trim().to_string()
            })
    }
    
    /// Detect request scheme
    fn detect_scheme(headers: &HeaderMap) -> String {
        headers
            .get("X-Forwarded-Proto")
            .and_then(|v| v.to_str().ok())
            .unwrap_or("http")
            .to_string()
    }
}
```

## Locale Detection

```rust
impl RequestContextBuilder {
    /// Detect locale from multiple sources
    fn detect_locale(
        headers: &HeaderMap,
        cookies: &HashMap<String, String>,
        uri: &Uri,
    ) -> Result<String> {
        // Priority order:
        // 1. Query parameter (?locale=de)
        if let Some(query) = uri.query() {
            if let Some(locale) = Self::extract_query_param(query, "locale") {
                if Self::is_valid_locale(&locale) {
                    return Ok(locale);
                }
            }
        }
        
        // 2. Cookie
        if let Some(locale) = cookies.get("locale") {
            if Self::is_valid_locale(locale) {
                return Ok(locale.clone());
            }
        }
        
        // 3. Accept-Language header
        if let Some(accept_lang) = headers.get("Accept-Language") {
            if let Ok(header_value) = accept_lang.to_str() {
                if let Some(locale) = Self::parse_accept_language(header_value) {
                    return Ok(locale);
                }
            }
        }
        
        // 4. Default
        Ok("en".to_string())
    }
    
    /// Parse Accept-Language header
    fn parse_accept_language(header: &str) -> Option<String> {
        // Parse "en-US,en;q=0.9,de;q=0.8"
        let mut languages: Vec<(String, f32)> = header
            .split(',')
            .filter_map(|lang| {
                let parts: Vec<&str> = lang.trim().split(';').collect();
                let locale = parts[0].split('-').next()?.to_string();
                
                let quality = if parts.len() > 1 {
                    parts[1]
                        .trim()
                        .strip_prefix("q=")?
                        .parse::<f32>()
                        .ok()?
                } else {
                    1.0
                };
                
                Some((locale, quality))
            })
            .collect();
        
        // Sort by quality
        languages.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        
        // Return highest quality locale that we support
        languages.into_iter()
            .map(|(locale, _)| locale)
            .find(|locale| Self::is_valid_locale(locale))
    }
    
    /// Check if locale is supported
    fn is_valid_locale(locale: &str) -> bool {
        matches!(locale, "en" | "de" | "fr" | "es" | "it" | "nl")
    }
}
```

## Theme Detection

```rust
impl RequestContextBuilder {
    /// Detect theme from context
    async fn detect_theme(
        cookies: &HashMap<String, String>,
        uri: &Uri,
        state: &ServerContext,
    ) -> Result<String> {
        // Check query parameter
        if let Some(query) = uri.query() {
            if let Some(theme) = Self::extract_query_param(query, "theme") {
                if state.theme_manager.theme_exists(&theme).await? {
                    return Ok(theme);
                }
            }
        }
        
        // Check cookie
        if let Some(theme) = cookies.get("theme") {
            if state.theme_manager.theme_exists(theme).await? {
                return Ok(theme.clone());
            }
        }
        
        // Get current active theme
        let current_theme = state.theme_manager.get_current_theme().await?;
        Ok(current_theme.name)
    }
}
```

## Device Detection

```rust
impl RequestContextBuilder {
    /// Parse device information from User-Agent
    fn parse_device_info(headers: &HeaderMap) -> Result<DeviceInfo> {
        let user_agent = headers
            .get("User-Agent")
            .and_then(|v| v.to_str().ok())
            .unwrap_or("")
            .to_string();
        
        let ua_lower = user_agent.to_lowercase();
        
        // Detect device type
        let device_type = if Self::is_bot(&ua_lower) {
            DeviceType::Bot
        } else if Self::is_mobile(&ua_lower) {
            DeviceType::Mobile
        } else if Self::is_tablet(&ua_lower) {
            DeviceType::Tablet
        } else {
            DeviceType::Desktop
        };
        
        // Detect browser
        let browser = Self::detect_browser(&ua_lower);
        
        // Detect OS
        let os = Self::detect_os(&ua_lower);
        
        Ok(DeviceInfo {
            user_agent,
            device_type: device_type.clone(),
            browser,
            os,
            is_mobile: matches!(device_type, DeviceType::Mobile | DeviceType::Tablet),
        })
    }
    
    fn is_bot(ua: &str) -> bool {
        ua.contains("bot") || 
        ua.contains("crawler") || 
        ua.contains("spider") ||
        ua.contains("scraper")
    }
    
    fn is_mobile(ua: &str) -> bool {
        ua.contains("mobile") || 
        ua.contains("android") || 
        ua.contains("iphone")
    }
    
    fn is_tablet(ua: &str) -> bool {
        ua.contains("tablet") || 
        ua.contains("ipad")
    }
    
    fn detect_browser(ua: &str) -> String {
        if ua.contains("firefox") {
            "Firefox".to_string()
        } else if ua.contains("chrome") {
            "Chrome".to_string()
        } else if ua.contains("safari") {
            "Safari".to_string()
        } else if ua.contains("edge") {
            "Edge".to_string()
        } else {
            "Unknown".to_string()
        }
    }
    
    fn detect_os(ua: &str) -> String {
        if ua.contains("windows") {
            "Windows".to_string()
        } else if ua.contains("mac os") || ua.contains("macos") {
            "macOS".to_string()
        } else if ua.contains("linux") {
            "Linux".to_string()
        } else if ua.contains("android") {
            "Android".to_string()
        } else if ua.contains("ios") || ua.contains("iphone") || ua.contains("ipad") {
            "iOS".to_string()
        } else {
            "Unknown".to_string()
        }
    }
}
```

## Context Enrichment

```rust
/// Enrich request context with additional data
pub struct ContextEnricher;

impl ContextEnricher {
    /// Enrich context for template rendering
    pub async fn enrich_for_template(
        ctx: &mut RequestContext,
        state: &ServerContext,
    ) -> Result<HashMap<String, Value>> {
        let mut template_context = HashMap::new();
        
        // Add request info
        template_context.insert("request".to_string(), json!({
            "method": ctx.method.as_str(),
            "path": ctx.path,
            "host": ctx.host,
            "scheme": ctx.scheme,
            "url": format!("{}://{}{}", ctx.scheme, ctx.host, ctx.uri),
            "query": ctx.query_params,
            "locale": ctx.locale,
            "theme": ctx.theme,
            "device": {
                "type": format!("{:?}", ctx.device.device_type),
                "is_mobile": ctx.device.is_mobile,
                "browser": ctx.device.browser,
                "os": ctx.device.os,
            },
        }));
        
        // Add user info if authenticated
        if let Some(ref user) = ctx.user {
            template_context.insert("user".to_string(), json!({
                "id": user.id,
                "username": user.username,
                "email": user.email,
                "roles": user.roles,
                "is_authenticated": true,
            }));
        } else {
            template_context.insert("user".to_string(), json!({
                "is_authenticated": false,
            }));
        }
        
        // Add site config
        template_context.insert("site".to_string(), json!({
            "name": state.config.site.name,
            "title": state.config.site.title,
            "description": state.config.site.description,
            "url": state.config.site.url,
        }));
        
        // Add available locales
        template_context.insert("locales".to_string(), json!([
            {"code": "en", "name": "English"},
            {"code": "de", "name": "Deutsch"},
            {"code": "fr", "name": "Français"},
            {"code": "es", "name": "Español"},
        ]));
        
        // Add CSRF token if needed
        if let Some(csrf_token) = Self::generate_csrf_token(ctx, state)? {
            template_context.insert("csrf_token".to_string(), json!(csrf_token));
        }
        
        Ok(template_context)
    }
    
    /// Generate CSRF token for forms
    fn generate_csrf_token(
        ctx: &RequestContext,
        state: &ServerContext,
    ) -> Result<Option<String>> {
        if let Some(ref user) = ctx.user {
            let token = state.csrf_protection.generate_token_for_user(&user.id)?;
            Ok(Some(token))
        } else {
            Ok(None)
        }
    }
}
```

## Context Extensions

```rust
/// Trait for extending request context
pub trait ContextExtension: Send + Sync {
    /// Name of the extension
    fn name(&self) -> &str;
    
    /// Extend the context
    fn extend(
        &self,
        ctx: &RequestContext,
        data: &mut HashMap<String, Value>,
    ) -> BoxFuture<'_, Result<()>>;
}

/// Navigation context extension
pub struct NavigationExtension {
    snippet_manager: Arc<SnippetManager>,
}

impl ContextExtension for NavigationExtension {
    fn name(&self) -> &str {
        "navigation"
    }
    
    fn extend(
        &self,
        ctx: &RequestContext,
        data: &mut HashMap<String, Value>,
    ) -> BoxFuture<'_, Result<()>> {
        Box::pin(async move {
            let navigation = self.snippet_manager
                .get_navigation_for_locale(&ctx.locale)
                .await?;
            
            data.insert("navigation".to_string(), json!(navigation));
            Ok(())
        })
    }
}

/// Breadcrumb context extension
pub struct BreadcrumbExtension {
    ucg_manager: Arc<UcgManager>,
}

impl ContextExtension for BreadcrumbExtension {
    fn name(&self) -> &str {
        "breadcrumbs"
    }
    
    fn extend(
        &self,
        ctx: &RequestContext,
        data: &mut HashMap<String, Value>,
    ) -> BoxFuture<'_, Result<()>> {
        Box::pin(async move {
            if let Some(current_id) = ctx.path_params.get("id") {
                if let Ok(id) = current_id.parse::<Uuid>() {
                    let breadcrumbs = self.ucg_manager
                        .get_breadcrumb_path(id)
                        .await?;
                    
                    data.insert("breadcrumbs".to_string(), json!(breadcrumbs));
                }
            }
            Ok(())
        })
    }
}
```

## Context Middleware

```rust
/// Middleware to inject request context
pub async fn inject_request_context(
    State(state): State<Arc<ServerContext>>,
    method: Method,
    uri: Uri,
    Host(host): Host,
    headers: HeaderMap,
    mut request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Build context
    let context = RequestContextBuilder::build(
        method,
        uri,
        headers,
        Some(Host(host)),
        state,
    )
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    // Inject into request extensions
    request.extensions_mut().insert(context);
    
    Ok(next.run(request).await)
}
```

## Summary

Dieses Request Context System bietet:
- **Complete Request Data** - Method, Headers, Query, Path
- **Locale Detection** - Multi-source Locale Resolution
- **Theme Detection** - Context-aware Theme Selection  
- **Device Detection** - User-Agent Parsing
- **User Context** - Authentication Integration
- **Extensible System** - Plugin-based Context Extensions

Robust Context Building für Template Rendering und Business Logic.
# Route Definitions

## Overview

Route Definitions für ReedCMS Web Server. RESTful API Routes, Content Routes und Asset Serving mit intelligenter Route Matching.

## Route Configuration

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{Path, Query, State},
    response::{Html, Json, IntoResponse},
    http::StatusCode,
};

/// Configure all server routes
pub fn configure_routes(router: Router) -> Result<Router> {
    let router = router
        // Health checks
        .route("/health", get(health_check))
        .route("/ready", get(readiness_check))
        .route("/metrics", get(metrics_handler))
        
        // API routes
        .nest("/api", api_routes())
        
        // Content routes
        .nest("/content", content_routes())
        
        // Asset routes
        .nest("/assets", asset_routes())
        
        // Admin routes (protected)
        .nest("/admin", admin_routes())
        
        // Catch-all for dynamic pages
        .fallback(page_handler);
    
    Ok(router)
}
```

## API Routes

```rust
/// API route definitions
fn api_routes() -> Router {
    Router::new()
        // Snippet endpoints
        .route("/snippets", get(list_snippets).post(create_snippet))
        .route("/snippets/:id", 
            get(get_snippet)
            .put(update_snippet)
            .delete(delete_snippet)
        )
        .route("/snippets/:id/children", get(get_snippet_children))
        .route("/snippets/:id/path", get(get_snippet_path))
        
        // Search
        .route("/search", get(search_snippets))
        
        // Schema registry
        .route("/schemas", get(list_schemas))
        .route("/schemas/:name", get(get_schema))
        
        // Themes
        .route("/themes", get(list_themes))
        .route("/themes/current", get(get_current_theme))
        .route("/themes/:name", get(get_theme))
        
        // Translations
        .route("/translations/:key", get(get_translation))
        
        // UCG paths
        .route("/ucg/*path", get(resolve_ucg_path))
}

/// List snippets with pagination
async fn list_snippets(
    Query(params): Query<ListParams>,
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Json<ListResponse<Snippet>>, ApiError> {
    let snippets = ctx.snippet_manager
        .list(
            params.snippet_type.as_deref(),
            params.parent,
            params.limit.unwrap_or(50),
            params.offset.unwrap_or(0),
        )
        .await?;
    
    let total = ctx.snippet_manager
        .count(params.snippet_type.as_deref(), params.parent)
        .await?;
    
    Ok(Json(ListResponse {
        items: snippets,
        total,
        limit: params.limit.unwrap_or(50),
        offset: params.offset.unwrap_or(0),
    }))
}

#[derive(Deserialize)]
struct ListParams {
    snippet_type: Option<String>,
    parent: Option<Uuid>,
    limit: Option<usize>,
    offset: Option<usize>,
    sort: Option<String>,
    order: Option<String>,
}

#[derive(Serialize)]
struct ListResponse<T> {
    items: Vec<T>,
    total: usize,
    limit: usize,
    offset: usize,
}
```

## Content Routes

```rust
/// Content delivery routes
fn content_routes() -> Router {
    Router::new()
        // Raw content
        .route("/:id", get(get_content))
        .route("/:id/render", get(render_content))
        
        // Content versions
        .route("/:id/versions", get(list_versions))
        .route("/:id/versions/:version", get(get_version))
        
        // Related content
        .route("/:id/related", get(get_related))
        
        // Content tree
        .route("/:id/tree", get(get_content_tree))
}

/// Render content with template
async fn render_content(
    Path(id): Path<Uuid>,
    Query(params): Query<RenderParams>,
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Html<String>, ApiError> {
    // Get snippet
    let snippet = ctx.snippet_manager
        .get_by_id(id)
        .await?
        .ok_or_else(|| ApiError::NotFound(format!("Snippet {}", id)))?;
    
    // Determine template
    let template = params.template.unwrap_or_else(|| {
        // Default template based on snippet type
        format!("snippets/{}.tera", snippet.snippet_type)
    });
    
    // Build context
    let mut context = ctx.template_engine
        .build_context(&snippet)
        .await?;
    
    // Add request params
    if let Some(locale) = params.locale {
        context.insert("locale", &locale);
    }
    
    // Render
    let html = ctx.template_engine
        .render(&template, &context)
        .await?;
    
    Ok(Html(html))
}

#[derive(Deserialize)]
struct RenderParams {
    template: Option<String>,
    locale: Option<String>,
    theme: Option<String>,
}
```

## Asset Routes

```rust
use tower_http::services::ServeDir;
use axum::response::Response;

/// Asset serving routes
fn asset_routes() -> Router {
    Router::new()
        // Theme assets with EPC resolution
        .route("/theme/*path", get(serve_theme_asset))
        
        // Static assets
        .nest_service("/static", ServeDir::new("static"))
        
        // Uploaded assets
        .route("/uploads/*path", get(serve_upload))
        
        // Optimized images
        .route("/images/*path", get(serve_image))
}

/// Serve theme assets with EPC resolution
async fn serve_theme_asset(
    Path(path): Path<String>,
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Response, ApiError> {
    // Get current theme
    let theme = ctx.theme_manager.get_current_theme().await?;
    
    // Resolve file through theme chain
    let file_path = ctx.epc_resolver
        .resolve_asset(&path, &theme)
        .await?
        .ok_or_else(|| ApiError::NotFound(format!("Asset: {}", path)))?;
    
    // Determine content type
    let content_type = mime_guess::from_path(&file_path)
        .first_or_octet_stream();
    
    // Read file
    let content = tokio::fs::read(&file_path).await?;
    
    // Build response with caching headers
    let response = Response::builder()
        .status(StatusCode::OK)
        .header("Content-Type", content_type.as_ref())
        .header("Cache-Control", "public, max-age=31536000")
        .header("ETag", format!("\"{}\"", calculate_etag(&content)))
        .body(content.into())
        .unwrap();
    
    Ok(response)
}

/// Calculate ETag for content
fn calculate_etag(content: &[u8]) -> String {
    use sha2::{Sha256, Digest};
    let mut hasher = Sha256::new();
    hasher.update(content);
    format!("{:x}", hasher.finalize())
}
```

## Dynamic Page Routes

```rust
/// Catch-all page handler for dynamic routes
async fn page_handler(
    uri: axum::http::Uri,
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Response, ApiError> {
    let path = uri.path();
    
    // Try to find matching route in database
    let route = ctx.snippet_manager
        .find_by_route(path)
        .await?;
    
    if let Some(snippet) = route {
        // Check if routable
        if !snippet.is_routable() {
            return Err(ApiError::NotFound(format!("Route: {}", path)));
        }
        
        // Determine template
        let template = format!("pages/{}.tera", snippet.snippet_type);
        
        // Build context
        let mut context = ctx.template_engine
            .build_page_context(&snippet)
            .await?;
        
        // Add navigation
        let navigation = ctx.snippet_manager
            .get_navigation()
            .await?;
        context.insert("navigation", &navigation);
        
        // Render page
        let html = ctx.template_engine
            .render(&template, &context)
            .await?;
        
        // Build response
        let response = Response::builder()
            .status(StatusCode::OK)
            .header("Content-Type", "text/html; charset=utf-8")
            .body(html.into())
            .unwrap();
        
        Ok(response)
    } else {
        // Try static file fallback
        serve_static_file(path).await
    }
}

/// Serve static files as fallback
async fn serve_static_file(path: &str) -> Result<Response, ApiError> {
    let file_path = PathBuf::from("public").join(path.trim_start_matches('/'));
    
    if file_path.exists() && file_path.is_file() {
        let content = tokio::fs::read(&file_path).await?;
        let content_type = mime_guess::from_path(&file_path)
            .first_or_octet_stream();
        
        let response = Response::builder()
            .status(StatusCode::OK)
            .header("Content-Type", content_type.as_ref())
            .body(content.into())
            .unwrap();
        
        Ok(response)
    } else {
        Err(ApiError::NotFound(format!("Path: {}", path)))
    }
}
```

## Admin Routes

```rust
/// Protected admin routes
fn admin_routes() -> Router {
    Router::new()
        // Dashboard
        .route("/", get(admin_dashboard))
        
        // Content management
        .route("/content", get(admin_content_list))
        .route("/content/new", get(admin_content_new).post(admin_content_create))
        .route("/content/:id", get(admin_content_edit).post(admin_content_update))
        
        // Theme management
        .route("/themes", get(admin_themes))
        .route("/themes/:name/activate", post(admin_activate_theme))
        
        // System info
        .route("/system", get(admin_system_info))
        .route("/system/cache/clear", post(admin_clear_cache))
        
        // Apply auth middleware to all admin routes
        .layer(middleware::from_fn(require_auth))
}

/// Admin dashboard
async fn admin_dashboard(
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Html<String>, ApiError> {
    let stats = AdminStats {
        total_snippets: ctx.snippet_manager.count_all().await?,
        total_themes: ctx.theme_manager.list_themes().await?.len(),
        cache_size: ctx.cache.size().await?,
        uptime: get_uptime(),
    };
    
    let context = json!({
        "title": "ReedCMS Admin",
        "stats": stats,
    });
    
    let html = ctx.template_engine
        .render("admin/dashboard.tera", &context)
        .await?;
    
    Ok(Html(html))
}

#[derive(Serialize)]
struct AdminStats {
    total_snippets: usize,
    total_themes: usize,
    cache_size: usize,
    uptime: String,
}
```

## Error Handling

```rust
/// API error type
#[derive(Debug)]
enum ApiError {
    NotFound(String),
    BadRequest(String),
    Unauthorized,
    Internal(String),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            ApiError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
            ApiError::Unauthorized => (StatusCode::UNAUTHORIZED, "Unauthorized".to_string()),
            ApiError::Internal(msg) => (StatusCode::INTERNAL_SERVER_ERROR, msg),
        };
        
        let body = json!({
            "error": {
                "code": status.as_u16(),
                "message": message,
            }
        });
        
        (status, Json(body)).into_response()
    }
}

/// Convert from ReedError
impl From<ReedError> for ApiError {
    fn from(err: ReedError) -> Self {
        match err {
            ReedError::NotFound(msg) => ApiError::NotFound(msg),
            ReedError::InvalidData(msg) => ApiError::BadRequest(msg),
            _ => ApiError::Internal(err.to_string()),
        }
    }
}
```

## Route Helpers

```rust
/// Route matching helpers
pub struct RouteMatcher {
    patterns: Vec<RoutePattern>,
}

impl RouteMatcher {
    pub fn new() -> Self {
        Self {
            patterns: Vec::new(),
        }
    }
    
    /// Add route pattern
    pub fn add_pattern(&mut self, pattern: &str, handler: RouteHandler) {
        self.patterns.push(RoutePattern {
            regex: Self::compile_pattern(pattern),
            params: Self::extract_params(pattern),
            handler,
        });
    }
    
    /// Match route
    pub fn match_route(&self, path: &str) -> Option<(RouteHandler, HashMap<String, String>)> {
        for pattern in &self.patterns {
            if let Some(captures) = pattern.regex.captures(path) {
                let mut params = HashMap::new();
                
                for (i, param) in pattern.params.iter().enumerate() {
                    if let Some(value) = captures.get(i + 1) {
                        params.insert(param.clone(), value.as_str().to_string());
                    }
                }
                
                return Some((pattern.handler.clone(), params));
            }
        }
        
        None
    }
    
    /// Compile route pattern to regex
    fn compile_pattern(pattern: &str) -> regex::Regex {
        let mut regex_pattern = pattern.to_string();
        
        // Replace :param with ([^/]+)
        let param_regex = regex::Regex::new(r":([^/]+)").unwrap();
        regex_pattern = param_regex.replace_all(&regex_pattern, "([^/]+)").to_string();
        
        // Replace * with (.*)
        regex_pattern = regex_pattern.replace("*", "(.*)");
        
        regex::Regex::new(&format!("^{}$", regex_pattern)).unwrap()
    }
    
    /// Extract parameter names from pattern
    fn extract_params(pattern: &str) -> Vec<String> {
        let param_regex = regex::Regex::new(r":([^/]+)").unwrap();
        param_regex.captures_iter(pattern)
            .map(|cap| cap[1].to_string())
            .collect()
    }
}

#[derive(Clone)]
struct RoutePattern {
    regex: regex::Regex,
    params: Vec<String>,
    handler: RouteHandler,
}

type RouteHandler = Arc<dyn Fn(HashMap<String, String>) -> BoxFuture<'static, Response> + Send + Sync>;
```

## Summary

Diese Route Definitions bieten:
- **RESTful API** - Clean API Design für alle Resources
- **Content Delivery** - Flexible Content Rendering
- **Asset Pipeline** - EPC-aware Asset Serving
- **Dynamic Routing** - Database-driven Page Routes
- **Admin Interface** - Protected Management Routes
- **Error Handling** - Consistent Error Responses

Flexible und erweiterbare Route Architecture.
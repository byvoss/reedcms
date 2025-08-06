# Asset Serving

## Overview

Asset Serving System für ReedCMS. Efficient Static File Delivery mit EPC Resolution, Conditional Requests und CDN Support.

## Asset Server Core

```rust
use axum::{
    extract::{Path, Query, State},
    response::{Response, IntoResponse},
    http::{StatusCode, header, HeaderMap, HeaderValue},
    body::Body,
};
use tokio::io::AsyncReadExt;
use std::path::PathBuf;

/// Asset server for static files
pub struct AssetServer {
    config: AssetServerConfig,
    resolver: Arc<AssetResolver>,
    cache: Arc<AssetCache>,
    processor: Arc<AssetProcessor>,
}

impl AssetServer {
    pub fn new(
        config: AssetServerConfig,
        epc_resolver: Arc<EpcResolver>,
        processor: Arc<AssetProcessor>,
    ) -> Self {
        let resolver = Arc::new(AssetResolver::new(
            config.clone(),
            epc_resolver,
        ));
        
        let cache = Arc::new(AssetCache::new(
            config.cache_size,
            config.cache_ttl,
        ));
        
        Self {
            config,
            resolver,
            cache,
            processor,
        }
    }
    
    /// Serve asset with EPC resolution
    pub async fn serve_asset(
        &self,
        path: &str,
        headers: &HeaderMap,
        context: &RequestContext,
    ) -> Result<Response> {
        // Resolve asset path through EPC
        let resolved_path = self.resolver
            .resolve_asset_path(path, &context.theme)
            .await?
            .ok_or_else(|| ReedError::NotFound(format!("Asset: {}", path)))?;
        
        // Check cache
        let cache_key = self.generate_cache_key(&resolved_path, context);
        
        if let Some(cached) = self.cache.get(&cache_key).await? {
            return self.serve_from_cache(cached, headers).await;
        }
        
        // Process and serve
        let asset = self.process_and_serve(&resolved_path, headers).await?;
        
        // Cache if appropriate
        if self.should_cache(&asset) {
            self.cache.set(&cache_key, &asset).await?;
        }
        
        Ok(asset.into_response())
    }
    
    /// Process and serve asset
    async fn process_and_serve(
        &self,
        path: &PathBuf,
        headers: &HeaderMap,
    ) -> Result<ServedAsset> {
        // Get file metadata
        let metadata = tokio::fs::metadata(path).await?;
        
        // Check if modified
        if let Some(response) = self.check_not_modified(headers, &metadata).await? {
            return Ok(ServedAsset::NotModified(response));
        }
        
        // Determine if processing is needed
        let needs_processing = self.needs_processing(path);
        
        if needs_processing {
            // Process asset
            let processed = self.processor
                .process_asset(path, ProcessOptions::default())
                .await?;
            
            self.serve_processed_asset(processed, &metadata).await
        } else {
            // Serve directly
            self.serve_static_file(path, &metadata).await
        }
    }
    
    /// Serve static file
    async fn serve_static_file(
        &self,
        path: &PathBuf,
        metadata: &std::fs::Metadata,
    ) -> Result<ServedAsset> {
        // Read file
        let content = tokio::fs::read(path).await?;
        
        // Determine content type
        let content_type = mime_guess::from_path(path)
            .first_or_octet_stream();
        
        // Generate ETag
        let etag = self.generate_etag(&content, metadata);
        
        // Build response
        let response = Response::builder()
            .status(StatusCode::OK)
            .header(header::CONTENT_TYPE, content_type.as_ref())
            .header(header::CONTENT_LENGTH, content.len())
            .header(header::ETAG, etag.clone())
            .header(header::LAST_MODIFIED, httpdate::fmt_http_date(
                metadata.modified().unwrap_or_else(|_| std::time::SystemTime::now())
            ))
            .header(header::CACHE_CONTROL, self.get_cache_control(path))
            .body(Body::from(content))?;
        
        Ok(ServedAsset::Full(response, etag))
    }
    
    /// Generate ETag for asset
    fn generate_etag(
        &self,
        content: &[u8],
        metadata: &std::fs::Metadata,
    ) -> String {
        use sha2::{Sha256, Digest};
        
        let mut hasher = Sha256::new();
        hasher.update(content);
        hasher.update(metadata.len().to_le_bytes());
        
        if let Ok(modified) = metadata.modified() {
            if let Ok(duration) = modified.duration_since(std::time::UNIX_EPOCH) {
                hasher.update(duration.as_secs().to_le_bytes());
            }
        }
        
        format!("\"{:x}\"", hasher.finalize())
    }
}

/// Served asset types
enum ServedAsset {
    Full(Response, String),
    NotModified(Response),
}

impl IntoResponse for ServedAsset {
    fn into_response(self) -> Response {
        match self {
            ServedAsset::Full(response, _) => response,
            ServedAsset::NotModified(response) => response,
        }
    }
}
```

## Asset Resolution

```rust
/// Asset resolver with EPC support
pub struct AssetResolver {
    config: AssetServerConfig,
    epc_resolver: Arc<EpcResolver>,
    search_paths: Vec<PathBuf>,
}

impl AssetResolver {
    pub fn new(
        config: AssetServerConfig,
        epc_resolver: Arc<EpcResolver>,
    ) -> Self {
        let search_paths = vec![
            config.assets_dir.clone(),
            config.themes_dir.clone(),
            config.public_dir.clone(),
        ];
        
        Self {
            config,
            epc_resolver,
            search_paths,
        }
    }
    
    /// Resolve asset path through theme chain
    pub async fn resolve_asset_path(
        &self,
        request_path: &str,
        theme: &str,
    ) -> Result<Option<PathBuf>> {
        // Security: prevent directory traversal
        let safe_path = self.sanitize_path(request_path)?;
        
        // Try theme-specific assets first
        if let Some(theme_asset) = self.resolve_theme_asset(&safe_path, theme).await? {
            return Ok(Some(theme_asset));
        }
        
        // Try global assets
        if let Some(global_asset) = self.resolve_global_asset(&safe_path).await? {
            return Ok(Some(global_asset));
        }
        
        // Try public directory
        if let Some(public_asset) = self.resolve_public_asset(&safe_path).await? {
            return Ok(Some(public_asset));
        }
        
        Ok(None)
    }
    
    /// Resolve theme-specific asset
    async fn resolve_theme_asset(
        &self,
        path: &str,
        theme: &str,
    ) -> Result<Option<PathBuf>> {
        // Get theme chain
        let theme_chain = self.epc_resolver
            .get_theme_chain(theme)
            .await?;
        
        // Try each theme in order
        for theme_name in theme_chain {
            let theme_asset_path = self.config.themes_dir
                .join(&theme_name)
                .join("assets")
                .join(path);
            
            if theme_asset_path.exists() && theme_asset_path.is_file() {
                return Ok(Some(theme_asset_path));
            }
        }
        
        Ok(None)
    }
    
    /// Sanitize path to prevent traversal
    fn sanitize_path(&self, path: &str) -> Result<String> {
        // Remove leading slashes
        let path = path.trim_start_matches('/');
        
        // Check for path traversal
        if path.contains("..") || path.contains("./") {
            return Err(ReedError::InvalidPath(
                "Path traversal detected".to_string()
            ));
        }
        
        // Normalize path
        let normalized = path.replace('\\', "/");
        
        Ok(normalized)
    }
}
```

## Conditional Requests

```rust
impl AssetServer {
    /// Check for conditional requests
    async fn check_not_modified(
        &self,
        headers: &HeaderMap,
        metadata: &std::fs::Metadata,
    ) -> Result<Option<Response>> {
        // Check If-None-Match
        if let Some(if_none_match) = headers.get(header::IF_NONE_MATCH) {
            if let Ok(client_etag) = if_none_match.to_str() {
                // Generate current ETag
                let current_etag = self.generate_etag_from_metadata(metadata);
                
                if client_etag == current_etag {
                    return Ok(Some(self.not_modified_response(current_etag)));
                }
            }
        }
        
        // Check If-Modified-Since
        if let Some(if_modified_since) = headers.get(header::IF_MODIFIED_SINCE) {
            if let Ok(client_date_str) = if_modified_since.to_str() {
                if let Ok(client_date) = httpdate::parse_http_date(client_date_str) {
                    if let Ok(file_modified) = metadata.modified() {
                        if file_modified <= client_date {
                            let etag = self.generate_etag_from_metadata(metadata);
                            return Ok(Some(self.not_modified_response(etag)));
                        }
                    }
                }
            }
        }
        
        Ok(None)
    }
    
    /// Generate 304 Not Modified response
    fn not_modified_response(&self, etag: String) -> Response {
        Response::builder()
            .status(StatusCode::NOT_MODIFIED)
            .header(header::ETAG, etag)
            .header(header::CACHE_CONTROL, "public, max-age=3600")
            .body(Body::empty())
            .unwrap()
    }
    
    /// Generate ETag from metadata only
    fn generate_etag_from_metadata(
        &self,
        metadata: &std::fs::Metadata,
    ) -> String {
        use sha2::{Sha256, Digest};
        
        let mut hasher = Sha256::new();
        hasher.update(metadata.len().to_le_bytes());
        
        if let Ok(modified) = metadata.modified() {
            if let Ok(duration) = modified.duration_since(std::time::UNIX_EPOCH) {
                hasher.update(duration.as_secs().to_le_bytes());
            }
        }
        
        format!("\"{:x}\"", hasher.finalize())
    }
}
```

## Range Requests

```rust
/// Handle HTTP range requests for large files
pub struct RangeHandler;

impl RangeHandler {
    /// Parse Range header
    pub fn parse_range_header(
        header: &str,
        file_size: u64,
    ) -> Result<Vec<Range>> {
        if !header.starts_with("bytes=") {
            return Err(ReedError::InvalidRange("Invalid range unit".into()));
        }
        
        let ranges_str = &header[6..];
        let mut ranges = Vec::new();
        
        for range_str in ranges_str.split(',') {
            let range_str = range_str.trim();
            
            if let Some((start_str, end_str)) = range_str.split_once('-') {
                let start = if start_str.is_empty() {
                    // Suffix range: -500 means last 500 bytes
                    let suffix_len: u64 = end_str.parse()
                        .map_err(|_| ReedError::InvalidRange("Invalid suffix".into()))?;
                    file_size.saturating_sub(suffix_len)
                } else {
                    start_str.parse()
                        .map_err(|_| ReedError::InvalidRange("Invalid start".into()))?
                };
                
                let end = if end_str.is_empty() {
                    file_size - 1
                } else {
                    end_str.parse::<u64>()
                        .map_err(|_| ReedError::InvalidRange("Invalid end".into()))?
                        .min(file_size - 1)
                };
                
                if start <= end && start < file_size {
                    ranges.push(Range { start, end });
                }
            }
        }
        
        if ranges.is_empty() {
            return Err(ReedError::InvalidRange("No valid ranges".into()));
        }
        
        Ok(ranges)
    }
    
    /// Serve partial content
    pub async fn serve_partial(
        path: &PathBuf,
        ranges: Vec<Range>,
        content_type: &mime::Mime,
    ) -> Result<Response> {
        if ranges.len() == 1 {
            // Single range
            Self::serve_single_range(path, &ranges[0], content_type).await
        } else {
            // Multiple ranges - multipart response
            Self::serve_multipart_ranges(path, ranges, content_type).await
        }
    }
    
    /// Serve single range
    async fn serve_single_range(
        path: &PathBuf,
        range: &Range,
        content_type: &mime::Mime,
    ) -> Result<Response> {
        let mut file = tokio::fs::File::open(path).await?;
        let file_size = file.metadata().await?.len();
        
        // Seek to start
        file.seek(std::io::SeekFrom::Start(range.start)).await?;
        
        // Read range
        let length = range.end - range.start + 1;
        let mut buffer = vec![0; length as usize];
        file.read_exact(&mut buffer).await?;
        
        // Build response
        Ok(Response::builder()
            .status(StatusCode::PARTIAL_CONTENT)
            .header(header::CONTENT_TYPE, content_type.as_ref())
            .header(header::CONTENT_LENGTH, length)
            .header(header::CONTENT_RANGE, format!(
                "bytes {}-{}/{}",
                range.start,
                range.end,
                file_size
            ))
            .body(Body::from(buffer))?)
    }
}

#[derive(Debug, Clone)]
struct Range {
    start: u64,
    end: u64,
}
```

## Asset Caching

```rust
use lru::LruCache;
use tokio::sync::Mutex;

/// In-memory asset cache
pub struct AssetCache {
    cache: Arc<Mutex<LruCache<String, CachedAsset>>>,
    max_size: usize,
    ttl: Duration,
}

impl AssetCache {
    pub fn new(max_size: usize, ttl: u64) -> Self {
        Self {
            cache: Arc::new(Mutex::new(LruCache::new(
                NonZeroUsize::new(max_size).unwrap()
            ))),
            max_size,
            ttl: Duration::from_secs(ttl),
        }
    }
    
    /// Get cached asset
    pub async fn get(&self, key: &str) -> Result<Option<CachedAsset>> {
        let mut cache = self.cache.lock().await;
        
        if let Some(cached) = cache.get(key) {
            // Check if expired
            if cached.is_expired() {
                cache.pop(key);
                return Ok(None);
            }
            
            return Ok(Some(cached.clone()));
        }
        
        Ok(None)
    }
    
    /// Set cached asset
    pub async fn set(&self, key: &str, asset: &ServedAsset) -> Result<()> {
        if let ServedAsset::Full(response, etag) = asset {
            // Extract cacheable data
            let cached = CachedAsset::from_response(response, etag)?;
            
            let mut cache = self.cache.lock().await;
            cache.put(key.to_string(), cached);
        }
        
        Ok(())
    }
}

#[derive(Debug, Clone)]
struct CachedAsset {
    content: Vec<u8>,
    content_type: String,
    etag: String,
    headers: Vec<(String, String)>,
    cached_at: Instant,
    ttl: Duration,
}

impl CachedAsset {
    /// Check if cache entry is expired
    fn is_expired(&self) -> bool {
        self.cached_at.elapsed() > self.ttl
    }
    
    /// Convert to response
    fn to_response(&self) -> Response {
        let mut response = Response::builder()
            .status(StatusCode::OK)
            .header(header::CONTENT_TYPE, &self.content_type)
            .header(header::CONTENT_LENGTH, self.content.len())
            .header(header::ETAG, &self.etag)
            .header("X-Cache", "HIT");
        
        // Add cached headers
        for (key, value) in &self.headers {
            response = response.header(key, value);
        }
        
        response.body(Body::from(self.content.clone())).unwrap()
    }
}
```

## CDN Integration

```rust
/// CDN headers and integration
pub struct CdnIntegration {
    config: CdnConfig,
}

impl CdnIntegration {
    /// Add CDN headers to response
    pub fn add_cdn_headers(
        &self,
        response: Response,
        asset_type: AssetType,
    ) -> Response {
        let (mut parts, body) = response.into_parts();
        
        // Set cache control based on asset type
        let cache_control = match asset_type {
            AssetType::Image => "public, max-age=31536000, immutable",
            AssetType::Css => "public, max-age=86400, stale-while-revalidate=604800",
            AssetType::Js => "public, max-age=86400, stale-while-revalidate=604800",
            AssetType::Font => "public, max-age=31536000, immutable",
            AssetType::Other => "public, max-age=3600",
        };
        
        parts.headers.insert(
            header::CACHE_CONTROL,
            HeaderValue::from_static(cache_control),
        );
        
        // Add CDN-specific headers
        if self.config.enabled {
            // Cloudflare
            parts.headers.insert(
                "CDN-Cache-Control",
                HeaderValue::from_str(cache_control).unwrap(),
            );
            
            // Fastly
            parts.headers.insert(
                "Surrogate-Control",
                HeaderValue::from_str(&format!("max-age={}", 
                    self.config.edge_ttl)).unwrap(),
            );
            
            // Generic CDN key for purging
            parts.headers.insert(
                "Surrogate-Key",
                HeaderValue::from_str(&self.generate_surrogate_key(&asset_type)).unwrap(),
            );
        }
        
        // Add CORS headers for CDN
        parts.headers.insert(
            header::ACCESS_CONTROL_ALLOW_ORIGIN,
            HeaderValue::from_static("*"),
        );
        
        // Timing-Allow-Origin for performance monitoring
        parts.headers.insert(
            "Timing-Allow-Origin",
            HeaderValue::from_static("*"),
        );
        
        Response::from_parts(parts, body)
    }
    
    /// Generate surrogate key for cache invalidation
    fn generate_surrogate_key(&self, asset_type: &AssetType) -> String {
        format!("asset-{:?}", asset_type).to_lowercase()
    }
}

#[derive(Debug, Clone)]
pub enum AssetType {
    Image,
    Css,
    Js,
    Font,
    Other,
}

#[derive(Debug, Clone)]
pub struct CdnConfig {
    pub enabled: bool,
    pub edge_ttl: u64,
    pub browser_ttl: u64,
    pub stale_while_revalidate: u64,
}
```

## Asset Routes

```rust
/// Configure asset routes
pub fn configure_asset_routes() -> Router {
    Router::new()
        // Theme assets
        .route("/theme/*path", get(serve_theme_asset))
        
        // Processed assets
        .route("/assets/*path", get(serve_processed_asset))
        
        // Raw uploads
        .route("/uploads/*path", get(serve_upload))
        
        // Manifest
        .route("/assets/manifest.json", get(serve_manifest))
}

/// Serve theme asset
async fn serve_theme_asset(
    Path(path): Path<String>,
    headers: HeaderMap,
    State(ctx): State<Arc<ServerContext>>,
    Extension(req_ctx): Extension<RequestContext>,
) -> Result<Response, StatusCode> {
    ctx.asset_server
        .serve_asset(&path, &headers, &req_ctx)
        .await
        .map_err(|_| StatusCode::NOT_FOUND)
}

/// Serve processed asset
async fn serve_processed_asset(
    Path(path): Path<String>,
    headers: HeaderMap,
    Query(params): Query<AssetParams>,
    State(ctx): State<Arc<ServerContext>>,
) -> Result<Response, StatusCode> {
    // Build process options from query params
    let options = ProcessOptions {
        resize: params.width.zip(params.height).map(|(w, h)| ResizeOptions {
            width: w,
            height: h,
            mode: params.mode.unwrap_or(ResizeMode::Fit),
            filter: ResizeFilter::Lanczos3,
        }),
        quality: params.quality,
        format: params.format.and_then(|f| ImageFormat::from_extension(&f)),
        minify: Some(true),
    };
    
    // Process and serve
    ctx.asset_processor
        .process_and_serve(&path, options, &headers)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
}

#[derive(Deserialize)]
struct AssetParams {
    width: Option<u32>,
    height: Option<u32>,
    mode: Option<ResizeMode>,
    quality: Option<u8>,
    format: Option<String>,
}
```

## Summary

Dieses Asset Serving System bietet:
- **EPC Resolution** - Theme-aware Asset Loading
- **Conditional Requests** - ETag und If-Modified-Since
- **Range Requests** - Partial Content für große Files
- **Memory Caching** - LRU Cache für häufige Assets
- **CDN Integration** - Optimierte Headers für Edge Caching
- **Security** - Path Traversal Protection

Effizientes und sicheres Asset Serving.
# API Extensions

## Overview

API Extensions Implementation für ReedCMS. GraphQL, REST API Versioning und Custom Endpoints.

## GraphQL API

```rust
use async_graphql::{
    Context, EmptyMutation, EmptySubscription, Object, Schema, SimpleObject,
    ID, InputObject, Result as GraphQLResult,
};
use std::sync::Arc;

/// GraphQL schema definition
pub type ReedSchema = Schema<QueryRoot, MutationRoot, SubscriptionRoot>;

/// Build GraphQL schema
pub fn build_schema(app_state: Arc<AppState>) -> ReedSchema {
    Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
        .data(app_state)
        .finish()
}

/// Query root
pub struct QueryRoot;

#[Object]
impl QueryRoot {
    /// Get content by ID
    async fn content(
        &self,
        ctx: &Context<'_>,
        id: ID,
    ) -> GraphQLResult<Option<Content>> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let content_id = ContentId::from_str(&id)?;
        
        let content = app_state
            .content_service
            .get_content(&content_id)
            .await?;
        
        Ok(content.map(Into::into))
    }
    
    /// List content with filters
    async fn contents(
        &self,
        ctx: &Context<'_>,
        filter: Option<ContentFilter>,
        pagination: Option<PaginationInput>,
    ) -> GraphQLResult<ContentConnection> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        
        let filter = filter.unwrap_or_default();
        let pagination = pagination.unwrap_or_default();
        
        let (contents, total) = app_state
            .content_service
            .list_content(filter.into(), pagination.into())
            .await?;
        
        Ok(ContentConnection {
            edges: contents.into_iter().map(|c| ContentEdge {
                node: c.into(),
                cursor: base64::encode(&c.id.to_string()),
            }).collect(),
            page_info: PageInfo {
                has_next_page: pagination.offset + pagination.limit < total,
                has_previous_page: pagination.offset > 0,
                total_count: total,
            },
        })
    }
    
    /// Get user by ID
    async fn user(
        &self,
        ctx: &Context<'_>,
        id: ID,
    ) -> GraphQLResult<Option<User>> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let user_id = UserId::from_str(&id)?;
        
        let user = app_state
            .user_service
            .get_user(&user_id)
            .await?;
        
        Ok(user.map(Into::into))
    }
    
    /// Search content
    async fn search(
        &self,
        ctx: &Context<'_>,
        query: String,
        options: Option<SearchOptions>,
    ) -> GraphQLResult<SearchResults> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let options = options.unwrap_or_default();
        
        let results = app_state
            .search_service
            .search(&query, options.into())
            .await?;
        
        Ok(results.into())
    }
}

/// Mutation root
pub struct MutationRoot;

#[Object]
impl MutationRoot {
    /// Create content
    async fn create_content(
        &self,
        ctx: &Context<'_>,
        input: CreateContentInput,
    ) -> GraphQLResult<Content> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let user = ctx.data::<AuthUser>()?;
        
        // Check permissions
        require_permission(&user, "content.create")?;
        
        let content = app_state
            .content_service
            .create_content(input.into(), &user.id)
            .await?;
        
        Ok(content.into())
    }
    
    /// Update content
    async fn update_content(
        &self,
        ctx: &Context<'_>,
        id: ID,
        input: UpdateContentInput,
    ) -> GraphQLResult<Content> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let user = ctx.data::<AuthUser>()?;
        let content_id = ContentId::from_str(&id)?;
        
        // Check permissions
        require_permission(&user, "content.update")?;
        
        let content = app_state
            .content_service
            .update_content(&content_id, input.into(), &user.id)
            .await?;
        
        Ok(content.into())
    }
    
    /// Delete content
    async fn delete_content(
        &self,
        ctx: &Context<'_>,
        id: ID,
    ) -> GraphQLResult<bool> {
        let app_state = ctx.data::<Arc<AppState>>()?;
        let user = ctx.data::<AuthUser>()?;
        let content_id = ContentId::from_str(&id)?;
        
        // Check permissions
        require_permission(&user, "content.delete")?;
        
        app_state
            .content_service
            .delete_content(&content_id, &user.id)
            .await?;
        
        Ok(true)
    }
}

/// Subscription root
pub struct SubscriptionRoot;

#[Object]
impl SubscriptionRoot {
    /// Subscribe to content changes
    async fn content_changed(
        &self,
        ctx: &Context<'_>,
        content_type: Option<String>,
    ) -> impl Stream<Item = ContentEvent> {
        let app_state = ctx.data::<Arc<AppState>>().unwrap();
        
        app_state
            .event_bus
            .subscribe("content.*")
            .filter(move |event| {
                if let Some(ref ct) = content_type {
                    event.data.get("content_type")
                        .and_then(|v| v.as_str())
                        .map(|t| t == ct)
                        .unwrap_or(false)
                } else {
                    true
                }
            })
            .map(|event| ContentEvent {
                event_type: event.event_type,
                content_id: event.data["content_id"].as_str().unwrap().to_string(),
                timestamp: event.timestamp,
            })
    }
}

/// GraphQL types
#[derive(SimpleObject)]
struct Content {
    id: ID,
    content_type: String,
    title: String,
    slug: String,
    status: ContentStatus,
    fields: serde_json::Value,
    created_at: chrono::DateTime<chrono::Utc>,
    updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(SimpleObject)]
struct ContentConnection {
    edges: Vec<ContentEdge>,
    page_info: PageInfo,
}

#[derive(SimpleObject)]
struct ContentEdge {
    node: Content,
    cursor: String,
}

#[derive(SimpleObject)]
struct PageInfo {
    has_next_page: bool,
    has_previous_page: bool,
    total_count: usize,
}

#[derive(InputObject)]
struct ContentFilter {
    content_type: Option<String>,
    status: Option<ContentStatus>,
    search: Option<String>,
}

#[derive(InputObject)]
struct PaginationInput {
    offset: usize,
    limit: usize,
}

#[derive(InputObject)]
struct CreateContentInput {
    content_type: String,
    title: String,
    slug: Option<String>,
    fields: serde_json::Value,
    status: Option<ContentStatus>,
}
```

## REST API Versioning

```rust
use axum::{
    extract::{Path, Query, State},
    response::{IntoResponse, Response},
    Json, Router,
};
use semver::Version;

/// API version manager
pub struct ApiVersionManager {
    versions: HashMap<Version, VersionedApi>,
    default_version: Version,
}

impl ApiVersionManager {
    pub fn new() -> Self {
        let mut manager = Self {
            versions: HashMap::new(),
            default_version: Version::new(1, 0, 0),
        };
        
        // Register API versions
        manager.register_v1();
        manager.register_v2();
        
        manager
    }
    
    /// Build versioned router
    pub fn build_router(&self) -> Router {
        let mut router = Router::new();
        
        // Add version routes
        for (version, api) in &self.versions {
            let version_str = format!("v{}", version.major);
            router = router.nest(&format!("/api/{}", version_str), api.router());
        }
        
        // Add version negotiation middleware
        router = router.layer(axum::middleware::from_fn(version_negotiation));
        
        router
    }
    
    /// Register v1 API
    fn register_v1(&mut self) {
        let version = Version::new(1, 0, 0);
        let api = VersionedApi::new(version.clone());
        
        // v1 routes
        let router = Router::new()
            .route("/content", get(v1::list_content))
            .route("/content/:id", get(v1::get_content))
            .route("/content", post(v1::create_content))
            .route("/content/:id", put(v1::update_content))
            .route("/content/:id", delete(v1::delete_content));
        
        self.versions.insert(version, api.with_router(router));
    }
    
    /// Register v2 API
    fn register_v2(&mut self) {
        let version = Version::new(2, 0, 0);
        let api = VersionedApi::new(version.clone());
        
        // v2 routes with breaking changes
        let router = Router::new()
            .route("/contents", get(v2::list_contents)) // Plural
            .route("/contents/:id", get(v2::get_content))
            .route("/contents", post(v2::create_content))
            .route("/contents/:id", patch(v2::patch_content)) // PATCH instead of PUT
            .route("/contents/:id", delete(v2::delete_content))
            .route("/contents/bulk", post(v2::bulk_operations)); // New in v2
        
        self.versions.insert(version, api.with_router(router));
    }
}

/// Version negotiation middleware
async fn version_negotiation(
    req: Request,
    next: Next,
) -> Result<Response, ApiError> {
    // Check Accept header
    if let Some(accept) = req.headers().get("Accept") {
        if let Ok(accept_str) = accept.to_str() {
            if accept_str.contains("application/vnd.reed.") {
                // Extract version from media type
                // e.g., application/vnd.reed.v2+json
                if let Some(version) = extract_version_from_media_type(accept_str) {
                    req.extensions_mut().insert(ApiVersion(version));
                }
            }
        }
    }
    
    // Check version header
    if let Some(version_header) = req.headers().get("X-API-Version") {
        if let Ok(version_str) = version_header.to_str() {
            if let Ok(version) = Version::parse(version_str) {
                req.extensions_mut().insert(ApiVersion(version));
            }
        }
    }
    
    Ok(next.run(req).await)
}

/// API version deprecation
pub struct DeprecationNotice {
    version: Version,
    deprecated_at: chrono::DateTime<chrono::Utc>,
    sunset_at: chrono::DateTime<chrono::Utc>,
    migration_guide: String,
}

impl DeprecationNotice {
    /// Add deprecation headers to response
    pub fn add_headers(&self, response: &mut Response) {
        let headers = response.headers_mut();
        
        headers.insert(
            "Deprecation",
            HeaderValue::from_str(&self.deprecated_at.to_rfc3339()).unwrap(),
        );
        
        headers.insert(
            "Sunset",
            HeaderValue::from_str(&self.sunset_at.to_rfc3339()).unwrap(),
        );
        
        headers.insert(
            "Link",
            HeaderValue::from_str(&format!(
                "<{}>; rel=\"deprecation\"",
                self.migration_guide
            )).unwrap(),
        );
    }
}
```

## Custom Endpoints

```rust
/// Custom endpoint registry
pub struct CustomEndpointRegistry {
    endpoints: Arc<RwLock<HashMap<String, CustomEndpoint>>>,
    middleware: Vec<Box<dyn EndpointMiddleware>>,
}

impl CustomEndpointRegistry {
    pub fn new() -> Self {
        Self {
            endpoints: Arc::new(RwLock::new(HashMap::new())),
            middleware: Vec::new(),
        }
    }
    
    /// Register custom endpoint
    pub async fn register(
        &self,
        path: String,
        endpoint: CustomEndpoint,
    ) -> Result<()> {
        // Validate endpoint
        endpoint.validate()?;
        
        // Check for conflicts
        if self.endpoints.read().await.contains_key(&path) {
            return Err(ApiError::EndpointConflict(path));
        }
        
        // Register
        self.endpoints.write().await.insert(path, endpoint);
        
        Ok(())
    }
    
    /// Build router for custom endpoints
    pub async fn build_router(&self) -> Router {
        let mut router = Router::new();
        
        for (path, endpoint) in self.endpoints.read().await.iter() {
            router = router.route(
                path,
                endpoint.to_handler(),
            );
        }
        
        // Apply middleware
        for middleware in &self.middleware {
            router = router.layer(middleware.layer());
        }
        
        router
    }
}

/// Custom endpoint definition
pub struct CustomEndpoint {
    pub name: String,
    pub method: Method,
    pub handler: Box<dyn EndpointHandler>,
    pub validation: Option<ValidationSchema>,
    pub rate_limit: Option<RateLimit>,
    pub auth_required: bool,
    pub permissions: Vec<String>,
}

impl CustomEndpoint {
    /// Convert to axum handler
    pub fn to_handler(&self) -> axum::routing::MethodRouter {
        let handler = self.handler.clone();
        let validation = self.validation.clone();
        let auth_required = self.auth_required;
        let permissions = self.permissions.clone();
        
        let service = move |req: Request| {
            let handler = handler.clone();
            let validation = validation.clone();
            let permissions = permissions.clone();
            
            async move {
                // Authentication
                if auth_required {
                    let user = authenticate_request(&req).await?;
                    
                    // Check permissions
                    for permission in &permissions {
                        require_permission(&user, permission)?;
                    }
                    
                    req.extensions_mut().insert(user);
                }
                
                // Validation
                if let Some(schema) = validation {
                    validate_request(&req, &schema).await?;
                }
                
                // Execute handler
                handler.handle(req).await
            }
        };
        
        match self.method {
            Method::GET => get(service),
            Method::POST => post(service),
            Method::PUT => put(service),
            Method::DELETE => delete(service),
            Method::PATCH => patch(service),
            _ => panic!("Unsupported method"),
        }
    }
}

/// Endpoint handler trait
#[async_trait]
pub trait EndpointHandler: Send + Sync {
    async fn handle(&self, req: Request) -> Result<Response>;
    fn clone_box(&self) -> Box<dyn EndpointHandler>;
}

impl Clone for Box<dyn EndpointHandler> {
    fn clone(&self) -> Self {
        self.clone_box()
    }
}

/// Lambda-style endpoint handler
pub struct LambdaEndpoint<F> {
    handler: F,
}

impl<F> LambdaEndpoint<F>
where
    F: Fn(Request) -> BoxFuture<'static, Result<Response>> + Send + Sync + Clone + 'static,
{
    pub fn new(handler: F) -> Self {
        Self { handler }
    }
}

#[async_trait]
impl<F> EndpointHandler for LambdaEndpoint<F>
where
    F: Fn(Request) -> BoxFuture<'static, Result<Response>> + Send + Sync + Clone + 'static,
{
    async fn handle(&self, req: Request) -> Result<Response> {
        (self.handler)(req).await
    }
    
    fn clone_box(&self) -> Box<dyn EndpointHandler> {
        Box::new(self.clone())
    }
}
```

## API Documentation

```rust
use utoipa::{OpenApi, ToSchema};
use utoipa_swagger_ui::SwaggerUi;

/// OpenAPI documentation
#[derive(OpenApi)]
#[openapi(
    paths(
        list_content,
        get_content,
        create_content,
        update_content,
        delete_content,
    ),
    components(
        schemas(Content, CreateContentRequest, UpdateContentRequest)
    ),
    tags(
        (name = "content", description = "Content management endpoints")
    ),
    info(
        title = "ReedCMS API",
        version = "2.0.0",
        description = "Content Management System API",
        contact(
            name = "API Support",
            email = "api@reedcms.com"
        ),
        license(
            name = "MIT"
        )
    ),
    servers(
        (url = "https://api.reedcms.com", description = "Production server"),
        (url = "https://staging-api.reedcms.com", description = "Staging server")
    )
)]
struct ApiDoc;

/// Generate OpenAPI spec
pub fn generate_openapi_spec() -> String {
    ApiDoc::openapi().to_pretty_json().unwrap()
}

/// Serve Swagger UI
pub fn swagger_ui() -> SwaggerUi {
    SwaggerUi::new("/swagger-ui")
        .url("/api-docs/openapi.json", ApiDoc::openapi())
}

/// API client SDK generator
pub struct SdkGenerator {
    spec: OpenApi,
    templates: HashMap<String, Template>,
}

impl SdkGenerator {
    /// Generate TypeScript SDK
    pub fn generate_typescript(&self) -> Result<String> {
        let template = self.templates.get("typescript")
            .ok_or(ApiError::TemplateNotFound)?;
        
        let mut context = Context::new();
        context.insert("spec", &self.spec);
        context.insert("version", &self.spec.info.version);
        
        template.render(&context)
    }
    
    /// Generate Python SDK
    pub fn generate_python(&self) -> Result<String> {
        let template = self.templates.get("python")
            .ok_or(ApiError::TemplateNotFound)?;
        
        let mut context = Context::new();
        context.insert("spec", &self.spec);
        context.insert("version", &self.spec.info.version);
        
        template.render(&context)
    }
    
    /// Generate Go SDK
    pub fn generate_go(&self) -> Result<String> {
        let template = self.templates.get("go")
            .ok_or(ApiError::TemplateNotFound)?;
        
        let mut context = Context::new();
        context.insert("spec", &self.spec);
        context.insert("package_name", "reedcms");
        
        template.render(&context)
    }
}
```

## API Gateway Features

```rust
/// API gateway with advanced features
pub struct ApiGateway {
    rate_limiter: Arc<RateLimiter>,
    cache: Arc<ApiCache>,
    transformer: Arc<ResponseTransformer>,
    metrics: Arc<ApiMetrics>,
}

impl ApiGateway {
    /// Gateway middleware
    pub fn middleware<S>(&self) -> impl Layer<S> + Clone {
        ServiceBuilder::new()
            .layer(self.rate_limiting_layer())
            .layer(self.caching_layer())
            .layer(self.transformation_layer())
            .layer(self.metrics_layer())
    }
    
    /// Rate limiting layer
    fn rate_limiting_layer(&self) -> RateLimitingLayer {
        RateLimitingLayer::new(self.rate_limiter.clone())
    }
    
    /// Response caching layer
    fn caching_layer(&self) -> CachingLayer {
        CachingLayer::new(self.cache.clone())
    }
    
    /// Response transformation layer
    fn transformation_layer(&self) -> TransformationLayer {
        TransformationLayer::new(self.transformer.clone())
    }
}

/// Response transformer
pub struct ResponseTransformer {
    transformers: Vec<Box<dyn Transformer>>,
}

impl ResponseTransformer {
    /// Transform response
    pub async fn transform(
        &self,
        mut response: Response,
        context: &TransformContext,
    ) -> Result<Response> {
        for transformer in &self.transformers {
            response = transformer.transform(response, context).await?;
        }
        
        Ok(response)
    }
}

/// Transformer trait
#[async_trait]
trait Transformer: Send + Sync {
    async fn transform(
        &self,
        response: Response,
        context: &TransformContext,
    ) -> Result<Response>;
}

/// Field filtering transformer
struct FieldFilterTransformer;

#[async_trait]
impl Transformer for FieldFilterTransformer {
    async fn transform(
        &self,
        response: Response,
        context: &TransformContext,
    ) -> Result<Response> {
        if let Some(fields) = &context.requested_fields {
            // Parse response body
            let body = response.into_body();
            let json: serde_json::Value = serde_json::from_slice(&body)?;
            
            // Filter fields
            let filtered = filter_fields(json, fields);
            
            // Rebuild response
            Ok(Response::new(Body::from(serde_json::to_vec(&filtered)?)))
        } else {
            Ok(response)
        }
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ApiExtensionsConfig {
    pub graphql: GraphQLConfig,
    pub rest: RestApiConfig,
    pub custom_endpoints: CustomEndpointConfig,
    pub gateway: GatewayConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GraphQLConfig {
    pub enabled: bool,
    pub playground: bool,
    pub introspection: bool,
    pub max_depth: usize,
    pub max_complexity: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RestApiConfig {
    pub versioning: VersioningStrategy,
    pub default_version: String,
    pub deprecated_versions: Vec<String>,
    pub response_format: ResponseFormat,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum VersioningStrategy {
    Path,
    Header,
    MediaType,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ResponseFormat {
    Json,
    JsonApi,
    Hal,
}

impl Default for ApiExtensionsConfig {
    fn default() -> Self {
        Self {
            graphql: GraphQLConfig {
                enabled: true,
                playground: true,
                introspection: true,
                max_depth: 10,
                max_complexity: 100,
            },
            rest: RestApiConfig {
                versioning: VersioningStrategy::Path,
                default_version: "v2".to_string(),
                deprecated_versions: vec!["v1".to_string()],
                response_format: ResponseFormat::Json,
            },
            custom_endpoints: CustomEndpointConfig {
                enabled: true,
                max_endpoints: 100,
                require_authentication: true,
            },
            gateway: GatewayConfig {
                rate_limiting: true,
                caching: true,
                transformation: true,
                compression: true,
            },
        }
    }
}
```

## Summary

Diese API Extensions Implementation bietet:
- **GraphQL API** - Full GraphQL Support mit Subscriptions
- **REST Versioning** - Multiple API Versions, Deprecation
- **Custom Endpoints** - Dynamic Endpoint Registration
- **API Documentation** - OpenAPI, Swagger UI, SDK Generation
- **Gateway Features** - Rate Limiting, Caching, Transformation
- **Developer Experience** - Type-Safe, Well-Documented APIs

Complete API Extension System für flexible Integrations.
# Plugin API

## Overview

Plugin API Definition für ReedCMS. Stable API Surface für Plugin Developers mit Type Safety und Versioning.

## Core Plugin API

```rust
use serde::{Serialize, Deserialize};
use async_trait::async_trait;

/// Main plugin API trait that all plugins must implement
#[async_trait]
pub trait ReedPlugin: Send + Sync {
    /// Plugin initialization
    async fn init(&mut self, api: PluginApi) -> PluginResult<()>;
    
    /// Plugin activation
    async fn activate(&mut self) -> PluginResult<()> {
        Ok(())
    }
    
    /// Plugin deactivation
    async fn deactivate(&mut self) -> PluginResult<()> {
        Ok(())
    }
    
    /// Get plugin info
    fn info(&self) -> PluginInfo;
}

/// Plugin information
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginInfo {
    pub name: String,
    pub version: String,
    pub author: String,
    pub description: String,
    pub license: String,
    pub homepage: Option<String>,
    pub repository: Option<String>,
    pub keywords: Vec<String>,
    pub categories: Vec<String>,
    pub api_version: String,
}

/// Plugin result type
pub type PluginResult<T> = Result<T, PluginError>;

/// Plugin error types
#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("Initialization failed: {0}")]
    InitializationFailed(String),
    
    #[error("API version mismatch: expected {expected}, got {actual}")]
    ApiVersionMismatch { expected: String, actual: String },
    
    #[error("Permission denied: {0}")]
    PermissionDenied(String),
    
    #[error("Resource not found: {0}")]
    ResourceNotFound(String),
    
    #[error("Invalid argument: {0}")]
    InvalidArgument(String),
    
    #[error("Internal error: {0}")]
    Internal(String),
}
```

## Plugin API Interface

```rust
/// Main API interface provided to plugins
#[derive(Clone)]
pub struct PluginApi {
    content: ContentApi,
    storage: StorageApi,
    cache: CacheApi,
    http: HttpApi,
    events: EventApi,
    hooks: HookApi,
    logger: LoggerApi,
    config: ConfigApi,
}

impl PluginApi {
    /// Create new plugin API instance
    pub fn new(plugin_id: String, context: Arc<PluginContext>) -> Self {
        Self {
            content: ContentApi::new(plugin_id.clone(), context.clone()),
            storage: StorageApi::new(plugin_id.clone(), context.clone()),
            cache: CacheApi::new(plugin_id.clone(), context.clone()),
            http: HttpApi::new(plugin_id.clone(), context.clone()),
            events: EventApi::new(plugin_id.clone(), context.clone()),
            hooks: HookApi::new(plugin_id.clone(), context.clone()),
            logger: LoggerApi::new(plugin_id.clone()),
            config: ConfigApi::new(plugin_id.clone(), context.clone()),
        }
    }
    
    /// Get content API
    pub fn content(&self) -> &ContentApi {
        &self.content
    }
    
    /// Get storage API
    pub fn storage(&self) -> &StorageApi {
        &self.storage
    }
    
    /// Get cache API
    pub fn cache(&self) -> &CacheApi {
        &self.cache
    }
    
    /// Get HTTP API
    pub fn http(&self) -> &HttpApi {
        &self.http
    }
    
    /// Get events API
    pub fn events(&self) -> &EventApi {
        &self.events
    }
    
    /// Get hooks API
    pub fn hooks(&self) -> &HookApi {
        &self.hooks
    }
    
    /// Get logger API
    pub fn logger(&self) -> &LoggerApi {
        &self.logger
    }
    
    /// Get config API
    pub fn config(&self) -> &ConfigApi {
        &self.config
    }
}
```

## Content API

```rust
/// API for content operations
#[derive(Clone)]
pub struct ContentApi {
    plugin_id: String,
    context: Arc<PluginContext>,
}

impl ContentApi {
    /// Create new content
    pub async fn create(
        &self,
        content_type: &str,
        data: ContentData,
    ) -> PluginResult<ContentId> {
        // Check permissions
        self.check_permission("content.create").await?;
        
        // Validate content type
        if !self.is_valid_content_type(content_type).await? {
            return Err(PluginError::InvalidArgument(
                format!("Invalid content type: {}", content_type)
            ));
        }
        
        // Create content
        let id = self.context
            .content_manager
            .create(content_type, data, &self.plugin_id)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(ContentId(id))
    }
    
    /// Get content by ID
    pub async fn get(&self, id: ContentId) -> PluginResult<Option<Content>> {
        self.check_permission("content.read").await?;
        
        let content = self.context
            .content_manager
            .get_by_id(id.0)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(content.map(|c| Content::from_entity(c)))
    }
    
    /// Update content
    pub async fn update(
        &self,
        id: ContentId,
        data: ContentData,
    ) -> PluginResult<()> {
        self.check_permission("content.update").await?;
        
        self.context
            .content_manager
            .update(id.0, data, &self.plugin_id)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// Delete content
    pub async fn delete(&self, id: ContentId) -> PluginResult<()> {
        self.check_permission("content.delete").await?;
        
        self.context
            .content_manager
            .delete(id.0, &self.plugin_id)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// List content
    pub async fn list(
        &self,
        filter: ContentFilter,
    ) -> PluginResult<Vec<Content>> {
        self.check_permission("content.read").await?;
        
        let contents = self.context
            .content_manager
            .list(filter)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(contents.into_iter()
            .map(Content::from_entity)
            .collect())
    }
    
    /// Search content
    pub async fn search(
        &self,
        query: &str,
        options: SearchOptions,
    ) -> PluginResult<SearchResults> {
        self.check_permission("content.search").await?;
        
        let results = self.context
            .search_engine
            .search(query, options)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(SearchResults::from_internal(results))
    }
    
    /// Check permission
    async fn check_permission(&self, permission: &str) -> PluginResult<()> {
        if !self.context.permission_checker
            .check_plugin_permission(&self.plugin_id, permission)
            .await? {
            return Err(PluginError::PermissionDenied(
                format!("Plugin {} lacks permission: {}", self.plugin_id, permission)
            ));
        }
        Ok(())
    }
}

/// Content representation for plugins
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Content {
    pub id: ContentId,
    pub content_type: String,
    pub data: ContentData,
    pub metadata: ContentMetadata,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentId(pub Uuid);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentData(pub serde_json::Value);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentMetadata {
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
    pub created_by: String,
    pub updated_by: String,
    pub version: u32,
    pub status: ContentStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ContentStatus {
    Draft,
    Published,
    Archived,
}
```

## Storage API

```rust
/// API for plugin storage
#[derive(Clone)]
pub struct StorageApi {
    plugin_id: String,
    context: Arc<PluginContext>,
}

impl StorageApi {
    /// Store data
    pub async fn put(
        &self,
        key: &str,
        value: serde_json::Value,
    ) -> PluginResult<()> {
        let full_key = format!("plugin:{}:{}", self.plugin_id, key);
        
        self.context
            .storage
            .set(&full_key, value)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// Get data
    pub async fn get(
        &self,
        key: &str,
    ) -> PluginResult<Option<serde_json::Value>> {
        let full_key = format!("plugin:{}:{}", self.plugin_id, key);
        
        self.context
            .storage
            .get(&full_key)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))
    }
    
    /// Delete data
    pub async fn delete(&self, key: &str) -> PluginResult<()> {
        let full_key = format!("plugin:{}:{}", self.plugin_id, key);
        
        self.context
            .storage
            .delete(&full_key)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// List keys
    pub async fn list_keys(
        &self,
        prefix: Option<&str>,
    ) -> PluginResult<Vec<String>> {
        let pattern = match prefix {
            Some(p) => format!("plugin:{}:{}*", self.plugin_id, p),
            None => format!("plugin:{}:*", self.plugin_id),
        };
        
        let keys = self.context
            .storage
            .keys(&pattern)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        // Strip plugin prefix from keys
        let prefix_len = format!("plugin:{}:", self.plugin_id).len();
        Ok(keys.into_iter()
            .map(|k| k[prefix_len..].to_string())
            .collect())
    }
}
```

## Event API

```rust
/// API for event handling
#[derive(Clone)]
pub struct EventApi {
    plugin_id: String,
    context: Arc<PluginContext>,
}

impl EventApi {
    /// Emit event
    pub async fn emit(
        &self,
        event_type: &str,
        data: serde_json::Value,
    ) -> PluginResult<()> {
        let event = PluginEvent {
            id: Uuid::new_v4(),
            event_type: event_type.to_string(),
            source: self.plugin_id.clone(),
            timestamp: chrono::Utc::now(),
            data,
        };
        
        self.context
            .event_bus
            .publish(event)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// Subscribe to events
    pub async fn subscribe<F>(
        &self,
        event_type: &str,
        handler: F,
    ) -> PluginResult<SubscriptionId>
    where
        F: Fn(PluginEvent) -> BoxFuture<'static, PluginResult<()>> + Send + Sync + 'static,
    {
        let subscription_id = self.context
            .event_bus
            .subscribe_plugin(
                &self.plugin_id,
                event_type,
                Box::new(handler),
            )
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(SubscriptionId(subscription_id))
    }
    
    /// Unsubscribe from events
    pub async fn unsubscribe(
        &self,
        subscription_id: SubscriptionId,
    ) -> PluginResult<()> {
        self.context
            .event_bus
            .unsubscribe(subscription_id.0)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginEvent {
    pub id: Uuid,
    pub event_type: String,
    pub source: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub data: serde_json::Value,
}

#[derive(Debug, Clone)]
pub struct SubscriptionId(Uuid);
```

## Hook API

```rust
/// API for hook system
#[derive(Clone)]
pub struct HookApi {
    plugin_id: String,
    context: Arc<PluginContext>,
}

impl HookApi {
    /// Register hook handler
    pub async fn register<F, T, R>(
        &self,
        hook_name: &str,
        priority: i32,
        handler: F,
    ) -> PluginResult<HookId>
    where
        F: Fn(T) -> BoxFuture<'static, PluginResult<R>> + Send + Sync + 'static,
        T: DeserializeOwned + Send + Sync + 'static,
        R: Serialize + Send + Sync + 'static,
    {
        let hook_id = self.context
            .hook_manager
            .register_plugin_hook(
                &self.plugin_id,
                hook_name,
                priority,
                Box::new(move |data: serde_json::Value| {
                    Box::pin(async move {
                        let input: T = serde_json::from_value(data)
                            .map_err(|e| PluginError::InvalidArgument(e.to_string()))?;
                        
                        let result = handler(input).await?;
                        
                        serde_json::to_value(result)
                            .map_err(|e| PluginError::Internal(e.to_string()))
                    })
                }),
            )
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(HookId(hook_id))
    }
    
    /// Unregister hook
    pub async fn unregister(&self, hook_id: HookId) -> PluginResult<()> {
        self.context
            .hook_manager
            .unregister_hook(hook_id.0)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(())
    }
    
    /// Execute hook
    pub async fn execute<T, R>(
        &self,
        hook_name: &str,
        data: T,
    ) -> PluginResult<R>
    where
        T: Serialize + Send + Sync,
        R: DeserializeOwned + Send + Sync,
    {
        let input = serde_json::to_value(data)
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        let result = self.context
            .hook_manager
            .execute_hook(hook_name, input)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        serde_json::from_value(result)
            .map_err(|e| PluginError::Internal(e.to_string()))
    }
}

#[derive(Debug, Clone)]
pub struct HookId(Uuid);
```

## HTTP API

```rust
/// API for HTTP requests
#[derive(Clone)]
pub struct HttpApi {
    plugin_id: String,
    context: Arc<PluginContext>,
    client: reqwest::Client,
}

impl HttpApi {
    /// Make GET request
    pub async fn get(
        &self,
        url: &str,
        headers: Option<HashMap<String, String>>,
    ) -> PluginResult<HttpResponse> {
        self.check_network_permission().await?;
        
        let mut request = self.client.get(url);
        
        if let Some(headers) = headers {
            for (key, value) in headers {
                request = request.header(key, value);
            }
        }
        
        let response = request
            .send()
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(HttpResponse::from_reqwest(response).await?)
    }
    
    /// Make POST request
    pub async fn post(
        &self,
        url: &str,
        body: serde_json::Value,
        headers: Option<HashMap<String, String>>,
    ) -> PluginResult<HttpResponse> {
        self.check_network_permission().await?;
        
        let mut request = self.client.post(url).json(&body);
        
        if let Some(headers) = headers {
            for (key, value) in headers {
                request = request.header(key, value);
            }
        }
        
        let response = request
            .send()
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))?;
        
        Ok(HttpResponse::from_reqwest(response).await?)
    }
    
    /// Check network permission
    async fn check_network_permission(&self) -> PluginResult<()> {
        if !self.context.permission_checker
            .check_plugin_permission(&self.plugin_id, "network.access")
            .await? {
            return Err(PluginError::PermissionDenied(
                "Plugin lacks network access permission".to_string()
            ));
        }
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HttpResponse {
    pub status: u16,
    pub headers: HashMap<String, String>,
    pub body: serde_json::Value,
}
```

## Logger API

```rust
/// API for logging
#[derive(Clone)]
pub struct LoggerApi {
    plugin_id: String,
}

impl LoggerApi {
    /// Log debug message
    pub fn debug(&self, message: &str) {
        debug!("[Plugin {}] {}", self.plugin_id, message);
    }
    
    /// Log info message
    pub fn info(&self, message: &str) {
        info!("[Plugin {}] {}", self.plugin_id, message);
    }
    
    /// Log warning message
    pub fn warn(&self, message: &str) {
        warn!("[Plugin {}] {}", self.plugin_id, message);
    }
    
    /// Log error message
    pub fn error(&self, message: &str) {
        error!("[Plugin {}] {}", self.plugin_id, message);
    }
    
    /// Log with structured data
    pub fn log(
        &self,
        level: LogLevel,
        message: &str,
        data: Option<serde_json::Value>,
    ) {
        match level {
            LogLevel::Debug => {
                if let Some(data) = data {
                    debug!("[Plugin {}] {} {:?}", self.plugin_id, message, data);
                } else {
                    debug!("[Plugin {}] {}", self.plugin_id, message);
                }
            }
            LogLevel::Info => {
                if let Some(data) = data {
                    info!("[Plugin {}] {} {:?}", self.plugin_id, message, data);
                } else {
                    info!("[Plugin {}] {}", self.plugin_id, message);
                }
            }
            LogLevel::Warn => {
                if let Some(data) = data {
                    warn!("[Plugin {}] {} {:?}", self.plugin_id, message, data);
                } else {
                    warn!("[Plugin {}] {}", self.plugin_id, message);
                }
            }
            LogLevel::Error => {
                if let Some(data) = data {
                    error!("[Plugin {}] {} {:?}", self.plugin_id, message, data);
                } else {
                    error!("[Plugin {}] {}", self.plugin_id, message);
                }
            }
        }
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}
```

## Config API

```rust
/// API for plugin configuration
#[derive(Clone)]
pub struct ConfigApi {
    plugin_id: String,
    context: Arc<PluginContext>,
}

impl ConfigApi {
    /// Get plugin configuration
    pub async fn get(&self) -> PluginResult<serde_json::Value> {
        self.context
            .config_manager
            .get_plugin_config(&self.plugin_id)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))
    }
    
    /// Get specific config value
    pub async fn get_value<T>(&self, key: &str) -> PluginResult<Option<T>>
    where
        T: DeserializeOwned,
    {
        let config = self.get().await?;
        
        if let Some(value) = config.get(key) {
            serde_json::from_value(value.clone())
                .map(Some)
                .map_err(|e| PluginError::InvalidArgument(
                    format!("Invalid config value for key '{}': {}", key, e)
                ))
        } else {
            Ok(None)
        }
    }
    
    /// Update configuration
    pub async fn update(
        &self,
        updates: serde_json::Value,
    ) -> PluginResult<()> {
        self.context
            .config_manager
            .update_plugin_config(&self.plugin_id, updates)
            .await
            .map_err(|e| PluginError::Internal(e.to_string()))
    }
    
    /// Get system info
    pub async fn system_info(&self) -> PluginResult<SystemInfo> {
        Ok(SystemInfo {
            version: env!("CARGO_PKG_VERSION").to_string(),
            api_version: "1.0.0".to_string(),
            platform: std::env::consts::OS.to_string(),
            architecture: std::env::consts::ARCH.to_string(),
        })
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SystemInfo {
    pub version: String,
    pub api_version: String,
    pub platform: String,
    pub architecture: String,
}
```

## Plugin Development Helpers

```rust
/// Macro for defining plugin
#[macro_export]
macro_rules! reed_plugin {
    ($plugin_type:ty) => {
        #[no_mangle]
        pub extern "C" fn _reed_plugin_create() -> Box<dyn ReedPlugin> {
            Box::new(<$plugin_type>::default())
        }
        
        #[no_mangle]
        pub extern "C" fn _reed_plugin_api_version() -> &'static str {
            "1.0.0"
        }
    };
}

/// Plugin builder for easier development
pub struct PluginBuilder {
    info: PluginInfo,
    init_handler: Option<Box<dyn Fn(PluginApi) -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
    activate_handler: Option<Box<dyn Fn() -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
    deactivate_handler: Option<Box<dyn Fn() -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
}

impl PluginBuilder {
    pub fn new(name: &str, version: &str) -> Self {
        Self {
            info: PluginInfo {
                name: name.to_string(),
                version: version.to_string(),
                author: String::new(),
                description: String::new(),
                license: "MIT".to_string(),
                homepage: None,
                repository: None,
                keywords: Vec::new(),
                categories: Vec::new(),
                api_version: "1.0.0".to_string(),
            },
            init_handler: None,
            activate_handler: None,
            deactivate_handler: None,
        }
    }
    
    pub fn author(mut self, author: &str) -> Self {
        self.info.author = author.to_string();
        self
    }
    
    pub fn description(mut self, description: &str) -> Self {
        self.info.description = description.to_string();
        self
    }
    
    pub fn on_init<F>(mut self, handler: F) -> Self
    where
        F: Fn(PluginApi) -> BoxFuture<'static, PluginResult<()>> + Send + Sync + 'static,
    {
        self.init_handler = Some(Box::new(handler));
        self
    }
    
    pub fn build(self) -> impl ReedPlugin {
        BuiltPlugin {
            info: self.info,
            init_handler: self.init_handler,
            activate_handler: self.activate_handler,
            deactivate_handler: self.deactivate_handler,
        }
    }
}

struct BuiltPlugin {
    info: PluginInfo,
    init_handler: Option<Box<dyn Fn(PluginApi) -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
    activate_handler: Option<Box<dyn Fn() -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
    deactivate_handler: Option<Box<dyn Fn() -> BoxFuture<'static, PluginResult<()>> + Send + Sync>>,
}

#[async_trait]
impl ReedPlugin for BuiltPlugin {
    async fn init(&mut self, api: PluginApi) -> PluginResult<()> {
        if let Some(ref handler) = self.init_handler {
            handler(api).await
        } else {
            Ok(())
        }
    }
    
    fn info(&self) -> PluginInfo {
        self.info.clone()
    }
}
```

## Summary

Diese Plugin API bietet:
- **Type-Safe API** - Strongly Typed Interfaces
- **Permission System** - Granular Permission Control
- **Resource Access** - Content, Storage, Cache APIs
- **Event System** - Pub/Sub Event Handling
- **Hook System** - Extensibility Points
- **HTTP Client** - External API Integration
- **Configuration** - Plugin Settings Management

Stable und erweiterbare API für Plugin Development.
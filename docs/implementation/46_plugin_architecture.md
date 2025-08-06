# Plugin Architecture

## Overview

Plugin Architecture für ReedCMS. Extensible System mit Hook Points, Event System und isolierter Plugin Execution.

## Plugin System Core

```rust
use async_trait::async_trait;
use std::any::Any;
use std::collections::HashMap;
use tokio::sync::RwLock;

/// Main plugin manager
pub struct PluginManager {
    registry: Arc<RwLock<PluginRegistry>>,
    loader: Arc<PluginLoader>,
    hooks: Arc<HookManager>,
    events: Arc<EventBus>,
    config: PluginConfig,
}

impl PluginManager {
    pub fn new(config: PluginConfig) -> Self {
        Self {
            registry: Arc::new(RwLock::new(PluginRegistry::new())),
            loader: Arc::new(PluginLoader::new(&config)),
            hooks: Arc::new(HookManager::new()),
            events: Arc::new(EventBus::new()),
            config,
        }
    }
    
    /// Load and initialize plugins
    pub async fn initialize(&self) -> Result<()> {
        // Discover plugins
        let plugin_paths = self.discover_plugins().await?;
        
        // Load each plugin
        for path in plugin_paths {
            match self.load_plugin(&path).await {
                Ok(plugin_id) => {
                    info!("Loaded plugin: {}", plugin_id);
                }
                Err(e) => {
                    error!("Failed to load plugin {}: {}", path.display(), e);
                    if self.config.fail_on_error {
                        return Err(e);
                    }
                }
            }
        }
        
        // Initialize all loaded plugins
        self.initialize_plugins().await?;
        
        Ok(())
    }
    
    /// Load single plugin
    async fn load_plugin(&self, path: &Path) -> Result<String> {
        // Load plugin metadata
        let metadata = self.loader.load_metadata(path).await?;
        
        // Check dependencies
        self.check_dependencies(&metadata).await?;
        
        // Load plugin code
        let plugin = self.loader.load_plugin(path, &metadata).await?;
        
        // Register plugin
        let plugin_id = metadata.id.clone();
        self.registry.write().await.register(plugin_id.clone(), plugin);
        
        Ok(plugin_id)
    }
    
    /// Initialize all plugins
    async fn initialize_plugins(&self) -> Result<()> {
        let plugins = self.registry.read().await.get_all_plugins();
        
        // Sort by priority
        let mut sorted_plugins = plugins;
        sorted_plugins.sort_by_key(|p| p.metadata().priority);
        
        // Initialize in order
        for plugin in sorted_plugins {
            let context = self.create_plugin_context(&plugin.metadata().id);
            plugin.initialize(context).await?;
        }
        
        Ok(())
    }
    
    /// Create plugin context
    fn create_plugin_context(&self, plugin_id: &str) -> PluginContext {
        PluginContext {
            plugin_id: plugin_id.to_string(),
            hooks: self.hooks.clone(),
            events: self.events.clone(),
            config: self.get_plugin_config(plugin_id),
        }
    }
}

/// Plugin registry
struct PluginRegistry {
    plugins: HashMap<String, Box<dyn Plugin>>,
    metadata: HashMap<String, PluginMetadata>,
}

impl PluginRegistry {
    fn new() -> Self {
        Self {
            plugins: HashMap::new(),
            metadata: HashMap::new(),
        }
    }
    
    fn register(&mut self, id: String, plugin: Box<dyn Plugin>) {
        let metadata = plugin.metadata().clone();
        self.metadata.insert(id.clone(), metadata);
        self.plugins.insert(id, plugin);
    }
    
    fn get_plugin(&self, id: &str) -> Option<&dyn Plugin> {
        self.plugins.get(id).map(|p| p.as_ref())
    }
    
    fn get_all_plugins(&self) -> Vec<&Box<dyn Plugin>> {
        self.plugins.values().collect()
    }
}
```

## Plugin Trait

```rust
/// Core plugin trait
#[async_trait]
pub trait Plugin: Send + Sync {
    /// Get plugin metadata
    fn metadata(&self) -> &PluginMetadata;
    
    /// Initialize plugin
    async fn initialize(&self, context: PluginContext) -> Result<()>;
    
    /// Shutdown plugin
    async fn shutdown(&self) -> Result<()>;
    
    /// Handle custom command
    async fn handle_command(
        &self,
        command: &str,
        args: serde_json::Value,
    ) -> Result<serde_json::Value> {
        Err(ReedError::NotImplemented(
            format!("Command {} not implemented", command)
        ))
    }
}

/// Plugin metadata
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginMetadata {
    pub id: String,
    pub name: String,
    pub version: String,
    pub author: String,
    pub description: String,
    pub priority: i32,
    pub dependencies: Vec<PluginDependency>,
    pub capabilities: Vec<PluginCapability>,
    pub config_schema: Option<serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginDependency {
    pub plugin_id: String,
    pub version: String,
    pub optional: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum PluginCapability {
    Hook(String),
    Event(String),
    Command(String),
    ContentType(String),
    TemplateFunction(String),
}

/// Plugin context provided to plugins
pub struct PluginContext {
    pub plugin_id: String,
    pub hooks: Arc<HookManager>,
    pub events: Arc<EventBus>,
    pub config: serde_json::Value,
}
```

## Hook System

```rust
/// Hook manager for plugin integration
pub struct HookManager {
    hooks: Arc<RwLock<HashMap<String, Vec<HookHandler>>>>,
}

impl HookManager {
    pub fn new() -> Self {
        Self {
            hooks: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Register hook handler
    pub async fn register_hook(
        &self,
        hook_name: &str,
        handler: HookHandler,
    ) -> Result<()> {
        let mut hooks = self.hooks.write().await;
        hooks.entry(hook_name.to_string())
            .or_insert_with(Vec::new)
            .push(handler);
        Ok(())
    }
    
    /// Execute hook
    pub async fn execute_hook<T, R>(
        &self,
        hook_name: &str,
        data: T,
    ) -> Result<R>
    where
        T: Send + Sync + 'static,
        R: Send + Sync + 'static,
    {
        let hooks = self.hooks.read().await;
        
        if let Some(handlers) = hooks.get(hook_name) {
            let mut result = Box::new(data) as Box<dyn Any + Send + Sync>;
            
            for handler in handlers {
                result = handler.execute(result).await?;
            }
            
            result.downcast::<R>()
                .map(|r| *r)
                .map_err(|_| ReedError::TypeMismatch(
                    "Hook result type mismatch".to_string()
                ))
        } else {
            // No hooks registered, return original data
            Box::new(data).downcast::<R>()
                .map(|r| *r)
                .map_err(|_| ReedError::TypeMismatch(
                    "Hook data type mismatch".to_string()
                ))
        }
    }
}

/// Hook handler
struct HookHandler {
    plugin_id: String,
    priority: i32,
    handler: Arc<dyn Fn(Box<dyn Any + Send + Sync>) -> BoxFuture<'static, Result<Box<dyn Any + Send + Sync>>> + Send + Sync>,
}

impl HookHandler {
    async fn execute(
        &self,
        data: Box<dyn Any + Send + Sync>,
    ) -> Result<Box<dyn Any + Send + Sync>> {
        (self.handler)(data).await
    }
}

/// Common hook points
pub struct Hooks;

impl Hooks {
    pub const BEFORE_RENDER: &'static str = "before_render";
    pub const AFTER_RENDER: &'static str = "after_render";
    pub const BEFORE_SAVE: &'static str = "before_save";
    pub const AFTER_SAVE: &'static str = "after_save";
    pub const BEFORE_DELETE: &'static str = "before_delete";
    pub const AFTER_DELETE: &'static str = "after_delete";
    pub const VALIDATE_DATA: &'static str = "validate_data";
    pub const TRANSFORM_DATA: &'static str = "transform_data";
    pub const ROUTE_MATCH: &'static str = "route_match";
    pub const AUTH_CHECK: &'static str = "auth_check";
}
```

## Event System

```rust
use tokio::sync::broadcast;

/// Event bus for plugin communication
pub struct EventBus {
    sender: broadcast::Sender<Event>,
    subscriber_count: Arc<AtomicUsize>,
}

impl EventBus {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(1024);
        Self {
            sender,
            subscriber_count: Arc::new(AtomicUsize::new(0)),
        }
    }
    
    /// Publish event
    pub async fn publish(&self, event: Event) -> Result<()> {
        self.sender.send(event)
            .map_err(|_| ReedError::EventPublishFailed)?;
        Ok(())
    }
    
    /// Subscribe to events
    pub fn subscribe(&self) -> EventSubscriber {
        let receiver = self.sender.subscribe();
        self.subscriber_count.fetch_add(1, Ordering::SeqCst);
        
        EventSubscriber {
            receiver,
            subscriber_count: self.subscriber_count.clone(),
        }
    }
    
    /// Subscribe to specific event type
    pub fn subscribe_filtered<F>(&self, filter: F) -> FilteredEventSubscriber
    where
        F: Fn(&Event) -> bool + Send + Sync + 'static,
    {
        let subscriber = self.subscribe();
        FilteredEventSubscriber {
            inner: subscriber,
            filter: Box::new(filter),
        }
    }
}

/// Event structure
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Event {
    pub id: String,
    pub event_type: String,
    pub source: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub data: serde_json::Value,
}

/// Event subscriber
pub struct EventSubscriber {
    receiver: broadcast::Receiver<Event>,
    subscriber_count: Arc<AtomicUsize>,
}

impl EventSubscriber {
    /// Receive next event
    pub async fn recv(&mut self) -> Result<Event> {
        self.receiver.recv().await
            .map_err(|e| ReedError::EventReceiveFailed(e.to_string()))
    }
}

impl Drop for EventSubscriber {
    fn drop(&mut self) {
        self.subscriber_count.fetch_sub(1, Ordering::SeqCst);
    }
}

/// Common events
pub struct Events;

impl Events {
    pub const PLUGIN_LOADED: &'static str = "plugin.loaded";
    pub const PLUGIN_UNLOADED: &'static str = "plugin.unloaded";
    pub const CONTENT_CREATED: &'static str = "content.created";
    pub const CONTENT_UPDATED: &'static str = "content.updated";
    pub const CONTENT_DELETED: &'static str = "content.deleted";
    pub const CACHE_CLEARED: &'static str = "cache.cleared";
    pub const USER_LOGIN: &'static str = "user.login";
    pub const USER_LOGOUT: &'static str = "user.logout";
}
```

## Plugin Loader

```rust
/// Plugin loader for different plugin types
pub struct PluginLoader {
    config: PluginConfig,
    validators: HashMap<String, Box<dyn PluginValidator>>,
}

impl PluginLoader {
    pub fn new(config: &PluginConfig) -> Self {
        let mut validators = HashMap::new();
        
        // Register validators for different plugin types
        validators.insert(
            "wasm".to_string(),
            Box::new(WasmPluginValidator::new()) as Box<dyn PluginValidator>,
        );
        
        validators.insert(
            "native".to_string(),
            Box::new(NativePluginValidator::new()) as Box<dyn PluginValidator>,
        );
        
        Self {
            config: config.clone(),
            validators,
        }
    }
    
    /// Load plugin metadata
    pub async fn load_metadata(&self, path: &Path) -> Result<PluginMetadata> {
        let manifest_path = path.join("plugin.toml");
        let content = tokio::fs::read_to_string(&manifest_path).await?;
        let metadata: PluginMetadata = toml::from_str(&content)?;
        
        // Validate metadata
        self.validate_metadata(&metadata)?;
        
        Ok(metadata)
    }
    
    /// Load plugin
    pub async fn load_plugin(
        &self,
        path: &Path,
        metadata: &PluginMetadata,
    ) -> Result<Box<dyn Plugin>> {
        // Determine plugin type
        let plugin_type = self.detect_plugin_type(path)?;
        
        // Validate plugin
        if let Some(validator) = self.validators.get(&plugin_type) {
            validator.validate(path, metadata).await?;
        }
        
        // Load based on type
        match plugin_type.as_str() {
            "wasm" => self.load_wasm_plugin(path, metadata).await,
            "native" => self.load_native_plugin(path, metadata).await,
            _ => Err(ReedError::UnsupportedPluginType(plugin_type)),
        }
    }
    
    /// Load WASM plugin
    async fn load_wasm_plugin(
        &self,
        path: &Path,
        metadata: &PluginMetadata,
    ) -> Result<Box<dyn Plugin>> {
        let wasm_path = path.join("plugin.wasm");
        let wasm_bytes = tokio::fs::read(&wasm_path).await?;
        
        // Create WASM runtime
        let runtime = WasmRuntime::new(&self.config.wasm)?;
        let instance = runtime.instantiate(&wasm_bytes, metadata).await?;
        
        Ok(Box::new(WasmPlugin::new(instance, metadata.clone())))
    }
}

/// Plugin validator trait
#[async_trait]
trait PluginValidator: Send + Sync {
    async fn validate(
        &self,
        path: &Path,
        metadata: &PluginMetadata,
    ) -> Result<()>;
}
```

## WASM Plugin Support

```rust
use wasmtime::{Engine, Instance, Module, Store};

/// WASM plugin runtime
pub struct WasmRuntime {
    engine: Engine,
    config: WasmConfig,
}

impl WasmRuntime {
    pub fn new(config: &WasmConfig) -> Result<Self> {
        let mut engine_config = wasmtime::Config::new();
        
        // Configure WASM engine
        engine_config
            .async_support(true)
            .consume_fuel(config.fuel_enabled)
            .epoch_interruption(true);
        
        let engine = Engine::new(&engine_config)?;
        
        Ok(Self {
            engine,
            config: config.clone(),
        })
    }
    
    /// Instantiate WASM module
    pub async fn instantiate(
        &self,
        wasm_bytes: &[u8],
        metadata: &PluginMetadata,
    ) -> Result<WasmInstance> {
        let module = Module::new(&self.engine, wasm_bytes)?;
        let mut store = Store::new(&self.engine, PluginState::new(metadata));
        
        // Set fuel limit if enabled
        if self.config.fuel_enabled {
            store.add_fuel(self.config.fuel_limit)?;
        }
        
        // Create host functions
        let host_functions = self.create_host_functions(&mut store)?;
        
        // Instantiate module
        let instance = Instance::new(&mut store, &module, &host_functions)?;
        
        Ok(WasmInstance {
            store,
            instance,
            metadata: metadata.clone(),
        })
    }
    
    /// Create host functions for plugins
    fn create_host_functions(
        &self,
        store: &mut Store<PluginState>,
    ) -> Result<Vec<wasmtime::Extern>> {
        let mut imports = Vec::new();
        
        // Log function
        let log_func = wasmtime::Func::wrap(
            store,
            |mut caller: wasmtime::Caller<'_, PluginState>, ptr: i32, len: i32| {
                let memory = caller.get_export("memory")
                    .and_then(|e| e.into_memory())
                    .ok_or_else(|| wasmtime::Error::msg("No memory export"))?;
                
                let data = memory.data(&caller);
                let message = std::str::from_utf8(&data[ptr as usize..(ptr + len) as usize])
                    .map_err(|_| wasmtime::Error::msg("Invalid UTF-8"))?;
                
                info!("[Plugin {}] {}", caller.data().metadata.id, message);
                Ok(())
            },
        );
        imports.push(log_func.into());
        
        // HTTP request function
        let http_func = wasmtime::Func::wrap(
            store,
            |caller: wasmtime::Caller<'_, PluginState>, url_ptr: i32, url_len: i32| -> i32 {
                // Implementation for HTTP requests from plugins
                // Return response handle
                0
            },
        );
        imports.push(http_func.into());
        
        Ok(imports)
    }
}

/// WASM plugin instance
pub struct WasmInstance {
    store: Store<PluginState>,
    instance: Instance,
    metadata: PluginMetadata,
}

/// Plugin state for WASM store
struct PluginState {
    metadata: PluginMetadata,
    memory_usage: usize,
}

impl PluginState {
    fn new(metadata: &PluginMetadata) -> Self {
        Self {
            metadata: metadata.clone(),
            memory_usage: 0,
        }
    }
}

/// WASM plugin implementation
struct WasmPlugin {
    instance: Arc<Mutex<WasmInstance>>,
    metadata: PluginMetadata,
}

#[async_trait]
impl Plugin for WasmPlugin {
    fn metadata(&self) -> &PluginMetadata {
        &self.metadata
    }
    
    async fn initialize(&self, context: PluginContext) -> Result<()> {
        let mut instance = self.instance.lock().await;
        
        // Call plugin initialize function
        let init_func = instance.instance
            .get_func(&mut instance.store, "initialize")
            .ok_or_else(|| ReedError::PluginError(
                "No initialize function found".to_string()
            ))?;
        
        // Serialize context
        let context_json = serde_json::to_string(&context)?;
        
        // Allocate memory for context
        let alloc_func = instance.instance
            .get_func(&mut instance.store, "allocate")
            .ok_or_else(|| ReedError::PluginError(
                "No allocate function found".to_string()
            ))?;
        
        let ptr = alloc_func
            .call(&mut instance.store, &[wasmtime::Val::I32(context_json.len() as i32)])
            .await?[0]
            .i32()
            .ok_or_else(|| ReedError::PluginError(
                "Invalid allocation result".to_string()
            ))?;
        
        // Write context to memory
        let memory = instance.instance
            .get_memory(&mut instance.store, "memory")
            .ok_or_else(|| ReedError::PluginError(
                "No memory export found".to_string()
            ))?;
        
        memory.write(&mut instance.store, ptr as usize, context_json.as_bytes())?;
        
        // Call initialize
        init_func.call(
            &mut instance.store,
            &[wasmtime::Val::I32(ptr), wasmtime::Val::I32(context_json.len() as i32)],
        ).await?;
        
        Ok(())
    }
    
    async fn shutdown(&self) -> Result<()> {
        let mut instance = self.instance.lock().await;
        
        // Call plugin shutdown function if exists
        if let Some(shutdown_func) = instance.instance
            .get_func(&mut instance.store, "shutdown") {
            shutdown_func.call(&mut instance.store, &[]).await?;
        }
        
        Ok(())
    }
}
```

## Plugin Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginConfig {
    pub enabled: bool,
    pub plugin_dir: PathBuf,
    pub auto_load: bool,
    pub fail_on_error: bool,
    pub isolation: IsolationLevel,
    pub wasm: WasmConfig,
    pub resource_limits: ResourceLimits,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum IsolationLevel {
    None,
    Process,
    Container,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WasmConfig {
    pub fuel_enabled: bool,
    pub fuel_limit: u64,
    pub memory_limit: usize,
    pub stack_size: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceLimits {
    pub max_memory: usize,
    pub max_cpu_time: u64,
    pub max_file_handles: usize,
    pub network_access: bool,
    pub filesystem_access: Vec<PathBuf>,
}

impl Default for PluginConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            plugin_dir: PathBuf::from("plugins"),
            auto_load: true,
            fail_on_error: false,
            isolation: IsolationLevel::Process,
            wasm: WasmConfig {
                fuel_enabled: true,
                fuel_limit: 1_000_000,
                memory_limit: 64 * 1024 * 1024, // 64MB
                stack_size: 1024 * 1024, // 1MB
            },
            resource_limits: ResourceLimits {
                max_memory: 256 * 1024 * 1024, // 256MB
                max_cpu_time: 30, // 30 seconds
                max_file_handles: 100,
                network_access: false,
                filesystem_access: vec![],
            },
        }
    }
}
```

## Summary

Diese Plugin Architecture bietet:
- **Plugin Types** - WASM und Native Plugin Support
- **Hook System** - Extensible Hook Points
- **Event System** - Async Event Bus für Communication
- **Isolation** - Sandboxed Plugin Execution
- **Resource Limits** - Memory und CPU Protection
- **Hot Reload** - Dynamic Plugin Loading/Unloading

Robuste und sichere Plugin Architecture.
# Plugin Loader

## Overview

Plugin Loader Implementation für ReedCMS. Dynamic Loading von WASM und Native Plugins mit Sandboxing und Resource Control.

## Plugin Discovery

```rust
use std::path::{Path, PathBuf};
use glob::glob;

/// Plugin discovery and loading system
pub struct PluginDiscovery {
    search_paths: Vec<PathBuf>,
    file_patterns: Vec<String>,
}

impl PluginDiscovery {
    pub fn new(config: &PluginConfig) -> Self {
        let mut search_paths = vec![config.plugin_dir.clone()];
        
        // Add user plugin directory
        if let Some(home) = dirs::home_dir() {
            search_paths.push(home.join(".reed/plugins"));
        }
        
        // Add system plugin directory
        #[cfg(unix)]
        search_paths.push(PathBuf::from("/usr/local/share/reed/plugins"));
        
        Self {
            search_paths,
            file_patterns: vec![
                "plugin.toml".to_string(),
                "*/plugin.toml".to_string(),
            ],
        }
    }
    
    /// Discover all available plugins
    pub async fn discover_plugins(&self) -> Result<Vec<PluginCandidate>> {
        let mut candidates = Vec::new();
        
        for search_path in &self.search_paths {
            if !search_path.exists() {
                continue;
            }
            
            for pattern in &self.file_patterns {
                let full_pattern = search_path.join(pattern);
                
                for entry in glob(full_pattern.to_str().unwrap())? {
                    if let Ok(manifest_path) = entry {
                        if let Some(candidate) = self.analyze_plugin(&manifest_path).await? {
                            candidates.push(candidate);
                        }
                    }
                }
            }
        }
        
        // Sort by priority and deduplicate
        candidates.sort_by_key(|c| (c.metadata.priority, c.metadata.name.clone()));
        candidates.dedup_by_key(|c| c.metadata.id.clone());
        
        Ok(candidates)
    }
    
    /// Analyze single plugin
    async fn analyze_plugin(
        &self,
        manifest_path: &Path,
    ) -> Result<Option<PluginCandidate>> {
        // Read manifest
        let manifest_content = tokio::fs::read_to_string(manifest_path).await?;
        let manifest: PluginManifest = toml::from_str(&manifest_content)?;
        
        // Validate manifest
        if let Err(e) = self.validate_manifest(&manifest) {
            warn!("Invalid plugin manifest at {:?}: {}", manifest_path, e);
            return Ok(None);
        }
        
        // Get plugin directory
        let plugin_dir = manifest_path.parent()
            .ok_or_else(|| ReedError::InvalidPath("No parent directory".into()))?;
        
        // Detect plugin type
        let plugin_type = self.detect_plugin_type(plugin_dir, &manifest).await?;
        
        Ok(Some(PluginCandidate {
            manifest_path: manifest_path.to_path_buf(),
            plugin_dir: plugin_dir.to_path_buf(),
            plugin_type,
            metadata: manifest.metadata,
            requirements: manifest.requirements,
        }))
    }
    
    /// Detect plugin type
    async fn detect_plugin_type(
        &self,
        plugin_dir: &Path,
        manifest: &PluginManifest,
    ) -> Result<PluginType> {
        // Check explicit type in manifest
        if let Some(ref plugin_type) = manifest.plugin_type {
            return Ok(plugin_type.clone());
        }
        
        // Auto-detect based on files
        if plugin_dir.join("plugin.wasm").exists() {
            Ok(PluginType::Wasm)
        } else if plugin_dir.join("plugin.so").exists() ||
                  plugin_dir.join("plugin.dll").exists() ||
                  plugin_dir.join("plugin.dylib").exists() {
            Ok(PluginType::Native)
        } else {
            Err(ReedError::PluginError(
                "Could not detect plugin type".to_string()
            ))
        }
    }
    
    /// Validate manifest
    fn validate_manifest(&self, manifest: &PluginManifest) -> Result<()> {
        // Check required fields
        if manifest.metadata.id.is_empty() {
            return Err(ReedError::InvalidManifest("Missing plugin ID".into()));
        }
        
        if manifest.metadata.name.is_empty() {
            return Err(ReedError::InvalidManifest("Missing plugin name".into()));
        }
        
        // Validate version
        semver::Version::parse(&manifest.metadata.version)
            .map_err(|_| ReedError::InvalidManifest("Invalid version format".into()))?;
        
        // Check API version compatibility
        let api_version = semver::Version::parse(&manifest.metadata.api_version)?;
        let current_api = semver::Version::parse(PLUGIN_API_VERSION)?;
        
        if !api_version.major.eq(&current_api.major) {
            return Err(ReedError::ApiVersionMismatch {
                expected: PLUGIN_API_VERSION.to_string(),
                actual: manifest.metadata.api_version.clone(),
            });
        }
        
        Ok(())
    }
}

/// Plugin candidate found during discovery
#[derive(Debug, Clone)]
pub struct PluginCandidate {
    pub manifest_path: PathBuf,
    pub plugin_dir: PathBuf,
    pub plugin_type: PluginType,
    pub metadata: PluginMetadata,
    pub requirements: PluginRequirements,
}

/// Plugin manifest structure
#[derive(Debug, Clone, Deserialize)]
pub struct PluginManifest {
    pub metadata: PluginMetadata,
    pub requirements: PluginRequirements,
    pub plugin_type: Option<PluginType>,
    pub entry_point: Option<String>,
    pub permissions: Option<PluginPermissions>,
    pub configuration: Option<serde_json::Value>,
}

#[derive(Debug, Clone, Deserialize)]
pub struct PluginRequirements {
    pub min_reed_version: Option<String>,
    pub dependencies: Vec<PluginDependency>,
    pub capabilities: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum PluginType {
    Wasm,
    Native,
    Script,
}

const PLUGIN_API_VERSION: &str = "1.0.0";
```

## WASM Plugin Loader

```rust
use wasmtime::*;

/// WASM plugin loader implementation
pub struct WasmPluginLoader {
    engine: Engine,
    config: WasmConfig,
    compiler: Compiler,
}

impl WasmPluginLoader {
    pub fn new(config: WasmConfig) -> Result<Self> {
        // Configure WASM engine
        let mut engine_config = Config::new();
        
        engine_config
            .async_support(true)
            .consume_fuel(config.fuel_enabled)
            .epoch_interruption(true)
            .cranelift_opt_level(OptLevel::Speed)
            .cache_config_load_default()?;
        
        // Memory limits
        engine_config.memory_guaranteed_dense_image_size(config.memory_limit);
        
        let engine = Engine::new(&engine_config)?;
        let compiler = Compiler::new();
        
        Ok(Self {
            engine,
            config,
            compiler,
        })
    }
    
    /// Load WASM plugin
    pub async fn load(
        &self,
        candidate: &PluginCandidate,
    ) -> Result<Box<dyn Plugin>> {
        let wasm_path = candidate.plugin_dir.join("plugin.wasm");
        
        // Read and validate WASM module
        let wasm_bytes = tokio::fs::read(&wasm_path).await?;
        let module = self.compile_module(&wasm_bytes).await?;
        
        // Create plugin instance
        let instance = self.instantiate_plugin(
            module,
            &candidate.metadata,
        ).await?;
        
        Ok(Box::new(WasmPlugin {
            instance: Arc::new(Mutex::new(instance)),
            metadata: candidate.metadata.clone(),
        }))
    }
    
    /// Compile WASM module
    async fn compile_module(&self, wasm_bytes: &[u8]) -> Result<Module> {
        // Validate WASM module
        wasmparser::validate(wasm_bytes)
            .map_err(|e| ReedError::InvalidWasm(e.to_string()))?;
        
        // Compile with caching
        let module = if self.config.cache_enabled {
            self.compile_with_cache(wasm_bytes).await?
        } else {
            Module::new(&self.engine, wasm_bytes)?
        };
        
        Ok(module)
    }
    
    /// Instantiate plugin
    async fn instantiate_plugin(
        &self,
        module: Module,
        metadata: &PluginMetadata,
    ) -> Result<WasmPluginInstance> {
        // Create store with limits
        let mut store = Store::new(
            &self.engine,
            WasmPluginState {
                metadata: metadata.clone(),
                memory_usage: 0,
                fuel_consumed: 0,
            },
        );
        
        // Set resource limits
        store.limiter(|state| &mut state as &mut dyn ResourceLimiter);
        
        // Set initial fuel
        if self.config.fuel_enabled {
            store.add_fuel(self.config.fuel_limit)?;
        }
        
        // Create linker with host functions
        let linker = self.create_linker(&mut store)?;
        
        // Instantiate module
        let instance = linker.instantiate_async(&mut store, &module).await?;
        
        // Initialize plugin
        self.initialize_plugin(&mut store, &instance).await?;
        
        Ok(WasmPluginInstance {
            store,
            instance,
            module,
        })
    }
    
    /// Create linker with host functions
    fn create_linker(
        &self,
        store: &mut Store<WasmPluginState>,
    ) -> Result<Linker<WasmPluginState>> {
        let mut linker = Linker::new(&self.engine);
        
        // Memory allocation
        linker.func_wrap(
            "env",
            "allocate",
            |mut caller: Caller<'_, WasmPluginState>, size: i32| -> i32 {
                // Allocate memory in WASM linear memory
                let memory = caller.get_export("memory")
                    .and_then(|e| e.into_memory())
                    .ok_or_else(|| Trap::new("No memory export"))?;
                
                // Simple bump allocator
                let state = caller.data_mut();
                let ptr = state.memory_usage as i32;
                state.memory_usage += size as usize;
                
                // Check memory limit
                if state.memory_usage > 64 * 1024 * 1024 { // 64MB limit
                    return Err(Trap::new("Memory limit exceeded"));
                }
                
                Ok(ptr)
            },
        )?;
        
        // Logging
        linker.func_wrap(
            "env",
            "log",
            |mut caller: Caller<'_, WasmPluginState>, level: i32, ptr: i32, len: i32| {
                let memory = caller.get_export("memory")
                    .and_then(|e| e.into_memory())
                    .ok_or_else(|| Trap::new("No memory export"))?;
                
                let data = memory.data(&caller);
                let message = std::str::from_utf8(
                    &data[ptr as usize..(ptr + len) as usize]
                ).unwrap_or("<invalid utf8>");
                
                let plugin_id = &caller.data().metadata.id;
                
                match level {
                    0 => debug!("[Plugin {}] {}", plugin_id, message),
                    1 => info!("[Plugin {}] {}", plugin_id, message),
                    2 => warn!("[Plugin {}] {}", plugin_id, message),
                    3 => error!("[Plugin {}] {}", plugin_id, message),
                    _ => info!("[Plugin {}] {}", plugin_id, message),
                }
                
                Ok(())
            },
        )?;
        
        // API calls
        self.link_api_functions(&mut linker)?;
        
        Ok(linker)
    }
    
    /// Link API functions
    fn link_api_functions(
        &self,
        linker: &mut Linker<WasmPluginState>,
    ) -> Result<()> {
        // Content API
        linker.func_wrap(
            "reed",
            "content_create",
            |caller: Caller<'_, WasmPluginState>, type_ptr: i32, type_len: i32, data_ptr: i32, data_len: i32| -> i32 {
                // Implementation
                0
            },
        )?;
        
        // Storage API
        linker.func_wrap(
            "reed",
            "storage_put",
            |caller: Caller<'_, WasmPluginState>, key_ptr: i32, key_len: i32, value_ptr: i32, value_len: i32| -> i32 {
                // Implementation
                0
            },
        )?;
        
        // Event API
        linker.func_wrap(
            "reed",
            "event_emit",
            |caller: Caller<'_, WasmPluginState>, event_ptr: i32, event_len: i32| -> i32 {
                // Implementation
                0
            },
        )?;
        
        Ok(())
    }
}

/// WASM plugin state
struct WasmPluginState {
    metadata: PluginMetadata,
    memory_usage: usize,
    fuel_consumed: u64,
}

impl ResourceLimiter for WasmPluginState {
    fn memory_growing(
        &mut self,
        current: usize,
        desired: usize,
        _maximum: Option<usize>,
    ) -> Result<bool> {
        // Check memory limit
        Ok(desired <= 64 * 1024 * 1024) // 64MB limit
    }
    
    fn table_growing(
        &mut self,
        current: u32,
        desired: u32,
        _maximum: Option<u32>,
    ) -> Result<bool> {
        // Allow reasonable table growth
        Ok(desired <= 10000)
    }
}

/// WASM plugin instance
struct WasmPluginInstance {
    store: Store<WasmPluginState>,
    instance: Instance,
    module: Module,
}
```

## Native Plugin Loader

```rust
use libloading::{Library, Symbol};

/// Native plugin loader implementation
pub struct NativePluginLoader {
    config: NativePluginConfig,
    sandbox: PluginSandbox,
}

impl NativePluginLoader {
    pub fn new(config: NativePluginConfig) -> Result<Self> {
        let sandbox = PluginSandbox::new(&config.sandbox)?;
        
        Ok(Self {
            config,
            sandbox,
        })
    }
    
    /// Load native plugin
    pub async fn load(
        &self,
        candidate: &PluginCandidate,
    ) -> Result<Box<dyn Plugin>> {
        // Determine library file
        let lib_file = self.find_library_file(&candidate.plugin_dir)?;
        
        // Validate library
        self.validate_library(&lib_file).await?;
        
        // Load in sandbox if configured
        let plugin = if self.config.sandbox.enabled {
            self.load_sandboxed(candidate, &lib_file).await?
        } else {
            self.load_direct(candidate, &lib_file).await?
        };
        
        Ok(plugin)
    }
    
    /// Find library file for current platform
    fn find_library_file(&self, plugin_dir: &Path) -> Result<PathBuf> {
        let lib_name = "plugin";
        
        #[cfg(target_os = "windows")]
        let lib_file = plugin_dir.join(format!("{}.dll", lib_name));
        
        #[cfg(target_os = "macos")]
        let lib_file = plugin_dir.join(format!("lib{}.dylib", lib_name));
        
        #[cfg(target_os = "linux")]
        let lib_file = plugin_dir.join(format!("lib{}.so", lib_name));
        
        if !lib_file.exists() {
            return Err(ReedError::PluginNotFound(
                format!("Library file not found: {:?}", lib_file)
            ));
        }
        
        Ok(lib_file)
    }
    
    /// Load plugin directly (no sandbox)
    async fn load_direct(
        &self,
        candidate: &PluginCandidate,
        lib_file: &Path,
    ) -> Result<Box<dyn Plugin>> {
        unsafe {
            // Load library
            let library = Library::new(lib_file)?;
            
            // Get plugin create function
            let create_fn: Symbol<unsafe extern "C" fn() -> *mut dyn ReedPlugin> =
                library.get(b"_reed_plugin_create")?;
            
            // Check API version
            let version_fn: Symbol<unsafe extern "C" fn() -> *const c_char> =
                library.get(b"_reed_plugin_api_version")?;
            
            let api_version = CStr::from_ptr(version_fn())
                .to_string_lossy()
                .to_string();
            
            if api_version != PLUGIN_API_VERSION {
                return Err(ReedError::ApiVersionMismatch {
                    expected: PLUGIN_API_VERSION.to_string(),
                    actual: api_version,
                });
            }
            
            // Create plugin instance
            let plugin_ptr = create_fn();
            let plugin = Box::from_raw(plugin_ptr);
            
            Ok(Box::new(NativePlugin {
                plugin,
                library: Arc::new(library),
                metadata: candidate.metadata.clone(),
            }))
        }
    }
    
    /// Validate library safety
    async fn validate_library(&self, lib_file: &Path) -> Result<()> {
        // Check file permissions
        let metadata = tokio::fs::metadata(lib_file).await?;
        
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            let mode = metadata.permissions().mode();
            
            // Check for suspicious permissions
            if mode & 0o002 != 0 {
                return Err(ReedError::SecurityError(
                    "Plugin library is world-writable".to_string()
                ));
            }
        }
        
        // Check signature if enabled
        if self.config.require_signature {
            self.verify_signature(lib_file).await?;
        }
        
        Ok(())
    }
}

/// Native plugin wrapper
struct NativePlugin {
    plugin: Box<dyn ReedPlugin>,
    library: Arc<Library>,
    metadata: PluginMetadata,
}

#[async_trait]
impl Plugin for NativePlugin {
    fn metadata(&self) -> &PluginMetadata {
        &self.metadata
    }
    
    async fn initialize(&self, context: PluginContext) -> Result<()> {
        // Create API instance for plugin
        let api = PluginApi::new(self.metadata.id.clone(), context);
        
        // Initialize plugin
        self.plugin.init(api).await
            .map_err(|e| ReedError::PluginError(e.to_string()))
    }
    
    async fn shutdown(&self) -> Result<()> {
        self.plugin.deactivate().await
            .map_err(|e| ReedError::PluginError(e.to_string()))
    }
}
```

## Plugin Sandbox

```rust
/// Plugin sandbox for isolation
pub struct PluginSandbox {
    config: SandboxConfig,
}

impl PluginSandbox {
    pub fn new(config: &SandboxConfig) -> Result<Self> {
        Ok(Self {
            config: config.clone(),
        })
    }
    
    /// Create sandboxed environment
    pub async fn create_sandbox(
        &self,
        plugin_id: &str,
    ) -> Result<SandboxEnvironment> {
        match self.config.mode {
            SandboxMode::None => {
                Ok(SandboxEnvironment::None)
            }
            SandboxMode::Process => {
                self.create_process_sandbox(plugin_id).await
            }
            SandboxMode::Container => {
                self.create_container_sandbox(plugin_id).await
            }
        }
    }
    
    /// Create process-based sandbox
    async fn create_process_sandbox(
        &self,
        plugin_id: &str,
    ) -> Result<SandboxEnvironment> {
        use tokio::process::Command;
        
        let child = Command::new(&self.config.sandbox_binary)
            .arg("--plugin-id")
            .arg(plugin_id)
            .arg("--socket")
            .arg(&self.config.socket_path)
            .stdin(std::process::Stdio::null())
            .stdout(std::process::Stdio::piped())
            .stderr(std::process::Stdio::piped())
            .spawn()?;
        
        Ok(SandboxEnvironment::Process {
            child,
            socket_path: self.config.socket_path.clone(),
        })
    }
}

#[derive(Debug)]
enum SandboxEnvironment {
    None,
    Process {
        child: tokio::process::Child,
        socket_path: PathBuf,
    },
    Container {
        container_id: String,
        socket_path: PathBuf,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SandboxConfig {
    pub enabled: bool,
    pub mode: SandboxMode,
    pub sandbox_binary: PathBuf,
    pub socket_path: PathBuf,
    pub resource_limits: ResourceLimits,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SandboxMode {
    None,
    Process,
    Container,
}
```

## Plugin Validation

```rust
/// Plugin validation system
pub struct PluginValidator {
    rules: Vec<Box<dyn ValidationRule>>,
}

impl PluginValidator {
    pub fn new() -> Self {
        let mut rules: Vec<Box<dyn ValidationRule>> = vec![
            Box::new(ApiVersionRule),
            Box::new(PermissionRule),
            Box::new(DependencyRule),
            Box::new(SecurityRule),
        ];
        
        Self { rules }
    }
    
    /// Validate plugin
    pub async fn validate(
        &self,
        candidate: &PluginCandidate,
        context: &ValidationContext,
    ) -> Result<ValidationResult> {
        let mut result = ValidationResult {
            valid: true,
            errors: Vec::new(),
            warnings: Vec::new(),
        };
        
        for rule in &self.rules {
            rule.validate(candidate, context, &mut result).await?;
        }
        
        Ok(result)
    }
}

/// Validation rule trait
#[async_trait]
trait ValidationRule: Send + Sync {
    async fn validate(
        &self,
        candidate: &PluginCandidate,
        context: &ValidationContext,
        result: &mut ValidationResult,
    ) -> Result<()>;
}

/// API version validation
struct ApiVersionRule;

#[async_trait]
impl ValidationRule for ApiVersionRule {
    async fn validate(
        &self,
        candidate: &PluginCandidate,
        _context: &ValidationContext,
        result: &mut ValidationResult,
    ) -> Result<()> {
        let plugin_api = semver::Version::parse(&candidate.metadata.api_version)?;
        let current_api = semver::Version::parse(PLUGIN_API_VERSION)?;
        
        if plugin_api.major != current_api.major {
            result.valid = false;
            result.errors.push(ValidationError {
                code: "api_version_mismatch".to_string(),
                message: format!(
                    "Plugin requires API version {}, but system provides {}",
                    candidate.metadata.api_version,
                    PLUGIN_API_VERSION
                ),
            });
        } else if plugin_api.minor > current_api.minor {
            result.warnings.push(ValidationWarning {
                code: "api_version_newer".to_string(),
                message: "Plugin uses newer API features that may not be available".to_string(),
            });
        }
        
        Ok(())
    }
}

#[derive(Debug)]
pub struct ValidationResult {
    pub valid: bool,
    pub errors: Vec<ValidationError>,
    pub warnings: Vec<ValidationWarning>,
}

#[derive(Debug)]
pub struct ValidationError {
    pub code: String,
    pub message: String,
}

#[derive(Debug)]
pub struct ValidationWarning {
    pub code: String,
    pub message: String,
}

pub struct ValidationContext {
    pub loaded_plugins: Vec<String>,
    pub system_capabilities: Vec<String>,
}
```

## Plugin Cache

```rust
/// Plugin compilation cache
pub struct PluginCache {
    cache_dir: PathBuf,
    index: Arc<RwLock<CacheIndex>>,
}

impl PluginCache {
    pub fn new(cache_dir: PathBuf) -> Result<Self> {
        std::fs::create_dir_all(&cache_dir)?;
        
        let index_path = cache_dir.join("index.json");
        let index = if index_path.exists() {
            let data = std::fs::read_to_string(&index_path)?;
            serde_json::from_str(&data)?
        } else {
            CacheIndex::new()
        };
        
        Ok(Self {
            cache_dir,
            index: Arc::new(RwLock::new(index)),
        })
    }
    
    /// Get cached plugin
    pub async fn get(
        &self,
        plugin_id: &str,
        version: &str,
    ) -> Result<Option<CachedPlugin>> {
        let index = self.index.read().await;
        
        if let Some(entry) = index.get_entry(plugin_id, version) {
            let cache_path = self.cache_dir.join(&entry.file_name);
            
            if cache_path.exists() {
                let data = tokio::fs::read(&cache_path).await?;
                return Ok(Some(CachedPlugin {
                    data,
                    metadata: entry.metadata.clone(),
                }));
            }
        }
        
        Ok(None)
    }
    
    /// Store compiled plugin
    pub async fn store(
        &self,
        plugin_id: &str,
        version: &str,
        data: Vec<u8>,
        metadata: CacheMetadata,
    ) -> Result<()> {
        let file_name = format!("{}-{}.cache", plugin_id, version);
        let cache_path = self.cache_dir.join(&file_name);
        
        // Write cache file
        tokio::fs::write(&cache_path, &data).await?;
        
        // Update index
        let mut index = self.index.write().await;
        index.add_entry(CacheEntry {
            plugin_id: plugin_id.to_string(),
            version: version.to_string(),
            file_name,
            metadata,
            created_at: chrono::Utc::now(),
        });
        
        // Save index
        self.save_index(&index).await?;
        
        Ok(())
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct CacheIndex {
    entries: HashMap<String, CacheEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct CacheEntry {
    plugin_id: String,
    version: String,
    file_name: String,
    metadata: CacheMetadata,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CacheMetadata {
    pub compiled_size: usize,
    pub compilation_time: u64,
    pub compiler_version: String,
}

#[derive(Debug)]
pub struct CachedPlugin {
    pub data: Vec<u8>,
    pub metadata: CacheMetadata,
}
```

## Summary

Dieser Plugin Loader bietet:
- **Plugin Discovery** - Automatic Plugin Detection
- **WASM Support** - Sandboxed WASM Execution
- **Native Support** - Dynamic Library Loading
- **Validation** - Comprehensive Plugin Validation
- **Sandboxing** - Process und Container Isolation
- **Caching** - Compiled Plugin Cache

Robuster und sicherer Plugin Loading Mechanism.
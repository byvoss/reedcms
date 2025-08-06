# EPC (Explicit Path Chain) Resolver

## Overview

EPC Resolver implementiert das File Resolution System für ReedCMS. Explizite Pfade durch Theme-Hierarchien für maximale Flexibilität.

## EPC Resolver Core

```rust
use std::path::{Path, PathBuf};
use std::collections::HashMap;

/// Resolves file paths through theme chain
pub struct EpcResolver {
    base_path: PathBuf,
    theme_manager: Arc<ThemeManager>,
    cache: Arc<RwLock<HashMap<String, ResolvedPath>>>,
}

impl EpcResolver {
    pub fn new(base_path: PathBuf, theme_manager: Arc<ThemeManager>) -> Self {
        Self {
            base_path,
            theme_manager,
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Resolve template file through theme chain
    pub async fn resolve_template(
        &self,
        template_name: &str,
        context: &RequestContext,
    ) -> Result<PathBuf> {
        self.resolve_file(
            template_name,
            FileType::Template,
            &context.theme,
        ).await
    }
    
    /// Resolve any file type through theme chain
    pub async fn resolve_file(
        &self,
        file_name: &str,
        file_type: FileType,
        theme: &Theme,
    ) -> Result<PathBuf> {
        // Build cache key
        let cache_key = format!("{:?}:{}:{}", file_type, theme.full_name(), file_name);
        
        // Check cache
        {
            let cache = self.cache.read().await;
            if let Some(resolved) = cache.get(&cache_key) {
                if resolved.is_valid() {
                    return Ok(resolved.path.clone());
                }
            }
        }
        
        // Build theme chain
        let theme_chain = self.theme_manager.build_theme_chain(theme).await?;
        
        // Search through chain
        for theme_name in &theme_chain {
            let path = self.build_file_path(theme_name, file_type, file_name);
            
            if path.exists() {
                // Cache result
                let resolved = ResolvedPath {
                    path: path.clone(),
                    theme: theme_name.clone(),
                    resolved_at: Instant::now(),
                };
                
                let mut cache = self.cache.write().await;
                cache.insert(cache_key, resolved);
                
                tracing::debug!(
                    "Resolved {} '{}' in theme '{}'",
                    file_type.as_str(),
                    file_name,
                    theme_name
                );
                
                return Ok(path);
            }
        }
        
        // Not found in any theme
        Err(ReedError::FileNotFound(FileNotFoundError {
            file_name: file_name.to_string(),
            file_type,
            searched_themes: theme_chain,
        }))
    }
    
    /// Build full file path
    fn build_file_path(
        &self,
        theme_name: &str,
        file_type: FileType,
        file_name: &str,
    ) -> PathBuf {
        let mut path = self.base_path.clone();
        path.push("themes");
        path.push(theme_name);
        path.push(file_type.directory());
        path.push(file_name);
        
        // Add extension if missing
        if path.extension().is_none() {
            let ext = file_type.default_extension();
            path.set_extension(ext);
        }
        
        path
    }
}

#[derive(Debug, Clone)]
struct ResolvedPath {
    path: PathBuf,
    theme: String,
    resolved_at: Instant,
}

impl ResolvedPath {
    fn is_valid(&self) -> bool {
        // Cache for 5 seconds in development
        #[cfg(debug_assertions)]
        {
            self.resolved_at.elapsed() < Duration::from_secs(5)
        }
        
        // Cache for 1 minute in production
        #[cfg(not(debug_assertions))]
        {
            self.resolved_at.elapsed() < Duration::from_secs(60)
        }
    }
}
```

## File Types

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum FileType {
    Template,
    Partial,
    Layout,
    Asset,
    Style,
    Script,
    Image,
    Font,
    Config,
}

impl FileType {
    /// Get directory name for file type
    pub fn directory(&self) -> &'static str {
        match self {
            FileType::Template => "templates",
            FileType::Partial => "partials",
            FileType::Layout => "layouts",
            FileType::Asset => "assets",
            FileType::Style => "assets/css",
            FileType::Script => "assets/js",
            FileType::Image => "assets/images",
            FileType::Font => "assets/fonts",
            FileType::Config => "config",
        }
    }
    
    /// Get default file extension
    pub fn default_extension(&self) -> &'static str {
        match self {
            FileType::Template => "tera",
            FileType::Partial => "tera",
            FileType::Layout => "tera",
            FileType::Style => "css",
            FileType::Script => "js",
            FileType::Config => "toml",
            _ => "",
        }
    }
    
    pub fn as_str(&self) -> &'static str {
        match self {
            FileType::Template => "template",
            FileType::Partial => "partial",
            FileType::Layout => "layout",
            FileType::Asset => "asset",
            FileType::Style => "style",
            FileType::Script => "script",
            FileType::Image => "image",
            FileType::Font => "font",
            FileType::Config => "config",
        }
    }
}
```

## Multi-file Resolution

```rust
impl EpcResolver {
    /// Resolve multiple files (for bundling)
    pub async fn resolve_files(
        &self,
        pattern: &str,
        file_type: FileType,
        theme: &Theme,
    ) -> Result<Vec<PathBuf>> {
        let theme_chain = self.theme_manager.build_theme_chain(theme).await?;
        let mut resolved_files = Vec::new();
        let mut seen = HashSet::new();
        
        // Search through theme chain in reverse order (base first)
        for theme_name in theme_chain.iter().rev() {
            let theme_path = self.base_path
                .join("themes")
                .join(theme_name)
                .join(file_type.directory());
            
            if !theme_path.exists() {
                continue;
            }
            
            // Find matching files
            let pattern_path = theme_path.join(pattern);
            let matches = glob::glob(&pattern_path.to_string_lossy())?;
            
            for entry in matches {
                if let Ok(path) = entry {
                    let relative = path.strip_prefix(&theme_path)
                        .unwrap_or(&path)
                        .to_path_buf();
                    
                    // Only add if not seen (allows overrides)
                    if seen.insert(relative.clone()) {
                        resolved_files.push(path);
                    }
                }
            }
        }
        
        Ok(resolved_files)
    }
    
    /// Get all available templates
    pub async fn list_templates(&self, theme: &Theme) -> Result<Vec<TemplateInfo>> {
        let files = self.resolve_files("**/*.tera", FileType::Template, theme).await?;
        let mut templates = Vec::new();
        
        for path in files {
            if let Some(name) = self.extract_template_name(&path) {
                templates.push(TemplateInfo {
                    name,
                    path,
                    theme: self.identify_theme(&path),
                });
            }
        }
        
        templates.sort_by(|a, b| a.name.cmp(&b.name));
        Ok(templates)
    }
    
    fn extract_template_name(&self, path: &Path) -> Option<String> {
        path.strip_prefix(&self.base_path)
            .ok()?
            .strip_prefix("themes")?
            .components()
            .skip(2) // Skip theme name and "templates"
            .map(|c| c.as_os_str().to_string_lossy())
            .collect::<Vec<_>>()
            .join("/")
            .strip_suffix(".tera")
            .map(|s| s.to_string())
    }
    
    fn identify_theme(&self, path: &Path) -> String {
        path.strip_prefix(&self.base_path)
            .ok()
            .and_then(|p| p.strip_prefix("themes").ok())
            .and_then(|p| p.components().next())
            .map(|c| c.as_os_str().to_string_lossy().to_string())
            .unwrap_or_else(|| "unknown".to_string())
    }
}

#[derive(Debug, Clone)]
pub struct TemplateInfo {
    pub name: String,
    pub path: PathBuf,
    pub theme: String,
}
```

## Asset Resolution

```rust
impl EpcResolver {
    /// Resolve asset with fallback chain
    pub async fn resolve_asset(
        &self,
        asset_path: &str,
        theme: &Theme,
    ) -> Result<AssetResolution> {
        // Try exact match first
        if let Ok(path) = self.resolve_file(asset_path, FileType::Asset, theme).await {
            return Ok(AssetResolution {
                path,
                url: self.build_asset_url(asset_path, theme),
                theme: theme.name.clone(),
            });
        }
        
        // Try common asset directories
        for file_type in &[FileType::Style, FileType::Script, FileType::Image, FileType::Font] {
            let typed_path = asset_path.trim_start_matches(&format!("{}/", file_type.directory()));
            
            if let Ok(path) = self.resolve_file(typed_path, *file_type, theme).await {
                return Ok(AssetResolution {
                    path,
                    url: self.build_asset_url(asset_path, theme),
                    theme: theme.name.clone(),
                });
            }
        }
        
        Err(ReedError::AssetNotFound(asset_path.to_string()))
    }
    
    /// Build public URL for asset
    fn build_asset_url(&self, asset_path: &str, theme: &Theme) -> String {
        format!("/themes/{}/assets/{}", theme.name, asset_path)
    }
    
    /// Get asset with content hash for cache busting
    pub async fn resolve_asset_with_hash(
        &self,
        asset_path: &str,
        theme: &Theme,
    ) -> Result<HashedAsset> {
        let resolution = self.resolve_asset(asset_path, theme).await?;
        
        // Calculate content hash
        let content = tokio::fs::read(&resolution.path).await?;
        let hash = self.calculate_hash(&content);
        
        Ok(HashedAsset {
            path: resolution.path,
            url: format!("{}?v={}", resolution.url, &hash[..8]),
            hash,
            size: content.len(),
        })
    }
    
    fn calculate_hash(&self, content: &[u8]) -> String {
        use sha2::{Sha256, Digest};
        let mut hasher = Sha256::new();
        hasher.update(content);
        format!("{:x}", hasher.finalize())
    }
}

#[derive(Debug)]
pub struct AssetResolution {
    pub path: PathBuf,
    pub url: String,
    pub theme: String,
}

#[derive(Debug)]
pub struct HashedAsset {
    pub path: PathBuf,
    pub url: String,
    pub hash: String,
    pub size: usize,
}
```

## Override Detection

```rust
impl EpcResolver {
    /// Get all overrides for a file
    pub async fn get_overrides(
        &self,
        file_name: &str,
        file_type: FileType,
        theme: &Theme,
    ) -> Result<Vec<Override>> {
        let theme_chain = self.theme_manager.build_theme_chain(theme).await?;
        let mut overrides = Vec::new();
        
        for (index, theme_name) in theme_chain.iter().enumerate() {
            let path = self.build_file_path(theme_name, file_type, file_name);
            
            if path.exists() {
                let metadata = tokio::fs::metadata(&path).await?;
                
                overrides.push(Override {
                    theme: theme_name.clone(),
                    path,
                    priority: index,
                    modified: metadata.modified().ok(),
                    size: metadata.len(),
                });
            }
        }
        
        Ok(overrides)
    }
    
    /// Check if file is overridden in child theme
    pub async fn is_overridden(
        &self,
        file_name: &str,
        file_type: FileType,
        theme: &Theme,
    ) -> Result<bool> {
        let overrides = self.get_overrides(file_name, file_type, theme).await?;
        Ok(overrides.len() > 1)
    }
}

#[derive(Debug)]
pub struct Override {
    pub theme: String,
    pub path: PathBuf,
    pub priority: usize,
    pub modified: Option<std::time::SystemTime>,
    pub size: u64,
}
```

## Resolution Debugging

```rust
/// Debug information for EPC resolution
pub struct EpcDebugger {
    resolutions: Arc<RwLock<Vec<ResolutionLog>>>,
}

#[derive(Debug, Clone)]
pub struct ResolutionLog {
    pub timestamp: Instant,
    pub file_name: String,
    pub file_type: FileType,
    pub theme_chain: Vec<String>,
    pub resolved_in: Option<String>,
    pub duration_us: u64,
}

impl EpcDebugger {
    pub fn new() -> Self {
        Self {
            resolutions: Arc::new(RwLock::new(Vec::new())),
        }
    }
    
    /// Log resolution attempt
    pub async fn log_resolution(
        &self,
        file_name: &str,
        file_type: FileType,
        theme_chain: Vec<String>,
        resolved_in: Option<String>,
        duration: Duration,
    ) {
        let log = ResolutionLog {
            timestamp: Instant::now(),
            file_name: file_name.to_string(),
            file_type,
            theme_chain,
            resolved_in,
            duration_us: duration.as_micros() as u64,
        };
        
        let mut logs = self.resolutions.write().await;
        logs.push(log);
        
        // Keep only last 1000 entries
        if logs.len() > 1000 {
            logs.drain(0..100);
        }
    }
    
    /// Get resolution statistics
    pub async fn get_stats(&self) -> ResolutionStats {
        let logs = self.resolutions.read().await;
        
        let total = logs.len();
        let successful = logs.iter().filter(|l| l.resolved_in.is_some()).count();
        let avg_duration = if total > 0 {
            logs.iter().map(|l| l.duration_us).sum::<u64>() / total as u64
        } else {
            0
        };
        
        let mut by_theme = HashMap::new();
        for log in logs.iter() {
            if let Some(theme) = &log.resolved_in {
                *by_theme.entry(theme.clone()).or_insert(0) += 1;
            }
        }
        
        ResolutionStats {
            total_resolutions: total,
            successful_resolutions: successful,
            avg_duration_us: avg_duration,
            resolutions_by_theme: by_theme,
        }
    }
}

#[derive(Debug, Serialize)]
pub struct ResolutionStats {
    pub total_resolutions: usize,
    pub successful_resolutions: usize,
    pub avg_duration_us: u64,
    pub resolutions_by_theme: HashMap<String, usize>,
}
```

## Summary

Dieses EPC Resolution System bietet:
- **Explicit Path Chain** - Klare, vorhersehbare File Resolution
- **Theme Override System** - Child Themes können Parent Files überschreiben
- **Performance Caching** - Schnelle wiederholte Resolutions
- **Asset Management** - Hash-based Cache Busting
- **Debug Support** - Detaillierte Resolution Logs

EPC macht Theme Development transparent und powerful.
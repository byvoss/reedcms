# EPC Theme Chain Building

## Overview

Theme Chain Building für das EPC System. Aufbau der Theme-Hierarchie mit Context-aware Resolution für maximale Flexibilität.

## Theme Manager Core

```rust
use std::collections::HashMap;
use std::sync::Arc;

/// Manages themes and builds theme chains
pub struct ThemeManager {
    themes: Arc<RwLock<HashMap<String, Theme>>>,
    active_chains: Arc<RwLock<HashMap<String, Vec<String>>>>,
    config: Arc<Config>,
}

impl ThemeManager {
    pub fn new(config: Arc<Config>) -> Self {
        Self {
            themes: Arc::new(RwLock::new(HashMap::new())),
            active_chains: Arc::new(RwLock::new(HashMap::new())),
            config,
        }
    }
    
    /// Initialize theme manager
    pub async fn initialize(&self) -> Result<()> {
        // Load theme configurations
        self.load_themes().await?;
        
        // Validate theme dependencies
        self.validate_dependencies().await?;
        
        // Build initial chains
        self.rebuild_all_chains().await?;
        
        Ok(())
    }
    
    /// Build theme chain for given context
    pub async fn build_theme_chain(&self, theme: &Theme) -> Result<Vec<String>> {
        // Check cache
        let cache_key = theme.full_name();
        {
            let chains = self.active_chains.read().await;
            if let Some(chain) = chains.get(&cache_key) {
                return Ok(chain.clone());
            }
        }
        
        // Build new chain
        let mut chain = Vec::new();
        let mut current = Some(theme.clone());
        let mut visited = HashSet::new();
        
        while let Some(t) = current {
            let name = t.name.clone();
            
            // Prevent cycles
            if !visited.insert(name.clone()) {
                return Err(ReedError::ThemeCycle(name));
            }
            
            chain.push(name.clone());
            
            // Get parent theme
            current = if let Some(parent_name) = &t.parent {
                let themes = self.themes.read().await;
                themes.get(parent_name).cloned()
            } else {
                None
            };
        }
        
        // Always end with base theme
        if !chain.contains(&"base".to_string()) {
            chain.push("base".to_string());
        }
        
        // Cache result
        {
            let mut chains = self.active_chains.write().await;
            chains.insert(cache_key, chain.clone());
        }
        
        Ok(chain)
    }
}
```

## Theme Structure

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Theme {
    pub name: String,
    pub parent: Option<String>,
    pub context: ThemeContext,
    pub metadata: ThemeMetadata,
    pub active: bool,
}

impl Theme {
    /// Get full theme name including context
    pub fn full_name(&self) -> String {
        match &self.context {
            ThemeContext::Default => self.name.clone(),
            ThemeContext::Location(loc) => format!("{}.location.{}", self.name, loc),
            ThemeContext::Season(season) => format!("{}.season.{}", self.name, season),
            ThemeContext::Event(event) => format!("{}.event.{}", self.name, event),
            ThemeContext::Custom(ctx) => format!("{}.{}", self.name, ctx),
        }
    }
    
    /// Check if theme matches context
    pub fn matches_context(&self, request_context: &RequestContext) -> bool {
        match &self.context {
            ThemeContext::Default => true,
            ThemeContext::Location(loc) => {
                request_context.location.as_ref()
                    .map(|l| l == loc)
                    .unwrap_or(false)
            }
            ThemeContext::Season(season) => {
                self.is_current_season(season)
            }
            ThemeContext::Event(event) => {
                request_context.active_events.contains(event)
            }
            ThemeContext::Custom(ctx) => {
                request_context.custom_contexts.contains(ctx)
            }
        }
    }
    
    fn is_current_season(&self, season: &str) -> bool {
        use chrono::{Datelike, Utc};
        let now = Utc::now();
        let month = now.month();
        
        match season {
            "spring" => month >= 3 && month <= 5,
            "summer" => month >= 6 && month <= 8,
            "fall" | "autumn" => month >= 9 && month <= 11,
            "winter" => month == 12 || month <= 2,
            _ => false,
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ThemeContext {
    Default,
    Location(String),
    Season(String),
    Event(String),
    Custom(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThemeMetadata {
    pub version: String,
    pub author: String,
    pub description: String,
    pub preview_image: Option<String>,
    pub requirements: ThemeRequirements,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThemeRequirements {
    pub min_version: Option<String>,
    pub required_plugins: Vec<String>,
    pub required_features: Vec<String>,
}
```

## Context-Aware Resolution

```rust
impl ThemeManager {
    /// Select best theme for request context
    pub async fn select_theme(&self, context: &RequestContext) -> Result<Theme> {
        let themes = self.themes.read().await;
        
        // Build candidate list
        let mut candidates: Vec<(Theme, u32)> = themes.values()
            .filter(|t| t.active && t.matches_context(context))
            .map(|t| (t.clone(), self.calculate_theme_score(t, context)))
            .collect();
        
        // Sort by score (highest first)
        candidates.sort_by(|a, b| b.1.cmp(&a.1));
        
        // Return best match
        candidates.into_iter()
            .next()
            .map(|(theme, _)| theme)
            .ok_or_else(|| ReedError::NoThemeAvailable)
    }
    
    /// Calculate theme matching score
    fn calculate_theme_score(&self, theme: &Theme, context: &RequestContext) -> u32 {
        let mut score = 0;
        
        // Base score for active theme
        score += 100;
        
        // Context specificity bonus
        match &theme.context {
            ThemeContext::Default => score += 10,
            ThemeContext::Location(_) => score += 50,
            ThemeContext::Season(_) => score += 30,
            ThemeContext::Event(_) => score += 40,
            ThemeContext::Custom(_) => score += 20,
        }
        
        // User preference bonus
        if let Some(pref) = &context.user_theme_preference {
            if theme.name == *pref {
                score += 100;
            }
        }
        
        // Feature compatibility bonus
        let available_features: HashSet<_> = context.features.iter().collect();
        let required_features: HashSet<_> = theme.metadata.requirements.required_features.iter().collect();
        
        if required_features.is_subset(&available_features) {
            score += 20;
        }
        
        score
    }
}
```

## Theme Loading

```rust
impl ThemeManager {
    /// Load all theme configurations
    async fn load_themes(&self) -> Result<()> {
        let themes_dir = self.config.themes_path.clone();
        let mut themes = self.themes.write().await;
        
        themes.clear();
        
        // Always add base theme
        themes.insert("base".to_string(), Theme {
            name: "base".to_string(),
            parent: None,
            context: ThemeContext::Default,
            metadata: ThemeMetadata {
                version: "1.0.0".to_string(),
                author: "ReedCMS".to_string(),
                description: "Base theme with minimal styling".to_string(),
                preview_image: None,
                requirements: ThemeRequirements {
                    min_version: None,
                    required_plugins: vec![],
                    required_features: vec![],
                },
            },
            active: true,
        });
        
        // Load theme directories
        let entries = tokio::fs::read_dir(&themes_dir).await?;
        let mut entries = tokio_stream::wrappers::ReadDirStream::new(entries);
        
        while let Some(entry) = entries.next().await {
            let entry = entry?;
            let path = entry.path();
            
            if path.is_dir() {
                if let Some(theme_name) = path.file_name().and_then(|n| n.to_str()) {
                    if theme_name != "base" {
                        if let Ok(theme) = self.load_theme_config(&path).await {
                            themes.insert(theme_name.to_string(), theme);
                        }
                    }
                }
            }
        }
        
        tracing::info!("Loaded {} themes", themes.len());
        Ok(())
    }
    
    /// Load single theme configuration
    async fn load_theme_config(&self, theme_path: &Path) -> Result<Theme> {
        let config_path = theme_path.join("theme.toml");
        
        if !config_path.exists() {
            return Err(ReedError::ThemeConfigMissing(
                theme_path.display().to_string()
            ));
        }
        
        let content = tokio::fs::read_to_string(&config_path).await?;
        let config: ThemeConfig = toml::from_str(&content)?;
        
        let theme_name = theme_path.file_name()
            .and_then(|n| n.to_str())
            .ok_or_else(|| ReedError::InvalidThemeName)?;
        
        Ok(Theme {
            name: theme_name.to_string(),
            parent: config.extends,
            context: self.parse_theme_context(&config.context),
            metadata: ThemeMetadata {
                version: config.version,
                author: config.author,
                description: config.description,
                preview_image: config.preview,
                requirements: ThemeRequirements {
                    min_version: config.min_version,
                    required_plugins: config.plugins.unwrap_or_default(),
                    required_features: config.features.unwrap_or_default(),
                },
            },
            active: config.active.unwrap_or(true),
        })
    }
    
    fn parse_theme_context(&self, context: &Option<ThemeContextConfig>) -> ThemeContext {
        match context {
            None => ThemeContext::Default,
            Some(ctx) => {
                if let Some(loc) = &ctx.location {
                    ThemeContext::Location(loc.clone())
                } else if let Some(season) = &ctx.season {
                    ThemeContext::Season(season.clone())
                } else if let Some(event) = &ctx.event {
                    ThemeContext::Event(event.clone())
                } else if let Some(custom) = &ctx.custom {
                    ThemeContext::Custom(custom.clone())
                } else {
                    ThemeContext::Default
                }
            }
        }
    }
}

#[derive(Debug, Deserialize)]
struct ThemeConfig {
    version: String,
    author: String,
    description: String,
    extends: Option<String>,
    active: Option<bool>,
    preview: Option<String>,
    min_version: Option<String>,
    plugins: Option<Vec<String>>,
    features: Option<Vec<String>>,
    context: Option<ThemeContextConfig>,
}

#[derive(Debug, Deserialize)]
struct ThemeContextConfig {
    location: Option<String>,
    season: Option<String>,
    event: Option<String>,
    custom: Option<String>,
}
```

## Theme Chain Optimization

```rust
impl ThemeManager {
    /// Optimize theme chain by removing redundant themes
    pub async fn optimize_chain(&self, chain: Vec<String>) -> Vec<String> {
        let mut optimized = Vec::new();
        let mut seen_files = HashSet::new();
        
        for theme_name in chain {
            let theme_path = self.config.themes_path.join(&theme_name);
            let mut has_unique_files = false;
            
            // Check if theme has unique files
            for file_type in &[FileType::Template, FileType::Partial, FileType::Asset] {
                let dir = theme_path.join(file_type.directory());
                
                if let Ok(entries) = self.scan_directory(&dir).await {
                    for file in entries {
                        if seen_files.insert(file.clone()) {
                            has_unique_files = true;
                        }
                    }
                }
            }
            
            // Only include theme if it has unique files
            if has_unique_files || theme_name == "base" {
                optimized.push(theme_name);
            }
        }
        
        optimized
    }
    
    async fn scan_directory(&self, dir: &Path) -> Result<Vec<String>> {
        let mut files = Vec::new();
        
        if !dir.exists() {
            return Ok(files);
        }
        
        let mut entries = tokio::fs::read_dir(dir).await?;
        
        while let Some(entry) = entries.next_entry().await? {
            if entry.file_type().await?.is_file() {
                if let Some(name) = entry.file_name().to_str() {
                    files.push(name.to_string());
                }
            }
        }
        
        Ok(files)
    }
}
```

## Theme Chain Caching

```rust
/// Cache for computed theme chains
pub struct ThemeChainCache {
    chains: Arc<RwLock<HashMap<String, CachedChain>>>,
    ttl: Duration,
}

#[derive(Debug, Clone)]
struct CachedChain {
    chain: Vec<String>,
    computed_at: Instant,
    access_count: u64,
}

impl ThemeChainCache {
    pub fn new(ttl: Duration) -> Self {
        Self {
            chains: Arc::new(RwLock::new(HashMap::new())),
            ttl,
        }
    }
    
    /// Get cached chain
    pub async fn get(&self, key: &str) -> Option<Vec<String>> {
        let mut chains = self.chains.write().await;
        
        if let Some(cached) = chains.get_mut(key) {
            if cached.computed_at.elapsed() < self.ttl {
                cached.access_count += 1;
                return Some(cached.chain.clone());
            }
        }
        
        None
    }
    
    /// Store chain in cache
    pub async fn set(&self, key: String, chain: Vec<String>) {
        let mut chains = self.chains.write().await;
        
        chains.insert(key, CachedChain {
            chain,
            computed_at: Instant::now(),
            access_count: 1,
        });
        
        // Clean old entries
        self.clean_expired(&mut chains);
    }
    
    fn clean_expired(&self, chains: &mut HashMap<String, CachedChain>) {
        let expired: Vec<String> = chains.iter()
            .filter(|(_, cached)| cached.computed_at.elapsed() > self.ttl * 2)
            .map(|(key, _)| key.clone())
            .collect();
        
        for key in expired {
            chains.remove(&key);
        }
    }
    
    /// Get cache statistics
    pub async fn stats(&self) -> CacheStats {
        let chains = self.chains.read().await;
        
        let total = chains.len();
        let total_accesses: u64 = chains.values()
            .map(|c| c.access_count)
            .sum();
        
        let most_used = chains.iter()
            .max_by_key(|(_, c)| c.access_count)
            .map(|(k, c)| (k.clone(), c.access_count));
        
        CacheStats {
            total_chains: total,
            total_accesses,
            most_used_chain: most_used,
            avg_chain_length: if total > 0 {
                chains.values().map(|c| c.chain.len()).sum::<usize>() / total
            } else {
                0
            },
        }
    }
}

#[derive(Debug, Serialize)]
pub struct CacheStats {
    pub total_chains: usize,
    pub total_accesses: u64,
    pub most_used_chain: Option<(String, u64)>,
    pub avg_chain_length: usize,
}
```

## Summary

Dieses Theme Chain Building System bietet:
- **Context-Aware Selection** - Themes basierend auf Location, Season, Events
- **Inheritance Support** - Child Themes erben von Parents
- **Score-based Selection** - Beste Theme-Auswahl für Context
- **Chain Optimization** - Entfernung redundanter Themes
- **Performance Caching** - Schnelle Chain Resolution

Theme Chains sind das Herzstück von ReedCMS's Flexibilität.
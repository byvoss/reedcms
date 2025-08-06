# Tera Context Building

## Overview

Context Building für Tera Templates. Strukturierter Aufbau des Template-Kontexts mit allen notwendigen Daten für das Rendering.

## Context Builder Core

```rust
use tera::Context;
use serde_json::Value;
use std::collections::HashMap;

/// Builds Tera context with all necessary data
pub struct ContextBuilder {
    ucg_query: Arc<UcgQuery>,
    snippet_manager: Arc<SnippetManager>,
    translation_manager: Arc<TranslationManager>,
    theme_manager: Arc<ThemeManager>,
    config: Arc<Config>,
}

impl ContextBuilder {
    pub fn new(
        ucg_query: Arc<UcgQuery>,
        snippet_manager: Arc<SnippetManager>,
        translation_manager: Arc<TranslationManager>,
        theme_manager: Arc<ThemeManager>,
        config: Arc<Config>,
    ) -> Self {
        Self {
            ucg_query,
            snippet_manager,
            translation_manager,
            theme_manager,
            config,
        }
    }
    
    /// Build context for page rendering
    pub async fn build_page_context(
        &self,
        page: &Entity,
        request: &RequestContext,
    ) -> Result<Context> {
        let mut context = Context::new();
        
        // Core data
        context.insert("page", page);
        context.insert("request", request);
        
        // Build navigation
        let navigation = self.build_navigation(request).await?;
        context.insert("navigation", &navigation);
        
        // Build breadcrumbs
        let breadcrumbs = self.build_breadcrumbs(page).await?;
        context.insert("breadcrumbs", &breadcrumbs);
        
        // Load page snippets
        let snippets = self.load_page_snippets(page).await?;
        context.insert("snippets", &snippets);
        
        // Global data
        self.add_global_data(&mut context, request).await?;
        
        // SEO metadata
        let seo = self.build_seo_data(page, request).await?;
        context.insert("seo", &seo);
        
        // Theme data
        let theme_data = self.build_theme_data(request).await?;
        context.insert("theme", &theme_data);
        
        Ok(context)
    }
    
    /// Build context for snippet rendering
    pub async fn build_snippet_context(
        &self,
        snippet: &Entity,
        parent_context: Option<&Context>,
    ) -> Result<Context> {
        let mut context = if let Some(parent) = parent_context {
            parent.clone()
        } else {
            Context::new()
        };
        
        // Add snippet data
        context.insert("snippet", snippet);
        
        // Add snippet-specific helpers
        context.insert("snippet_id", &snippet.id.to_string());
        context.insert("snippet_type", &snippet.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown"));
        
        Ok(context)
    }
}
```

## Navigation Building

```rust
#[derive(Debug, Serialize)]
pub struct Navigation {
    pub main: Vec<NavigationItem>,
    pub footer: Vec<NavigationItem>,
    pub mobile: Vec<NavigationItem>,
}

#[derive(Debug, Serialize)]
pub struct NavigationItem {
    pub id: Uuid,
    pub title: String,
    pub url: String,
    pub active: bool,
    pub children: Vec<NavigationItem>,
    pub meta: HashMap<String, Value>,
}

impl ContextBuilder {
    /// Build navigation structure
    async fn build_navigation(&self, request: &RequestContext) -> Result<Navigation> {
        // Get navigation root
        let nav_root = self.ucg_query
            .get_by_semantic_name("navigation.main")
            .await?
            .ok_or_else(|| ReedError::NotFound("Main navigation not found".into()))?;
        
        // Build navigation tree
        let main = self.build_navigation_tree(&nav_root, request, 0).await?;
        
        // Footer navigation
        let footer = if let Some(footer_root) = self.ucg_query
            .get_by_semantic_name("navigation.footer")
            .await?
        {
            self.build_navigation_tree(&footer_root, request, 0).await?
        } else {
            vec![]
        };
        
        // Mobile navigation (could be different)
        let mobile = main.clone(); // Or build separate mobile nav
        
        Ok(Navigation { main, footer, mobile })
    }
    
    /// Recursively build navigation tree
    async fn build_navigation_tree(
        &self,
        parent: &Entity,
        request: &RequestContext,
        depth: u32,
    ) -> Result<Vec<NavigationItem>> {
        if depth > 5 {
            return Ok(vec![]); // Prevent infinite recursion
        }
        
        let children = self.ucg_query
            .get_children(parent.id, Some("navigation-item"))
            .await?;
        
        let mut items = Vec::new();
        
        for child in children {
            let title = child.data.get("title")
                .and_then(|v| v.as_str())
                .unwrap_or("Untitled")
                .to_string();
            
            let url = child.data.get("url")
                .and_then(|v| v.as_str())
                .unwrap_or("#")
                .to_string();
            
            let active = self.is_active_navigation(&url, &request.path);
            
            let child_items = if child.data.get("has_children")
                .and_then(|v| v.as_bool())
                .unwrap_or(false)
            {
                self.build_navigation_tree(&child, request, depth + 1).await?
            } else {
                vec![]
            };
            
            items.push(NavigationItem {
                id: child.id,
                title,
                url,
                active,
                children: child_items,
                meta: child.data.as_object()
                    .map(|o| o.iter()
                        .filter(|(k, _)| !k.starts_with('_'))
                        .map(|(k, v)| (k.clone(), v.clone()))
                        .collect())
                    .unwrap_or_default(),
            });
        }
        
        Ok(items)
    }
    
    fn is_active_navigation(&self, nav_url: &str, current_path: &str) -> bool {
        if nav_url == current_path {
            return true;
        }
        
        // Check if current path is child of nav_url
        if nav_url != "/" && current_path.starts_with(nav_url) {
            return true;
        }
        
        false
    }
}
```

## Breadcrumb Building

```rust
#[derive(Debug, Serialize)]
pub struct Breadcrumb {
    pub title: String,
    pub url: Option<String>,
}

impl ContextBuilder {
    /// Build breadcrumb trail
    async fn build_breadcrumbs(&self, page: &Entity) -> Result<Vec<Breadcrumb>> {
        let mut breadcrumbs = Vec::new();
        
        // Always start with home
        breadcrumbs.push(Breadcrumb {
            title: "Home".to_string(),
            url: Some("/".to_string()),
        });
        
        // Get ancestors
        let ancestors = self.ucg_query
            .get_ancestors(page.id, false)
            .await?;
        
        // Build breadcrumb for each ancestor
        for ancestor in ancestors {
            let title = ancestor.data.get("title")
                .and_then(|v| v.as_str())
                .unwrap_or("Untitled")
                .to_string();
            
            let url = ancestor.data.get("slug")
                .and_then(|v| v.as_str())
                .map(|s| format!("/{}", s));
            
            breadcrumbs.push(Breadcrumb { title, url });
        }
        
        // Add current page (without URL)
        let current_title = page.data.get("title")
            .and_then(|v| v.as_str())
            .unwrap_or("Current Page")
            .to_string();
        
        breadcrumbs.push(Breadcrumb {
            title: current_title,
            url: None,
        });
        
        Ok(breadcrumbs)
    }
}
```

## Global Data Loading

```rust
impl ContextBuilder {
    /// Add global data to context
    async fn add_global_data(
        &self,
        context: &mut Context,
        request: &RequestContext,
    ) -> Result<()> {
        // Site configuration
        let site_config = self.build_site_config().await?;
        context.insert("site", &site_config);
        
        // Current user
        if let Some(user) = &request.user {
            context.insert("user", user);
            context.insert("is_authenticated", &true);
        } else {
            context.insert("is_authenticated", &false);
        }
        
        // Current locale
        context.insert("locale", &request.locale);
        context.insert("available_locales", &self.config.locales);
        
        // Current time
        context.insert("now", &chrono::Utc::now());
        
        // Environment
        context.insert("environment", &self.config.environment);
        context.insert("is_development", &(self.config.environment == "development"));
        
        // Feature flags
        let features = self.get_feature_flags(request).await?;
        context.insert("features", &features);
        
        Ok(())
    }
    
    /// Build site configuration
    async fn build_site_config(&self) -> Result<HashMap<String, Value>> {
        let mut config = HashMap::new();
        
        // Load from global snippets
        if let Some(site_entity) = self.ucg_query
            .get_by_semantic_name("site.config")
            .await?
        {
            if let Some(data) = site_entity.data.as_object() {
                for (key, value) in data {
                    config.insert(key.clone(), value.clone());
                }
            }
        }
        
        // Add defaults
        config.entry("title".to_string())
            .or_insert(Value::String(self.config.site_name.clone()));
        config.entry("url".to_string())
            .or_insert(Value::String(self.config.base_url.clone()));
        
        Ok(config)
    }
    
    /// Get active feature flags
    async fn get_feature_flags(&self, request: &RequestContext) -> Result<HashMap<String, bool>> {
        let mut features = HashMap::new();
        
        // Load feature configuration
        if let Some(features_entity) = self.ucg_query
            .get_by_semantic_name("features.config")
            .await?
        {
            if let Some(data) = features_entity.data.as_object() {
                for (key, value) in data {
                    if let Some(enabled) = value.as_bool() {
                        features.insert(key.clone(), enabled);
                    }
                }
            }
        }
        
        // Check user-specific features
        if let Some(user) = &request.user {
            // Add user role features
            for role in &user.roles {
                features.insert(format!("role_{}", role), true);
            }
        }
        
        Ok(features)
    }
}
```

## SEO Data Building

```rust
#[derive(Debug, Serialize)]
pub struct SeoData {
    pub title: String,
    pub description: String,
    pub keywords: Vec<String>,
    pub canonical_url: String,
    pub og_data: OpenGraphData,
    pub twitter_data: TwitterCardData,
    pub json_ld: Value,
}

#[derive(Debug, Serialize)]
pub struct OpenGraphData {
    pub title: String,
    pub description: String,
    pub image: Option<String>,
    pub url: String,
    pub site_name: String,
    pub og_type: String,
}

#[derive(Debug, Serialize)]
pub struct TwitterCardData {
    pub card: String,
    pub title: String,
    pub description: String,
    pub image: Option<String>,
}

impl ContextBuilder {
    /// Build SEO metadata
    async fn build_seo_data(
        &self,
        page: &Entity,
        request: &RequestContext,
    ) -> Result<SeoData> {
        // Get page SEO data
        let seo_title = page.data.get("seo_title")
            .or_else(|| page.data.get("title"))
            .and_then(|v| v.as_str())
            .unwrap_or("Untitled Page");
        
        let seo_description = page.data.get("seo_description")
            .or_else(|| page.data.get("description"))
            .and_then(|v| v.as_str())
            .unwrap_or("");
        
        let keywords = page.data.get("keywords")
            .and_then(|v| v.as_array())
            .map(|arr| arr.iter()
                .filter_map(|v| v.as_str())
                .map(|s| s.to_string())
                .collect())
            .unwrap_or_default();
        
        let canonical_url = format!("{}{}", self.config.base_url, request.path);
        
        // Build OpenGraph data
        let og_data = OpenGraphData {
            title: seo_title.to_string(),
            description: seo_description.to_string(),
            image: page.data.get("og_image")
                .and_then(|v| v.as_str())
                .map(|s| s.to_string()),
            url: canonical_url.clone(),
            site_name: self.config.site_name.clone(),
            og_type: page.data.get("og_type")
                .and_then(|v| v.as_str())
                .unwrap_or("website")
                .to_string(),
        };
        
        // Build Twitter Card data
        let twitter_data = TwitterCardData {
            card: "summary_large_image".to_string(),
            title: seo_title.to_string(),
            description: seo_description.to_string(),
            image: page.data.get("twitter_image")
                .or_else(|| page.data.get("og_image"))
                .and_then(|v| v.as_str())
                .map(|s| s.to_string()),
        };
        
        // Build JSON-LD
        let json_ld = self.build_json_ld(page, &canonical_url)?;
        
        Ok(SeoData {
            title: seo_title.to_string(),
            description: seo_description.to_string(),
            keywords,
            canonical_url,
            og_data,
            twitter_data,
            json_ld,
        })
    }
    
    /// Build JSON-LD structured data
    fn build_json_ld(&self, page: &Entity, url: &str) -> Result<Value> {
        let mut json_ld = serde_json::json!({
            "@context": "https://schema.org",
            "@type": "WebPage",
            "url": url,
            "name": page.data.get("title"),
            "description": page.data.get("description"),
        });
        
        // Add breadcrumb list
        if let Some(obj) = json_ld.as_object_mut() {
            obj.insert("breadcrumb".to_string(), serde_json::json!({
                "@type": "BreadcrumbList",
                "itemListElement": []
            }));
        }
        
        Ok(json_ld)
    }
}
```

## Theme Data Building

```rust
#[derive(Debug, Serialize)]
pub struct ThemeData {
    pub name: String,
    pub assets_path: String,
    pub css_files: Vec<String>,
    pub js_files: Vec<String>,
    pub settings: HashMap<String, Value>,
}

impl ContextBuilder {
    /// Build theme-specific data
    async fn build_theme_data(&self, request: &RequestContext) -> Result<ThemeData> {
        let theme = &request.theme;
        
        // Get theme configuration
        let theme_config = self.theme_manager
            .get_theme_config(&theme.name)
            .await?;
        
        Ok(ThemeData {
            name: theme.name.clone(),
            assets_path: format!("/themes/{}/assets", theme.name),
            css_files: theme_config.css_files,
            js_files: theme_config.js_files,
            settings: theme_config.settings,
        })
    }
}
```

## Page Snippet Loading

```rust
impl ContextBuilder {
    /// Load all snippets for a page
    async fn load_page_snippets(&self, page: &Entity) -> Result<HashMap<String, Vec<Entity>>> {
        let mut snippets_by_region = HashMap::new();
        
        // Get all child snippets
        let children = self.ucg_query
            .get_children(page.id, Some("snippet"))
            .await?;
        
        // Group by region
        for child in children {
            let region = child.data.get("region")
                .and_then(|v| v.as_str())
                .unwrap_or("content")
                .to_string();
            
            snippets_by_region
                .entry(region)
                .or_insert_with(Vec::new)
                .push(child);
        }
        
        // Sort each region by weight
        for snippets in snippets_by_region.values_mut() {
            snippets.sort_by_key(|s| {
                s.data.get("weight")
                    .and_then(|v| v.as_i64())
                    .unwrap_or(0)
            });
        }
        
        Ok(snippets_by_region)
    }
}
```

## Context Caching

```rust
/// Caches built contexts for performance
pub struct ContextCache {
    cache: Arc<RwLock<HashMap<String, (Context, Instant)>>>,
    ttl: Duration,
}

impl ContextCache {
    pub fn new(ttl: Duration) -> Self {
        Self {
            cache: Arc::new(RwLock::new(HashMap::new())),
            ttl,
        }
    }
    
    /// Get cached context if fresh
    pub async fn get(&self, key: &str) -> Option<Context> {
        let cache = self.cache.read().await;
        
        if let Some((context, created)) = cache.get(key) {
            if created.elapsed() < self.ttl {
                return Some(context.clone());
            }
        }
        
        None
    }
    
    /// Store context in cache
    pub async fn set(&self, key: String, context: Context) {
        let mut cache = self.cache.write().await;
        cache.insert(key, (context, Instant::now()));
        
        // Clean old entries
        cache.retain(|_, (_, created)| created.elapsed() < self.ttl * 2);
    }
}
```

## Summary

Dieses Context Building System bietet:
- **Structured Data** - Klare Hierarchie für Template-Daten
- **Navigation Building** - Automatische Navigation mit Active-States
- **SEO Integration** - OpenGraph, Twitter Cards, JSON-LD
- **Performance** - Context Caching für häufige Requests
- **Extensibility** - Einfach erweiterbar für neue Datentypen

Context Building ist das Herzstück des Template Systems.
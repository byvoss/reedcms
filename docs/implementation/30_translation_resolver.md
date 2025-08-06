# Translation Resolver System

## Overview

Resolution Logic für das Translation System. Context-aware Translation Resolution mit Dynamic Scoping und Namespace Support.

## Translation Resolver Core

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

/// Resolves translations with context awareness
pub struct TranslationResolver {
    manager: Arc<TranslationManager>,
    context_provider: Arc<ContextProvider>,
    namespace_stack: Arc<RwLock<Vec<String>>>,
    resolution_cache: Arc<ResolutionCache>,
}

impl TranslationResolver {
    pub fn new(
        manager: Arc<TranslationManager>,
        context_provider: Arc<ContextProvider>,
    ) -> Self {
        Self {
            manager,
            context_provider,
            namespace_stack: Arc::new(RwLock::new(Vec::new())),
            resolution_cache: Arc::new(ResolutionCache::new()),
        }
    }
    
    /// Resolve translation with full context
    pub async fn resolve(
        &self,
        key: &str,
        locale: Option<&str>,
        params: Option<&HashMap<String, String>>,
    ) -> Result<String> {
        // Get current context
        let context = self.context_provider.get_current_context().await?;
        let locale = locale.unwrap_or(&context.locale);
        
        // Build resolution key
        let resolution_key = self.build_resolution_key(key, locale, &context).await?;
        
        // Check cache
        if let Some(cached) = self.resolution_cache.get(&resolution_key).await? {
            return Ok(self.apply_params(&cached, params.unwrap_or(&HashMap::new())));
        }
        
        // Resolve through layers
        let translation = self.resolve_through_layers(key, locale, &context).await?;
        
        // Cache result
        self.resolution_cache.set(resolution_key, translation.clone()).await?;
        
        // Apply parameters
        Ok(self.apply_params(&translation, params.unwrap_or(&HashMap::new())))
    }
    
    /// Push namespace onto stack
    pub async fn push_namespace(&self, namespace: &str) -> Result<()> {
        let mut stack = self.namespace_stack.write().await;
        stack.push(namespace.to_string());
        Ok(())
    }
    
    /// Pop namespace from stack
    pub async fn pop_namespace(&self) -> Result<Option<String>> {
        let mut stack = self.namespace_stack.write().await;
        Ok(stack.pop())
    }
}
```

## Context Provider

```rust
/// Provides translation context
pub struct ContextProvider {
    request_context: Arc<RwLock<Option<RequestContext>>>,
    static_context: StaticContext,
}

#[derive(Debug, Clone)]
pub struct TranslationContext {
    pub locale: String,
    pub theme: String,
    pub snippet_type: Option<String>,
    pub user_preferences: UserPreferences,
    pub features: HashSet<String>,
    pub custom_data: HashMap<String, String>,
}

#[derive(Debug, Clone)]
pub struct UserPreferences {
    pub preferred_locale: Option<String>,
    pub date_format: DateFormat,
    pub number_format: NumberFormat,
    pub timezone: String,
}

#[derive(Debug, Clone)]
pub enum DateFormat {
    ISO,      // 2024-03-14
    US,       // 03/14/2024
    European, // 14.03.2024
    Custom(String),
}

#[derive(Debug, Clone)]
pub enum NumberFormat {
    Decimal,    // 1,234.56
    European,   // 1.234,56
    Scientific, // 1.23456e3
    Custom(String),
}

impl ContextProvider {
    /// Get current translation context
    pub async fn get_current_context(&self) -> Result<TranslationContext> {
        if let Some(request_ctx) = self.request_context.read().await.as_ref() {
            Ok(self.build_context_from_request(request_ctx))
        } else {
            Ok(self.get_default_context())
        }
    }
    
    /// Build context from request
    fn build_context_from_request(&self, request: &RequestContext) -> TranslationContext {
        TranslationContext {
            locale: request.locale.clone(),
            theme: request.theme.name.clone(),
            snippet_type: request.current_snippet_type.clone(),
            user_preferences: self.get_user_preferences(&request.user),
            features: request.features.clone(),
            custom_data: request.custom_data.clone(),
        }
    }
    
    /// Get user preferences
    fn get_user_preferences(&self, user: &Option<UserContext>) -> UserPreferences {
        if let Some(user) = user {
            UserPreferences {
                preferred_locale: user.preferences.get("locale").cloned(),
                date_format: self.parse_date_format(
                    user.preferences.get("date_format").map(|s| s.as_str())
                ),
                number_format: self.parse_number_format(
                    user.preferences.get("number_format").map(|s| s.as_str())
                ),
                timezone: user.preferences.get("timezone")
                    .cloned()
                    .unwrap_or_else(|| "UTC".to_string()),
            }
        } else {
            self.get_default_preferences()
        }
    }
}
```

## Layer Resolution

```rust
impl TranslationResolver {
    /// Resolve through translation layers
    async fn resolve_through_layers(
        &self,
        key: &str,
        locale: &str,
        context: &TranslationContext,
    ) -> Result<String> {
        // Build key variants with namespaces
        let key_variants = self.build_key_variants(key).await?;
        
        // Build locale chain
        let locale_chain = self.build_locale_chain(locale, &context.user_preferences);
        
        // Try each key variant
        for key_variant in &key_variants {
            // Try each locale
            for loc in &locale_chain {
                // Try each layer
                for layer in self.get_layers_for_context(context) {
                    if let Some(translation) = self.manager
                        .get_from_layer(loc, key_variant, &layer)
                        .await?
                    {
                        return Ok(translation);
                    }
                }
            }
        }
        
        // Fallback to key
        Ok(self.format_missing_key(key))
    }
    
    /// Build key variants with namespaces
    async fn build_key_variants(&self, key: &str) -> Result<Vec<String>> {
        let mut variants = Vec::new();
        
        // Get current namespace stack
        let stack = self.namespace_stack.read().await;
        
        // Full namespaced key
        if !stack.is_empty() {
            let namespace = stack.join(".");
            variants.push(format!("{}.{}", namespace, key));
        }
        
        // Key with each namespace level
        for i in (0..stack.len()).rev() {
            let partial_namespace = stack[..=i].join(".");
            variants.push(format!("{}.{}", partial_namespace, key));
        }
        
        // Original key
        variants.push(key.to_string());
        
        Ok(variants)
    }
    
    /// Build locale fallback chain
    fn build_locale_chain(
        &self,
        locale: &str,
        preferences: &UserPreferences,
    ) -> Vec<String> {
        let mut chain = Vec::new();
        
        // Requested locale
        chain.push(locale.to_string());
        
        // User preferred locale
        if let Some(preferred) = &preferences.preferred_locale {
            if preferred != locale {
                chain.push(preferred.clone());
            }
        }
        
        // Language without region
        if locale.contains('-') {
            let lang = locale.split('-').next().unwrap();
            chain.push(lang.to_string());
        }
        
        // Default locale
        chain.push(self.manager.default_locale.clone());
        
        // English fallback
        if !chain.contains(&"en".to_string()) {
            chain.push("en".to_string());
        }
        
        chain
    }
    
    /// Get layers for current context
    fn get_layers_for_context(&self, context: &TranslationContext) -> Vec<TranslationLayer> {
        let mut layers = Vec::new();
        
        // Snippet layer (highest priority)
        if let Some(snippet_type) = &context.snippet_type {
            layers.push(TranslationLayer::Snippet(snippet_type.clone()));
        }
        
        // Feature layers
        for feature in &context.features {
            layers.push(TranslationLayer::Feature(feature.clone()));
        }
        
        // Theme layer
        layers.push(TranslationLayer::Theme(context.theme.clone()));
        
        // Global layer (lowest priority)
        layers.push(TranslationLayer::Global);
        
        layers
    }
}
```

## Advanced Resolution

```rust
impl TranslationResolver {
    /// Resolve with inheritance
    pub async fn resolve_with_inheritance(
        &self,
        key: &str,
        locale: &str,
        inherit_from: Option<&str>,
    ) -> Result<String> {
        // Try direct resolution first
        if let Ok(translation) = self.resolve(key, Some(locale), None).await {
            if !self.is_missing_key(&translation) {
                return Ok(translation);
            }
        }
        
        // Try inheritance
        if let Some(parent) = inherit_from {
            if let Ok(translation) = self.resolve(parent, Some(locale), None).await {
                return Ok(translation);
            }
        }
        
        // Fallback
        Ok(self.format_missing_key(key))
    }
    
    /// Resolve with context override
    pub async fn resolve_with_override(
        &self,
        key: &str,
        locale: &str,
        override_context: HashMap<String, String>,
    ) -> Result<String> {
        // Build special key with context
        let context_key = self.build_contextual_key(key, &override_context);
        
        // Try contextual key first
        if let Ok(translation) = self.resolve(&context_key, Some(locale), None).await {
            if !self.is_missing_key(&translation) {
                return Ok(translation);
            }
        }
        
        // Fall back to regular resolution
        self.resolve(key, Some(locale), None).await
    }
    
    /// Build contextual key
    fn build_contextual_key(
        &self,
        base_key: &str,
        context: &HashMap<String, String>,
    ) -> String {
        let mut parts = vec![base_key.to_string()];
        
        // Add context values in consistent order
        let mut context_parts: Vec<String> = context.iter()
            .map(|(k, v)| format!("{}_{}", k, v))
            .collect();
        context_parts.sort();
        
        parts.extend(context_parts);
        parts.join(".")
    }
}
```

## Resolution Cache

```rust
/// Cache for resolved translations
pub struct ResolutionCache {
    cache: Arc<RwLock<HashMap<String, CachedTranslation>>>,
    ttl: Duration,
}

#[derive(Debug, Clone)]
struct CachedTranslation {
    translation: String,
    cached_at: Instant,
    access_count: u64,
}

impl ResolutionCache {
    pub fn new() -> Self {
        Self {
            cache: Arc::new(RwLock::new(HashMap::new())),
            ttl: Duration::from_secs(300), // 5 minutes
        }
    }
    
    /// Get from cache
    pub async fn get(&self, key: &str) -> Result<Option<String>> {
        let mut cache = self.cache.write().await;
        
        if let Some(cached) = cache.get_mut(key) {
            if cached.cached_at.elapsed() < self.ttl {
                cached.access_count += 1;
                return Ok(Some(cached.translation.clone()));
            } else {
                // Remove stale entry
                cache.remove(key);
            }
        }
        
        Ok(None)
    }
    
    /// Set in cache
    pub async fn set(&self, key: String, translation: String) -> Result<()> {
        let mut cache = self.cache.write().await;
        
        cache.insert(key, CachedTranslation {
            translation,
            cached_at: Instant::now(),
            access_count: 1,
        });
        
        // Clean old entries if cache is too large
        if cache.len() > 10000 {
            self.clean_old_entries(&mut cache);
        }
        
        Ok(())
    }
    
    /// Clean old cache entries
    fn clean_old_entries(&self, cache: &mut HashMap<String, CachedTranslation>) {
        let now = Instant::now();
        let expired: Vec<String> = cache.iter()
            .filter(|(_, v)| now.duration_since(v.cached_at) > self.ttl * 2)
            .map(|(k, _)| k.clone())
            .collect();
        
        for key in expired {
            cache.remove(&key);
        }
    }
}
```

## Parameter Processing

```rust
impl TranslationResolver {
    /// Apply parameters to translation
    fn apply_params(&self, translation: &str, params: &HashMap<String, String>) -> String {
        let mut result = translation.to_string();
        
        // Simple replacements
        for (key, value) in params {
            result = result.replace(&format!("{{{}}}", key), value);
        }
        
        // Advanced replacements
        result = self.apply_advanced_params(result, params);
        
        result
    }
    
    /// Apply advanced parameter processing
    fn apply_advanced_params(
        &self,
        mut text: String,
        params: &HashMap<String, String>,
    ) -> String {
        // Conditional blocks: {?has_items}...{/has_items}
        let conditional_re = regex::Regex::new(r"\{\?(\w+)\}(.*?)\{/\1\}").unwrap();
        for cap in conditional_re.captures_iter(&text.clone()) {
            if let (Some(key), Some(content)) = (cap.get(1), cap.get(2)) {
                let should_show = params.get(key.as_str())
                    .map(|v| v != "0" && v != "false" && !v.is_empty())
                    .unwrap_or(false);
                
                let replacement = if should_show {
                    content.as_str()
                } else {
                    ""
                };
                
                text = text.replace(&cap[0], replacement);
            }
        }
        
        // Lists: {#items}...{/items}
        let list_re = regex::Regex::new(r"\{#(\w+)\}(.*?)\{/\1\}").unwrap();
        for cap in list_re.captures_iter(&text.clone()) {
            if let (Some(key), Some(template)) = (cap.get(1), cap.get(2)) {
                if let Some(items_json) = params.get(key.as_str()) {
                    if let Ok(items) = serde_json::from_str::<Vec<HashMap<String, String>>>(items_json) {
                        let rendered: Vec<String> = items.iter()
                            .map(|item| self.apply_params(template.as_str(), item))
                            .collect();
                        
                        text = text.replace(&cap[0], &rendered.join(""));
                    }
                }
            }
        }
        
        text
    }
}
```

## Translation Helpers

```rust
impl TranslationResolver {
    /// Format missing translation key
    fn format_missing_key(&self, key: &str) -> String {
        #[cfg(debug_assertions)]
        {
            format!("[MISSING: {}]", key)
        }
        
        #[cfg(not(debug_assertions))]
        {
            // In production, return key as fallback
            key.to_string()
        }
    }
    
    /// Check if translation is missing
    fn is_missing_key(&self, translation: &str) -> bool {
        translation.starts_with("[MISSING:") && translation.ends_with(']')
    }
    
    /// Build resolution cache key
    async fn build_resolution_key(
        &self,
        key: &str,
        locale: &str,
        context: &TranslationContext,
    ) -> Result<String> {
        let namespace = self.namespace_stack.read().await.join(".");
        
        format!(
            "{}:{}:{}:{}:{}",
            locale,
            namespace,
            key,
            context.theme,
            context.snippet_type.as_ref().unwrap_or(&"none".to_string())
        )
    }
}
```

## Batch Resolution

```rust
impl TranslationResolver {
    /// Resolve multiple translations in batch
    pub async fn resolve_batch(
        &self,
        keys: Vec<&str>,
        locale: Option<&str>,
    ) -> Result<HashMap<String, String>> {
        let mut results = HashMap::new();
        
        // Group by namespace for efficiency
        let mut grouped: HashMap<String, Vec<&str>> = HashMap::new();
        for key in keys {
            let namespace = self.extract_namespace(key);
            grouped.entry(namespace).or_insert_with(Vec::new).push(key);
        }
        
        // Resolve each group
        for (namespace, group_keys) in grouped {
            if !namespace.is_empty() {
                self.push_namespace(&namespace).await?;
            }
            
            for key in group_keys {
                let translation = self.resolve(key, locale, None).await?;
                results.insert(key.to_string(), translation);
            }
            
            if !namespace.is_empty() {
                self.pop_namespace().await?;
            }
        }
        
        Ok(results)
    }
    
    /// Extract namespace from key
    fn extract_namespace(&self, key: &str) -> String {
        if let Some(pos) = key.rfind('.') {
            key[..pos].to_string()
        } else {
            String::new()
        }
    }
}
```

## Summary

Dieses Resolution System bietet:
- **Context-Aware Resolution** - Theme, Snippet, Feature basierte Auflösung
- **Namespace Support** - Hierarchische Translation Keys
- **Smart Caching** - Resolution Cache mit TTL
- **Advanced Parameters** - Conditionals und Lists in Translations
- **Batch Operations** - Effiziente Multi-Key Resolution

Translation Resolution macht i18n in ReedCMS powerful und flexibel.
# Translation Loader System

## Overview

Multi-Layer Translation Loading für ReedCMS. Hierarchisches System mit Theme-aware Resolution und CSV-basierter Storage.

## Translation Manager Core

```rust
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use glob::glob;

/// Manages multi-layer translations
pub struct TranslationManager {
    base_path: PathBuf,
    epc_resolver: Arc<EpcResolver>,
    cache: Arc<TranslationCache>,
    default_locale: String,
    fallback_chain: Vec<String>,
}

impl TranslationManager {
    pub fn new(
        base_path: PathBuf,
        epc_resolver: Arc<EpcResolver>,
        default_locale: String,
    ) -> Self {
        Self {
            base_path,
            epc_resolver,
            cache: Arc::new(TranslationCache::new()),
            default_locale: default_locale.clone(),
            fallback_chain: vec![default_locale, "en".to_string()],
        }
    }
    
    /// Initialize translation system
    pub async fn initialize(&self) -> Result<()> {
        tracing::info!("Initializing translation system");
        
        // Load global translations
        self.load_global_translations().await?;
        
        // Load theme translations
        self.load_theme_translations().await?;
        
        // Load snippet translations
        self.load_snippet_translations().await?;
        
        tracing::info!("Translation system initialized");
        Ok(())
    }
    
    /// Translate key with parameters
    pub async fn translate(
        &self,
        key: &str,
        locale: Option<&str>,
        params: &HashMap<String, String>,
    ) -> Result<String> {
        let locale = locale.unwrap_or(&self.default_locale);
        
        // Try cache first
        if let Some(translation) = self.cache.get(locale, key).await? {
            return Ok(self.interpolate(&translation, params));
        }
        
        // Load and translate
        let translation = self.resolve_translation(key, locale).await?;
        
        // Cache result
        self.cache.set(locale, key, &translation).await?;
        
        Ok(self.interpolate(&translation, params))
    }
    
    /// Translate with plural forms
    pub async fn translate_plural(
        &self,
        key: &str,
        count: usize,
        locale: Option<&str>,
    ) -> Result<String> {
        let locale = locale.unwrap_or(&self.default_locale);
        
        // Determine plural form
        let plural_key = self.get_plural_key(key, count, locale);
        
        // Translate with count parameter
        let mut params = HashMap::new();
        params.insert("count".to_string(), count.to_string());
        
        self.translate(&plural_key, Some(locale), &params).await
    }
}
```

## Translation Loading

```rust
impl TranslationManager {
    /// Load global translations
    async fn load_global_translations(&self) -> Result<()> {
        let global_path = self.base_path.join("translations");
        
        if !global_path.exists() {
            return Ok(());
        }
        
        // Find all translation files
        let pattern = format!("{}/*.csv", global_path.display());
        
        for entry in glob(&pattern)? {
            if let Ok(path) = entry {
                self.load_translation_file(&path, TranslationLayer::Global).await?;
            }
        }
        
        Ok(())
    }
    
    /// Load theme translations
    async fn load_theme_translations(&self) -> Result<()> {
        let themes_path = self.base_path.join("themes");
        
        // Load translations for each theme
        let pattern = format!("{}/*/translations/*.csv", themes_path.display());
        
        for entry in glob(&pattern)? {
            if let Ok(path) = entry {
                // Extract theme name from path
                if let Some(theme_name) = self.extract_theme_name(&path) {
                    self.load_translation_file(
                        &path,
                        TranslationLayer::Theme(theme_name),
                    ).await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Load snippet translations
    async fn load_snippet_translations(&self) -> Result<()> {
        let snippets_path = self.base_path.join("snippets");
        
        // Load translations for each snippet type
        let pattern = format!("{}/*/translations/*.csv", snippets_path.display());
        
        for entry in glob(&pattern)? {
            if let Ok(path) = entry {
                // Extract snippet type from path
                if let Some(snippet_type) = self.extract_snippet_type(&path) {
                    self.load_translation_file(
                        &path,
                        TranslationLayer::Snippet(snippet_type),
                    ).await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Load single translation file
    async fn load_translation_file(
        &self,
        path: &Path,
        layer: TranslationLayer,
    ) -> Result<()> {
        // Extract locale from filename
        let locale = self.extract_locale_from_filename(path)?;
        
        // Read CSV file
        let mut reader = csv::Reader::from_path(path)?;
        let mut translations = HashMap::new();
        
        for result in reader.records() {
            let record = result?;
            
            if record.len() >= 2 {
                let key = record.get(0).unwrap_or("").to_string();
                let value = record.get(1).unwrap_or("").to_string();
                
                if !key.is_empty() {
                    translations.insert(key, value);
                }
            }
        }
        
        // Store in cache
        self.cache.store_layer(&locale, layer, translations).await?;
        
        tracing::debug!(
            "Loaded {} translations for locale '{}' from {:?}",
            translations.len(),
            locale,
            path
        );
        
        Ok(())
    }
}
```

## Translation Resolution

```rust
impl TranslationManager {
    /// Resolve translation through layers
    async fn resolve_translation(
        &self,
        key: &str,
        locale: &str,
    ) -> Result<String> {
        // Build fallback chain
        let locales = self.build_locale_chain(locale);
        
        // Try each locale in chain
        for loc in &locales {
            // Check each layer in order
            for layer in self.get_active_layers().await? {
                if let Some(translation) = self.cache
                    .get_from_layer(loc, key, &layer)
                    .await?
                {
                    return Ok(translation);
                }
            }
        }
        
        // Return key if no translation found
        Ok(key.to_string())
    }
    
    /// Build locale fallback chain
    fn build_locale_chain(&self, locale: &str) -> Vec<String> {
        let mut chain = vec![locale.to_string()];
        
        // Add language without region (e.g., "de" from "de-CH")
        if locale.contains('-') {
            let parts: Vec<&str> = locale.split('-').collect();
            if let Some(lang) = parts.first() {
                chain.push(lang.to_string());
            }
        }
        
        // Add configured fallbacks
        for fallback in &self.fallback_chain {
            if !chain.contains(fallback) {
                chain.push(fallback.clone());
            }
        }
        
        chain
    }
    
    /// Get active translation layers in priority order
    async fn get_active_layers(&self) -> Result<Vec<TranslationLayer>> {
        let mut layers = Vec::new();
        
        // Current request context would determine active layers
        // For now, return default order
        
        // 1. Snippet-specific translations (highest priority)
        if let Some(snippet_type) = self.get_current_snippet_type() {
            layers.push(TranslationLayer::Snippet(snippet_type));
        }
        
        // 2. Theme translations
        if let Some(theme) = self.get_current_theme() {
            layers.push(TranslationLayer::Theme(theme));
        }
        
        // 3. Global translations (lowest priority)
        layers.push(TranslationLayer::Global);
        
        Ok(layers)
    }
}

#[derive(Debug, Clone, Hash, Eq, PartialEq)]
pub enum TranslationLayer {
    Global,
    Theme(String),
    Snippet(String),
}
```

## Translation Interpolation

```rust
impl TranslationManager {
    /// Interpolate parameters into translation
    fn interpolate(&self, translation: &str, params: &HashMap<String, String>) -> String {
        let mut result = translation.to_string();
        
        // Replace {key} with value
        for (key, value) in params {
            let placeholder = format!("{{{}}}", key);
            result = result.replace(&placeholder, value);
        }
        
        // Handle special formatters
        result = self.apply_formatters(result, params);
        
        result
    }
    
    /// Apply special formatters
    fn apply_formatters(
        &self,
        mut text: String,
        params: &HashMap<String, String>,
    ) -> String {
        // Number formatting: {count:number}
        let number_re = regex::Regex::new(r"\{(\w+):number\}").unwrap();
        for cap in number_re.captures_iter(&text.clone()) {
            if let Some(key) = cap.get(1) {
                if let Some(value) = params.get(key.as_str()) {
                    if let Ok(num) = value.parse::<f64>() {
                        let formatted = self.format_number(num);
                        text = text.replace(&cap[0], &formatted);
                    }
                }
            }
        }
        
        // Date formatting: {date:short}
        let date_re = regex::Regex::new(r"\{(\w+):date(?::(\w+))?\}").unwrap();
        for cap in date_re.captures_iter(&text.clone()) {
            if let Some(key) = cap.get(1) {
                if let Some(value) = params.get(key.as_str()) {
                    let format = cap.get(2).map(|m| m.as_str()).unwrap_or("medium");
                    let formatted = self.format_date(value, format);
                    text = text.replace(&cap[0], &formatted);
                }
            }
        }
        
        text
    }
    
    /// Format number based on locale
    fn format_number(&self, num: f64) -> String {
        // Simple formatting, could use locale-specific rules
        if num.fract() == 0.0 {
            format!("{:.0}", num)
        } else {
            format!("{:.2}", num)
        }
    }
    
    /// Format date based on locale and format
    fn format_date(&self, date_str: &str, format: &str) -> String {
        // Parse and format date
        if let Ok(date) = chrono::DateTime::parse_from_rfc3339(date_str) {
            match format {
                "short" => date.format("%Y-%m-%d").to_string(),
                "long" => date.format("%B %d, %Y").to_string(),
                _ => date.format("%Y-%m-%d %H:%M").to_string(),
            }
        } else {
            date_str.to_string()
        }
    }
}
```

## Plural Forms

```rust
impl TranslationManager {
    /// Get plural key based on count and locale
    fn get_plural_key(&self, base_key: &str, count: usize, locale: &str) -> String {
        let plural_form = self.get_plural_form(count, locale);
        format!("{}.{}", base_key, plural_form)
    }
    
    /// Determine plural form for locale
    fn get_plural_form(&self, count: usize, locale: &str) -> &'static str {
        // Extract language code
        let lang = locale.split('-').next().unwrap_or(locale);
        
        match lang {
            // Germanic languages (English, German, Dutch, etc.)
            "en" | "de" | "nl" | "sv" | "da" | "no" => {
                if count == 1 { "one" } else { "other" }
            }
            
            // Romance languages with special zero
            "fr" | "pt-BR" => {
                if count == 0 || count == 1 { "one" } else { "other" }
            }
            
            // Slavic languages (complex rules)
            "ru" | "uk" | "pl" => {
                self.get_slavic_plural_form(count)
            }
            
            // No plurals
            "ja" | "zh" | "ko" | "th" => "other",
            
            // Default to English rules
            _ => if count == 1 { "one" } else { "other" }
        }
    }
    
    /// Get plural form for Slavic languages
    fn get_slavic_plural_form(&self, count: usize) -> &'static str {
        if count % 10 == 1 && count % 100 != 11 {
            "one"
        } else if (2..=4).contains(&(count % 10)) && !(12..=14).contains(&(count % 100)) {
            "few"
        } else {
            "many"
        }
    }
}
```

## Translation Cache

```rust
/// Cache for loaded translations
pub struct TranslationCache {
    cache: Arc<RwLock<HashMap<String, LayeredTranslations>>>,
}

#[derive(Debug, Default)]
struct LayeredTranslations {
    global: HashMap<String, String>,
    themes: HashMap<String, HashMap<String, String>>,
    snippets: HashMap<String, HashMap<String, String>>,
}

impl TranslationCache {
    pub fn new() -> Self {
        Self {
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Get translation from cache
    pub async fn get(&self, locale: &str, key: &str) -> Result<Option<String>> {
        let cache = self.cache.read().await;
        
        if let Some(translations) = cache.get(locale) {
            // Check all layers
            if let Some(value) = translations.global.get(key) {
                return Ok(Some(value.clone()));
            }
            
            // Check active theme
            // ... theme resolution logic ...
        }
        
        Ok(None)
    }
    
    /// Get from specific layer
    pub async fn get_from_layer(
        &self,
        locale: &str,
        key: &str,
        layer: &TranslationLayer,
    ) -> Result<Option<String>> {
        let cache = self.cache.read().await;
        
        if let Some(translations) = cache.get(locale) {
            match layer {
                TranslationLayer::Global => {
                    Ok(translations.global.get(key).cloned())
                }
                TranslationLayer::Theme(theme) => {
                    Ok(translations.themes
                        .get(theme)
                        .and_then(|t| t.get(key))
                        .cloned())
                }
                TranslationLayer::Snippet(snippet) => {
                    Ok(translations.snippets
                        .get(snippet)
                        .and_then(|t| t.get(key))
                        .cloned())
                }
            }
        } else {
            Ok(None)
        }
    }
    
    /// Store layer translations
    pub async fn store_layer(
        &self,
        locale: &str,
        layer: TranslationLayer,
        translations: HashMap<String, String>,
    ) -> Result<()> {
        let mut cache = self.cache.write().await;
        let entry = cache.entry(locale.to_string())
            .or_insert_with(LayeredTranslations::default);
        
        match layer {
            TranslationLayer::Global => {
                entry.global.extend(translations);
            }
            TranslationLayer::Theme(theme) => {
                entry.themes.entry(theme)
                    .or_insert_with(HashMap::new)
                    .extend(translations);
            }
            TranslationLayer::Snippet(snippet) => {
                entry.snippets.entry(snippet)
                    .or_insert_with(HashMap::new)
                    .extend(translations);
            }
        }
        
        Ok(())
    }
}
```

## Translation Export

```rust
impl TranslationManager {
    /// Export translations for JavaScript
    pub async fn export_for_frontend(
        &self,
        locale: &str,
        namespace: Option<&str>,
    ) -> Result<serde_json::Value> {
        let mut translations = serde_json::Map::new();
        
        // Get all translations for locale
        let cache = self.cache.cache.read().await;
        
        if let Some(locale_translations) = cache.get(locale) {
            // Export global translations
            for (key, value) in &locale_translations.global {
                if namespace.is_none() || key.starts_with(namespace.unwrap()) {
                    translations.insert(key.clone(), serde_json::Value::String(value.clone()));
                }
            }
            
            // Could also export theme/snippet translations based on context
        }
        
        Ok(serde_json::Value::Object(translations))
    }
    
    /// Generate translation catalog
    pub async fn generate_catalog(&self) -> Result<TranslationCatalog> {
        let cache = self.cache.cache.read().await;
        let mut catalog = TranslationCatalog::default();
        
        for (locale, translations) in cache.iter() {
            let mut locale_info = LocaleInfo {
                locale: locale.clone(),
                total_keys: 0,
                coverage: HashMap::new(),
            };
            
            // Count global keys
            locale_info.total_keys += translations.global.len();
            locale_info.coverage.insert(
                "global".to_string(),
                translations.global.len(),
            );
            
            // Count theme keys
            for (theme, theme_trans) in &translations.themes {
                locale_info.total_keys += theme_trans.len();
                locale_info.coverage.insert(
                    format!("theme.{}", theme),
                    theme_trans.len(),
                );
            }
            
            catalog.locales.push(locale_info);
        }
        
        Ok(catalog)
    }
}

#[derive(Debug, Default, Serialize)]
pub struct TranslationCatalog {
    pub locales: Vec<LocaleInfo>,
}

#[derive(Debug, Serialize)]
pub struct LocaleInfo {
    pub locale: String,
    pub total_keys: usize,
    pub coverage: HashMap<String, usize>,
}
```

## Helpers

```rust
impl TranslationManager {
    /// Extract locale from filename
    fn extract_locale_from_filename(&self, path: &Path) -> Result<String> {
        let filename = path.file_stem()
            .and_then(|s| s.to_str())
            .ok_or_else(|| ReedError::InvalidPath(path.display().to_string()))?;
        
        // Format: translations.de-CH.csv
        let parts: Vec<&str> = filename.split('.').collect();
        
        if parts.len() >= 2 {
            Ok(parts[1].to_string())
        } else {
            Ok(self.default_locale.clone())
        }
    }
    
    /// Extract theme name from path
    fn extract_theme_name(&self, path: &Path) -> Option<String> {
        path.components()
            .rev()
            .nth(2) // themes/{theme_name}/translations/...
            .and_then(|c| c.as_os_str().to_str())
            .map(|s| s.to_string())
    }
    
    /// Extract snippet type from path
    fn extract_snippet_type(&self, path: &Path) -> Option<String> {
        path.components()
            .rev()
            .nth(2) // snippets/{snippet_type}/translations/...
            .and_then(|c| c.as_os_str().to_str())
            .map(|s| s.to_string())
    }
    
    /// Check if translation exists
    pub async fn has_translation(&self, key: &str, locale: Option<&str>) -> bool {
        let locale = locale.unwrap_or(&self.default_locale);
        self.resolve_translation(key, locale)
            .await
            .map(|t| t != key)
            .unwrap_or(false)
    }
}
```

## Summary

Dieses Translation System bietet:
- **Multi-Layer Loading** - Global, Theme und Snippet Translations
- **Fallback Chain** - Locale-basierte Fallback Resolution
- **Plural Support** - Locale-spezifische Plural Rules
- **Interpolation** - Parameter Replacement mit Formatters
- **CSV-based** - Einfach editierbare Translation Files

Translation Layer macht ReedCMS truly multilingual.
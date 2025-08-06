# Tera Template Engine Setup

## Overview

Tera Template Engine Integration für ReedCMS. Basis für das gesamte Rendering System mit Custom Functions und EPC Support.

## Tera Engine Configuration

```rust
use tera::{Tera, Context, Value};
use std::sync::Arc;
use tokio::sync::RwLock;

/// Tera engine wrapper with ReedCMS extensions
pub struct TeraEngine {
    tera: Arc<RwLock<Tera>>,
    epc_resolver: Arc<EpcResolver>,
    snippet_manager: Arc<SnippetManager>,
    translation_manager: Arc<TranslationManager>,
}

impl TeraEngine {
    pub fn new(
        base_path: PathBuf,
        epc_resolver: Arc<EpcResolver>,
        snippet_manager: Arc<SnippetManager>,
        translation_manager: Arc<TranslationManager>,
    ) -> Result<Self> {
        let mut tera = Tera::new(&format!("{}/**/*.tera", base_path.display()))?;
        
        // Disable auto-escaping for maximum control
        tera.autoescape_on(vec![]);
        
        // Register custom filters
        Self::register_filters(&mut tera);
        
        // Register custom functions
        Self::register_functions(&mut tera);
        
        // Register custom testers
        Self::register_testers(&mut tera);
        
        Ok(Self {
            tera: Arc::new(RwLock::new(tera)),
            epc_resolver,
            snippet_manager,
            translation_manager,
        })
    }
    
    /// Render template with context
    pub async fn render(
        &self,
        template_name: &str,
        context: &Context,
    ) -> Result<String> {
        let tera = self.tera.read().await;
        
        // Resolve template path via EPC
        let resolved_path = self.epc_resolver
            .resolve_template(template_name)
            .await?;
        
        // Render with error context
        tera.render(&resolved_path, context)
            .map_err(|e| ReedError::Template(TemplateError::RenderError {
                template: template_name.to_string(),
                source: e,
            }))
    }
    
    /// Render string template (for inline templates)
    pub async fn render_string(
        &self,
        template: &str,
        context: &Context,
    ) -> Result<String> {
        let mut tera = Tera::default();
        
        // Copy functions and filters
        let engine = self.tera.read().await;
        Self::copy_extensions(&engine, &mut tera);
        
        // Add template
        tera.add_raw_template("inline", template)?;
        
        // Render
        tera.render("inline", context)
            .map_err(|e| ReedError::Template(TemplateError::RenderError {
                template: "inline".to_string(),
                source: e,
            }))
    }
}
```

## Custom Filters

```rust
impl TeraEngine {
    fn register_filters(tera: &mut Tera) {
        // String manipulation
        tera.register_filter("slugify", filters::slugify);
        tera.register_filter("truncate_words", filters::truncate_words);
        tera.register_filter("highlight", filters::highlight);
        
        // Date formatting
        tera.register_filter("date", filters::date);
        tera.register_filter("relative_time", filters::relative_time);
        
        // Number formatting
        tera.register_filter("number_format", filters::number_format);
        tera.register_filter("file_size", filters::file_size);
        
        // URL manipulation
        tera.register_filter("absolute_url", filters::absolute_url);
        tera.register_filter("asset_url", filters::asset_url);
        
        // Content processing
        tera.register_filter("markdown", filters::markdown);
        tera.register_filter("strip_html", filters::strip_html);
        tera.register_filter("excerpt", filters::excerpt);
    }
}

mod filters {
    use tera::{Value, Result as TeraResult, Error as TeraError};
    use std::collections::HashMap;
    
    pub fn slugify(value: &Value, _: &HashMap<String, Value>) -> TeraResult<Value> {
        let s = value.as_str()
            .ok_or_else(|| TeraError::msg("slugify requires string"))?;
        
        let slug = s.to_lowercase()
            .chars()
            .map(|c| if c.is_alphanumeric() { c } else { '-' })
            .collect::<String>()
            .split('-')
            .filter(|s| !s.is_empty())
            .collect::<Vec<_>>()
            .join("-");
        
        Ok(Value::String(slug))
    }
    
    pub fn truncate_words(value: &Value, args: &HashMap<String, Value>) -> TeraResult<Value> {
        let text = value.as_str()
            .ok_or_else(|| TeraError::msg("truncate_words requires string"))?;
        
        let word_count = args.get("words")
            .and_then(|v| v.as_u64())
            .unwrap_or(50) as usize;
        
        let suffix = args.get("suffix")
            .and_then(|v| v.as_str())
            .unwrap_or("...");
        
        let words: Vec<&str> = text.split_whitespace().collect();
        
        if words.len() <= word_count {
            Ok(Value::String(text.to_string()))
        } else {
            let truncated = words[..word_count].join(" ");
            Ok(Value::String(format!("{}{}", truncated, suffix)))
        }
    }
    
    pub fn date(value: &Value, args: &HashMap<String, Value>) -> TeraResult<Value> {
        let format = args.get("format")
            .and_then(|v| v.as_str())
            .unwrap_or("%Y-%m-%d");
        
        // Parse various date inputs
        let date_str = if let Some(s) = value.as_str() {
            chrono::DateTime::parse_from_rfc3339(s)
                .map(|dt| dt.format(format).to_string())
                .unwrap_or_else(|_| s.to_string())
        } else if let Some(timestamp) = value.as_i64() {
            chrono::NaiveDateTime::from_timestamp_opt(timestamp, 0)
                .map(|dt| dt.format(format).to_string())
                .unwrap_or_else(|| timestamp.to_string())
        } else {
            return Err(TeraError::msg("date requires string or timestamp"));
        };
        
        Ok(Value::String(date_str))
    }
    
    pub fn markdown(value: &Value, _: &HashMap<String, Value>) -> TeraResult<Value> {
        let text = value.as_str()
            .ok_or_else(|| TeraError::msg("markdown requires string"))?;
        
        // Simple markdown to HTML (use pulldown-cmark in real implementation)
        let html = text
            .replace("**", "<strong>")
            .replace("*", "<em>")
            .replace("\n\n", "</p><p>")
            .replace("\n", "<br>");
        
        Ok(Value::String(format!("<p>{}</p>", html)))
    }
}
```

## Custom Functions

```rust
impl TeraEngine {
    fn register_functions(tera: &mut Tera) {
        // Content functions
        tera.register_function("snippet", functions::snippet);
        tera.register_function("render_snippet", functions::render_snippet);
        tera.register_function("include_snippet", functions::include_snippet);
        
        // Translation functions
        tera.register_function("t", functions::translate);
        tera.register_function("tn", functions::translate_plural);
        
        // Asset functions
        tera.register_function("asset", functions::asset);
        tera.register_function("image", functions::image);
        
        // URL functions
        tera.register_function("url", functions::url);
        tera.register_function("route", functions::route);
        
        // Utility functions
        tera.register_function("debug", functions::debug);
        tera.register_function("dump", functions::dump);
    }
}

mod functions {
    use tera::{Function, Value, Result as TeraResult, Error as TeraError};
    use std::collections::HashMap;
    
    /// Get snippet by ID or path
    pub fn snippet(args: &HashMap<String, Value>) -> TeraResult<Value> {
        let id = args.get("id")
            .and_then(|v| v.as_str())
            .ok_or_else(|| TeraError::msg("snippet() requires 'id' parameter"))?;
        
        // This would be injected via context in real implementation
        // For now, return placeholder
        Ok(Value::Object(serde_json::Map::new()))
    }
    
    /// Render snippet with data
    pub fn render_snippet(args: &HashMap<String, Value>) -> TeraResult<Value> {
        let snippet_type = args.get("type")
            .and_then(|v| v.as_str())
            .ok_or_else(|| TeraError::msg("render_snippet() requires 'type'"))?;
        
        let data = args.get("data")
            .cloned()
            .unwrap_or(Value::Object(serde_json::Map::new()));
        
        // Would call snippet renderer in real implementation
        Ok(Value::String(format!(
            "<!-- snippet: {} -->",
            snippet_type
        )))
    }
    
    /// Translate string
    pub fn translate(args: &HashMap<String, Value>) -> TeraResult<Value> {
        let key = args.get("key")
            .and_then(|v| v.as_str())
            .ok_or_else(|| TeraError::msg("t() requires 'key'"))?;
        
        let params = args.get("params")
            .and_then(|v| v.as_object())
            .cloned()
            .unwrap_or_default();
        
        // Would call translation manager in real implementation
        Ok(Value::String(key.to_string()))
    }
    
    /// Get asset URL with cache busting
    pub fn asset(args: &HashMap<String, Value>) -> TeraResult<Value> {
        let path = args.get("path")
            .and_then(|v| v.as_str())
            .ok_or_else(|| TeraError::msg("asset() requires 'path'"))?;
        
        // Add cache buster in real implementation
        Ok(Value::String(format!("/assets/{}", path)))
    }
}
```

## Custom Testers

```rust
impl TeraEngine {
    fn register_testers(tera: &mut Tera) {
        // Type testers
        tera.register_tester("empty", testers::is_empty);
        tera.register_tester("snippet", testers::is_snippet);
        tera.register_tester("routable", testers::is_routable);
        
        // State testers
        tera.register_tester("published", testers::is_published);
        tera.register_tester("draft", testers::is_draft);
        
        // Permission testers
        tera.register_tester("editable", testers::is_editable);
        tera.register_tester("deletable", testers::is_deletable);
    }
}

mod testers {
    use tera::{Value, Result as TeraResult};
    
    pub fn is_empty(value: Option<&Value>, _: &[Value]) -> TeraResult<bool> {
        Ok(match value {
            None => true,
            Some(Value::Null) => true,
            Some(Value::String(s)) => s.is_empty(),
            Some(Value::Array(a)) => a.is_empty(),
            Some(Value::Object(o)) => o.is_empty(),
            _ => false,
        })
    }
    
    pub fn is_snippet(value: Option<&Value>, _: &[Value]) -> TeraResult<bool> {
        Ok(value
            .and_then(|v| v.as_object())
            .and_then(|o| o.get("_type"))
            .and_then(|t| t.as_str())
            .map(|t| t == "snippet")
            .unwrap_or(false))
    }
    
    pub fn is_routable(value: Option<&Value>, _: &[Value]) -> TeraResult<bool> {
        Ok(value
            .and_then(|v| v.as_object())
            .and_then(|o| o.get("_meta"))
            .and_then(|m| m.as_object())
            .and_then(|m| m.get("is_routable"))
            .and_then(|r| r.as_bool())
            .unwrap_or(false))
    }
}
```

## Template Loader with EPC

```rust
/// Custom template loader that uses EPC resolution
pub struct EpcTemplateLoader {
    epc_resolver: Arc<EpcResolver>,
    cache: Arc<RwLock<HashMap<String, String>>>,
}

impl EpcTemplateLoader {
    pub fn new(epc_resolver: Arc<EpcResolver>) -> Self {
        Self {
            epc_resolver,
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Load template via EPC
    pub async fn load_template(&self, name: &str) -> Result<String> {
        // Check cache
        {
            let cache = self.cache.read().await;
            if let Some(content) = cache.get(name) {
                return Ok(content.clone());
            }
        }
        
        // Resolve via EPC
        let path = self.epc_resolver.resolve_template(name).await?;
        let content = tokio::fs::read_to_string(&path).await?;
        
        // Cache for development
        #[cfg(debug_assertions)]
        {
            let mut cache = self.cache.write().await;
            cache.insert(name.to_string(), content.clone());
        }
        
        Ok(content)
    }
    
    /// Clear template cache (for hot reload)
    pub async fn clear_cache(&self) {
        let mut cache = self.cache.write().await;
        cache.clear();
    }
}
```

## Error Handling

```rust
#[derive(Debug, thiserror::Error)]
pub enum TemplateError {
    #[error("Template not found: {template}")]
    NotFound { template: String },
    
    #[error("Template syntax error in {template}: {message}")]
    SyntaxError { template: String, message: String },
    
    #[error("Render error in {template}")]
    RenderError {
        template: String,
        #[source]
        source: tera::Error,
    },
    
    #[error("Function error: {function} - {message}")]
    FunctionError { function: String, message: String },
}

/// Template debugging helper
pub struct TemplateDebugger {
    templates: HashMap<String, TemplateDebugInfo>,
}

#[derive(Debug)]
pub struct TemplateDebugInfo {
    pub render_count: u64,
    pub avg_render_ms: f64,
    pub last_error: Option<String>,
    pub dependencies: Vec<String>,
}

impl TemplateDebugger {
    pub fn track_render(&mut self, template: &str, duration_ms: f64) {
        let info = self.templates.entry(template.to_string())
            .or_insert_with(|| TemplateDebugInfo {
                render_count: 0,
                avg_render_ms: 0.0,
                last_error: None,
                dependencies: vec![],
            });
        
        info.render_count += 1;
        info.avg_render_ms = (info.avg_render_ms * (info.render_count - 1) as f64
            + duration_ms) / info.render_count as f64;
    }
}
```

## Summary

Dieses Tera Setup bietet:
- **Custom Functions** - snippet(), render_snippet(), t() für ReedCMS
- **EPC Integration** - Template Resolution via Theme Chain
- **Performance Tracking** - Debug Info für Template Rendering
- **Flexible Filters** - Markdown, Slugify, Date Formatting
- **Error Context** - Detaillierte Fehlerinfos für Development

Tera als Template Engine passt perfekt zu ReedCMS's KISS Philosophy.
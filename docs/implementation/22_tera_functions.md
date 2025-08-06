# Tera Custom Functions

## Overview

Custom Tera Functions für ReedCMS. Diese Functions ermöglichen Snippet-Rendering, Translations und Asset-Management direkt in Templates.

## Function Registry

```rust
use tera::{Function, Value, Context, Error as TeraError};
use std::collections::HashMap;
use std::sync::Arc;

/// Registry for all custom Tera functions
pub struct TeraFunctionRegistry {
    snippet_manager: Arc<SnippetManager>,
    translation_manager: Arc<TranslationManager>,
    asset_manager: Arc<AssetManager>,
    ucg_query: Arc<UcgQuery>,
}

impl TeraFunctionRegistry {
    pub fn new(
        snippet_manager: Arc<SnippetManager>,
        translation_manager: Arc<TranslationManager>,
        asset_manager: Arc<AssetManager>,
        ucg_query: Arc<UcgQuery>,
    ) -> Self {
        Self {
            snippet_manager,
            translation_manager,
            asset_manager,
            ucg_query,
        }
    }
    
    /// Register all functions with Tera
    pub fn register_all(&self, tera: &mut Tera) {
        // Snippet functions
        tera.register_function("snippet", self.make_snippet_function());
        tera.register_function("render_snippet", self.make_render_snippet_function());
        tera.register_function("snippets", self.make_snippets_query_function());
        
        // Translation functions
        tera.register_function("t", self.make_translate_function());
        tera.register_function("tn", self.make_translate_plural_function());
        tera.register_function("has_translation", self.make_has_translation_function());
        
        // Asset functions
        tera.register_function("asset", self.make_asset_function());
        tera.register_function("image", self.make_image_function());
        tera.register_function("inline_svg", self.make_inline_svg_function());
        
        // UCG query functions
        tera.register_function("children", self.make_children_function());
        tera.register_function("parent", self.make_parent_function());
        tera.register_function("ancestors", self.make_ancestors_function());
        
        // Utility functions
        tera.register_function("json", self.make_json_function());
        tera.register_function("debug", self.make_debug_function());
        tera.register_function("env", self.make_env_function());
    }
}
```

## Snippet Functions

```rust
impl TeraFunctionRegistry {
    /// Get snippet by ID or semantic name
    fn make_snippet_function(&self) -> impl Function {
        let manager = self.snippet_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let id_or_name = args.get("id")
                .or_else(|| args.get("name"))
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("snippet() requires 'id' or 'name'"))?;
            
            // Block on async operation (Tera is sync)
            let entity = tokio::runtime::Handle::current()
                .block_on(async {
                    if let Ok(uuid) = Uuid::parse_str(id_or_name) {
                        manager.get_by_id(uuid).await
                    } else {
                        manager.get_by_name(id_or_name).await
                    }
                })
                .map_err(|e| TeraError::msg(format!("Failed to load snippet: {}", e)))?;
            
            // Convert entity to Value
            let value = serde_json::to_value(&entity)
                .map_err(|e| TeraError::msg(format!("Serialization error: {}", e)))?;
            
            Ok(value)
        })
    }
    
    /// Render snippet with provided data
    fn make_render_snippet_function(&self) -> impl Function {
        let manager = self.snippet_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let snippet_type = args.get("type")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("render_snippet() requires 'type'"))?;
            
            let data = args.get("data")
                .cloned()
                .unwrap_or(Value::Object(serde_json::Map::new()));
            
            let template = args.get("template")
                .and_then(|v| v.as_str());
            
            // Render snippet
            let html = tokio::runtime::Handle::current()
                .block_on(async {
                    manager.render_snippet(snippet_type, data, template).await
                })
                .map_err(|e| TeraError::msg(format!("Render failed: {}", e)))?;
            
            Ok(Value::String(html))
        })
    }
    
    /// Query snippets by type and filters
    fn make_snippets_query_function(&self) -> impl Function {
        let ucg = self.ucg_query.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let snippet_type = args.get("type")
                .and_then(|v| v.as_str());
            
            let parent_id = args.get("parent")
                .and_then(|v| v.as_str())
                .and_then(|s| Uuid::parse_str(s).ok());
            
            let limit = args.get("limit")
                .and_then(|v| v.as_u64())
                .unwrap_or(100) as usize;
            
            let order_by = args.get("order_by")
                .and_then(|v| v.as_str())
                .unwrap_or("weight");
            
            // Build query
            let mut query = UcgQueryBuilder::new();
            
            if let Some(st) = snippet_type {
                query = query.filter_by_type(st);
            }
            
            if let Some(pid) = parent_id {
                query = query.filter_by_parent(pid);
            }
            
            query = query.order_by(order_by).limit(limit);
            
            // Execute query
            let entities = tokio::runtime::Handle::current()
                .block_on(async {
                    ucg.execute_query(query.build()).await
                })
                .map_err(|e| TeraError::msg(format!("Query failed: {}", e)))?;
            
            // Convert to Value
            let value = serde_json::to_value(&entities)
                .map_err(|e| TeraError::msg(format!("Serialization error: {}", e)))?;
            
            Ok(value)
        })
    }
}
```

## Translation Functions

```rust
impl TeraFunctionRegistry {
    /// Translate key with parameters
    fn make_translate_function(&self) -> impl Function {
        let manager = self.translation_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let key = args.get("key")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("t() requires 'key'"))?;
            
            let locale = args.get("locale")
                .and_then(|v| v.as_str());
            
            let params = args.get("params")
                .and_then(|v| v.as_object())
                .map(|obj| {
                    obj.iter()
                        .filter_map(|(k, v)| v.as_str().map(|s| (k.clone(), s.to_string())))
                        .collect::<HashMap<_, _>>()
                })
                .unwrap_or_default();
            
            // Get translation
            let translated = tokio::runtime::Handle::current()
                .block_on(async {
                    manager.translate(key, locale, &params).await
                })
                .unwrap_or_else(|_| key.to_string());
            
            Ok(Value::String(translated))
        })
    }
    
    /// Translate with plural forms
    fn make_translate_plural_function(&self) -> impl Function {
        let manager = self.translation_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let key = args.get("key")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("tn() requires 'key'"))?;
            
            let count = args.get("count")
                .and_then(|v| v.as_u64())
                .ok_or_else(|| TeraError::msg("tn() requires 'count'"))? as usize;
            
            let locale = args.get("locale")
                .and_then(|v| v.as_str());
            
            // Get plural form
            let translated = tokio::runtime::Handle::current()
                .block_on(async {
                    manager.translate_plural(key, count, locale).await
                })
                .unwrap_or_else(|_| format!("{} ({})", key, count));
            
            Ok(Value::String(translated))
        })
    }
    
    /// Check if translation exists
    fn make_has_translation_function(&self) -> impl Function {
        let manager = self.translation_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let key = args.get("key")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("has_translation() requires 'key'"))?;
            
            let locale = args.get("locale")
                .and_then(|v| v.as_str());
            
            let exists = tokio::runtime::Handle::current()
                .block_on(async {
                    manager.has_translation(key, locale).await
                });
            
            Ok(Value::Bool(exists))
        })
    }
}
```

## Asset Functions

```rust
impl TeraFunctionRegistry {
    /// Get asset URL with cache busting
    fn make_asset_function(&self) -> impl Function {
        let manager = self.asset_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let path = args.get("path")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("asset() requires 'path'"))?;
            
            let bust_cache = args.get("bust_cache")
                .and_then(|v| v.as_bool())
                .unwrap_or(true);
            
            let url = if bust_cache {
                manager.get_url_with_hash(path)
            } else {
                manager.get_url(path)
            };
            
            Ok(Value::String(url))
        })
    }
    
    /// Generate responsive image tag
    fn make_image_function(&self) -> impl Function {
        let manager = self.asset_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let src = args.get("src")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("image() requires 'src'"))?;
            
            let alt = args.get("alt")
                .and_then(|v| v.as_str())
                .unwrap_or("");
            
            let sizes = args.get("sizes")
                .and_then(|v| v.as_array())
                .map(|arr| {
                    arr.iter()
                        .filter_map(|v| v.as_u64())
                        .map(|n| n as u32)
                        .collect::<Vec<_>>()
                })
                .unwrap_or_else(|| vec![320, 640, 1024, 1920]);
            
            let lazy = args.get("lazy")
                .and_then(|v| v.as_bool())
                .unwrap_or(true);
            
            // Generate responsive image HTML
            let srcset = sizes.iter()
                .map(|&size| {
                    let url = manager.get_image_url(src, size);
                    format!("{} {}w", url, size)
                })
                .collect::<Vec<_>>()
                .join(", ");
            
            let loading = if lazy { "lazy" } else { "eager" };
            
            let html = format!(
                r#"<img src="{}" srcset="{}" alt="{}" loading="{}" />"#,
                manager.get_url(src),
                srcset,
                alt,
                loading
            );
            
            Ok(Value::String(html))
        })
    }
    
    /// Inline SVG content
    fn make_inline_svg_function(&self) -> impl Function {
        let manager = self.asset_manager.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let path = args.get("path")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("inline_svg() requires 'path'"))?;
            
            let class = args.get("class")
                .and_then(|v| v.as_str());
            
            // Load SVG content
            let svg_content = tokio::runtime::Handle::current()
                .block_on(async {
                    manager.load_svg(path).await
                })
                .map_err(|e| TeraError::msg(format!("Failed to load SVG: {}", e)))?;
            
            // Add class if provided
            let svg = if let Some(cls) = class {
                svg_content.replace("<svg", &format!(r#"<svg class="{}""#, cls))
            } else {
                svg_content
            };
            
            Ok(Value::String(svg))
        })
    }
}
```

## UCG Query Functions

```rust
impl TeraFunctionRegistry {
    /// Get children of entity
    fn make_children_function(&self) -> impl Function {
        let ucg = self.ucg_query.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let parent_id = args.get("parent")
                .and_then(|v| v.as_str())
                .and_then(|s| Uuid::parse_str(s).ok())
                .ok_or_else(|| TeraError::msg("children() requires valid 'parent' UUID"))?;
            
            let entity_type = args.get("type")
                .and_then(|v| v.as_str());
            
            let children = tokio::runtime::Handle::current()
                .block_on(async {
                    ucg.get_children(parent_id, entity_type).await
                })
                .map_err(|e| TeraError::msg(format!("Query failed: {}", e)))?;
            
            let value = serde_json::to_value(&children)
                .map_err(|e| TeraError::msg(format!("Serialization error: {}", e)))?;
            
            Ok(value)
        })
    }
    
    /// Get parent of entity
    fn make_parent_function(&self) -> impl Function {
        let ucg = self.ucg_query.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let entity_id = args.get("entity")
                .and_then(|v| v.as_str())
                .and_then(|s| Uuid::parse_str(s).ok())
                .ok_or_else(|| TeraError::msg("parent() requires valid 'entity' UUID"))?;
            
            let parent = tokio::runtime::Handle::current()
                .block_on(async {
                    ucg.get_parent(entity_id).await
                })
                .map_err(|e| TeraError::msg(format!("Query failed: {}", e)))?;
            
            match parent {
                Some(p) => {
                    let value = serde_json::to_value(&p)
                        .map_err(|e| TeraError::msg(format!("Serialization error: {}", e)))?;
                    Ok(value)
                }
                None => Ok(Value::Null),
            }
        })
    }
    
    /// Get ancestor chain
    fn make_ancestors_function(&self) -> impl Function {
        let ucg = self.ucg_query.clone();
        
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let entity_id = args.get("entity")
                .and_then(|v| v.as_str())
                .and_then(|s| Uuid::parse_str(s).ok())
                .ok_or_else(|| TeraError::msg("ancestors() requires valid 'entity' UUID"))?;
            
            let include_self = args.get("include_self")
                .and_then(|v| v.as_bool())
                .unwrap_or(false);
            
            let ancestors = tokio::runtime::Handle::current()
                .block_on(async {
                    ucg.get_ancestors(entity_id, include_self).await
                })
                .map_err(|e| TeraError::msg(format!("Query failed: {}", e)))?;
            
            let value = serde_json::to_value(&ancestors)
                .map_err(|e| TeraError::msg(format!("Serialization error: {}", e)))?;
            
            Ok(value)
        })
    }
}
```

## Utility Functions

```rust
impl TeraFunctionRegistry {
    /// Convert value to JSON string
    fn make_json_function(&self) -> impl Function {
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let value = args.get("value")
                .ok_or_else(|| TeraError::msg("json() requires 'value'"))?;
            
            let pretty = args.get("pretty")
                .and_then(|v| v.as_bool())
                .unwrap_or(false);
            
            let json = if pretty {
                serde_json::to_string_pretty(value)
            } else {
                serde_json::to_string(value)
            }
            .map_err(|e| TeraError::msg(format!("JSON error: {}", e)))?;
            
            Ok(Value::String(json))
        })
    }
    
    /// Debug dump value
    fn make_debug_function(&self) -> impl Function {
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let value = args.get("value")
                .cloned()
                .unwrap_or(Value::Null);
            
            let label = args.get("label")
                .and_then(|v| v.as_str())
                .unwrap_or("Debug");
            
            let output = format!(
                r#"<pre class="debug"><strong>{}</strong>: {}</pre>"#,
                label,
                serde_json::to_string_pretty(&value).unwrap_or_else(|_| "ERROR".to_string())
            );
            
            Ok(Value::String(output))
        })
    }
    
    /// Get environment variable
    fn make_env_function(&self) -> impl Function {
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            let key = args.get("key")
                .and_then(|v| v.as_str())
                .ok_or_else(|| TeraError::msg("env() requires 'key'"))?;
            
            let default = args.get("default")
                .and_then(|v| v.as_str());
            
            // Only allow safe env vars
            let allowed_prefixes = ["REED_", "PUBLIC_"];
            let is_allowed = allowed_prefixes.iter()
                .any(|prefix| key.starts_with(prefix));
            
            if !is_allowed {
                return Err(TeraError::msg(format!(
                    "env() can only access variables starting with: {:?}",
                    allowed_prefixes
                )));
            }
            
            let value = std::env::var(key)
                .unwrap_or_else(|_| default.unwrap_or("").to_string());
            
            Ok(Value::String(value))
        })
    }
}
```

## Function Context Injection

```rust
/// Injects ReedCMS context into Tera functions
pub struct FunctionContextInjector {
    request_context: RequestContext,
    user_context: UserContext,
}

impl FunctionContextInjector {
    /// Create Tera context with injected values
    pub fn create_context(&self) -> Context {
        let mut ctx = Context::new();
        
        // Inject request data
        ctx.insert("request", &self.request_context);
        ctx.insert("user", &self.user_context);
        
        // Inject current theme
        ctx.insert("theme", &self.request_context.theme);
        
        // Inject current locale
        ctx.insert("locale", &self.request_context.locale);
        
        ctx
    }
    
    /// Wrap function with context injection
    pub fn wrap_function<F>(function: F) -> impl Function
    where
        F: Function + 'static,
    {
        Box::new(move |args: &HashMap<String, Value>| -> Result<Value, TeraError> {
            // Functions can access injected context via args
            function(args)
        })
    }
}
```

## Summary

Diese Custom Functions bieten:
- **Snippet Integration** - Direkter Zugriff auf UCG Content
- **Translation Support** - Multi-locale mit Pluralization
- **Asset Management** - Cache-busting und Responsive Images
- **UCG Queries** - Navigation und Relationships
- **Developer Tools** - Debug und JSON Output

Functions sind der Schlüssel zu ReedCMS's Template Power.
# Plugin Examples

## Overview

Example Plugin Implementations für ReedCMS. Praktische Beispiele für verschiedene Plugin Types und Use Cases.

## Simple Content Extension Plugin

```rust
// Cargo.toml
[package]
name = "reed-gallery-plugin"
version = "0.1.0"
edition = "2021"

[dependencies]
reed-plugin-api = "1.0"
async-trait = "0.1"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

[lib]
crate-type = ["cdylib"]

// src/lib.rs
use reed_plugin_api::*;
use async_trait::async_trait;
use serde::{Serialize, Deserialize};

/// Gallery plugin that adds image gallery functionality
pub struct GalleryPlugin {
    api: Option<PluginApi>,
}

impl Default for GalleryPlugin {
    fn default() -> Self {
        Self { api: None }
    }
}

#[async_trait]
impl ReedPlugin for GalleryPlugin {
    async fn init(&mut self, api: PluginApi) -> PluginResult<()> {
        // Store API reference
        self.api = Some(api.clone());
        
        // Register content type
        self.register_gallery_type(&api).await?;
        
        // Register hooks
        self.register_hooks(&api).await?;
        
        // Register template functions
        self.register_template_functions(&api).await?;
        
        api.logger().info("Gallery plugin initialized");
        Ok(())
    }
    
    async fn activate(&mut self) -> PluginResult<()> {
        if let Some(ref api) = self.api {
            api.logger().info("Gallery plugin activated");
        }
        Ok(())
    }
    
    async fn deactivate(&mut self) -> PluginResult<()> {
        if let Some(ref api) = self.api {
            api.logger().info("Gallery plugin deactivated");
        }
        Ok(())
    }
    
    fn info(&self) -> PluginInfo {
        PluginInfo {
            name: "Gallery Plugin".to_string(),
            version: "0.1.0".to_string(),
            author: "ReedCMS Team".to_string(),
            description: "Adds image gallery functionality to ReedCMS".to_string(),
            license: "MIT".to_string(),
            homepage: Some("https://example.com/gallery-plugin".to_string()),
            repository: Some("https://github.com/example/gallery-plugin".to_string()),
            keywords: vec!["gallery".to_string(), "images".to_string()],
            categories: vec!["media".to_string()],
            api_version: "1.0.0".to_string(),
        }
    }
}

impl GalleryPlugin {
    /// Register gallery content type
    async fn register_gallery_type(&self, api: &PluginApi) -> PluginResult<()> {
        // Define gallery schema
        let schema = json!({
            "name": "gallery",
            "display_name": "Image Gallery",
            "fields": [
                {
                    "name": "title",
                    "type": "text",
                    "required": true,
                    "display_name": "Gallery Title"
                },
                {
                    "name": "description",
                    "type": "textarea",
                    "required": false,
                    "display_name": "Description"
                },
                {
                    "name": "images",
                    "type": "array",
                    "required": true,
                    "display_name": "Images",
                    "item_type": {
                        "type": "object",
                        "fields": [
                            {
                                "name": "url",
                                "type": "url",
                                "required": true
                            },
                            {
                                "name": "caption",
                                "type": "text",
                                "required": false
                            },
                            {
                                "name": "alt_text",
                                "type": "text",
                                "required": true
                            }
                        ]
                    }
                },
                {
                    "name": "layout",
                    "type": "select",
                    "required": true,
                    "default": "grid",
                    "options": [
                        {"value": "grid", "label": "Grid Layout"},
                        {"value": "masonry", "label": "Masonry Layout"},
                        {"value": "carousel", "label": "Carousel"}
                    ]
                }
            ]
        });
        
        // Store schema in plugin storage
        api.storage().put("schema:gallery", schema).await?;
        
        Ok(())
    }
    
    /// Register hooks
    async fn register_hooks(&self, api: &PluginApi) -> PluginResult<()> {
        // Before render hook to process gallery data
        api.hooks().register(
            "before_render",
            100,
            |data: RenderData| async move {
                // Process gallery content
                if data.content_type == "gallery" {
                    // Add gallery-specific data
                    let mut processed = data.clone();
                    processed.context.insert(
                        "gallery_assets".to_string(),
                        json!({
                            "css": ["/plugins/gallery/gallery.css"],
                            "js": ["/plugins/gallery/gallery.js"]
                        })
                    );
                    Ok(processed)
                } else {
                    Ok(data)
                }
            }
        ).await?;
        
        // Validation hook
        api.hooks().register(
            "validate_data",
            50,
            |data: ValidationData| async move {
                if data.content_type == "gallery" {
                    // Validate gallery data
                    if let Some(images) = data.fields.get("images") {
                        if let Some(arr) = images.as_array() {
                            if arr.is_empty() {
                                return Ok(ValidationResult {
                                    valid: false,
                                    errors: vec![ValidationError {
                                        field: "images".to_string(),
                                        message: "Gallery must contain at least one image".to_string(),
                                    }],
                                });
                            }
                        }
                    }
                }
                Ok(ValidationResult {
                    valid: true,
                    errors: vec![],
                })
            }
        ).await?;
        
        Ok(())
    }
    
    /// Register template functions
    async fn register_template_functions(&self, api: &PluginApi) -> PluginResult<()> {
        // Gallery renderer function
        api.templates().register_function(
            "render_gallery",
            |args: TemplateFunctionArgs| async move {
                let gallery_id = args.get("id")
                    .and_then(|v| v.as_str())
                    .ok_or_else(|| PluginError::InvalidArgument(
                        "Gallery ID required".to_string()
                    ))?;
                
                // Generate gallery HTML
                let html = format!(
                    r#"<div class="reed-gallery" data-gallery-id="{}">
                        <!-- Gallery content will be rendered by JavaScript -->
                    </div>"#,
                    gallery_id
                );
                
                Ok(json!(html))
            }
        ).await?;
        
        Ok(())
    }
}

// Export plugin
reed_plugin!(GalleryPlugin);
```

## WASM Plugin Example

```rust
// WASM plugin with restricted capabilities
// Cargo.toml
[package]
name = "reed-markdown-plugin"
version = "0.1.0"
edition = "2021"

[dependencies]
reed-plugin-api = "1.0"
wasm-bindgen = "0.2"
pulldown-cmark = "0.9"

[lib]
crate-type = ["cdylib"]

// src/lib.rs
use reed_plugin_api::*;
use wasm_bindgen::prelude::*;
use pulldown_cmark::{Parser, Options, html};

#[wasm_bindgen]
pub struct MarkdownPlugin {
    initialized: bool,
}

#[wasm_bindgen]
impl MarkdownPlugin {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        Self {
            initialized: false,
        }
    }
    
    /// Initialize plugin
    #[wasm_bindgen]
    pub fn initialize(&mut self, config: JsValue) -> Result<(), JsValue> {
        // Parse configuration
        let config: PluginConfig = config.into_serde().unwrap();
        
        self.initialized = true;
        
        // Log initialization
        web_sys::console::log_1(&"Markdown plugin initialized".into());
        
        Ok(())
    }
    
    /// Process markdown content
    #[wasm_bindgen]
    pub fn process_markdown(&self, content: &str, options: JsValue) -> Result<String, JsValue> {
        if !self.initialized {
            return Err("Plugin not initialized".into());
        }
        
        // Parse options
        let opts: MarkdownOptions = options.into_serde().unwrap_or_default();
        
        // Setup parser options
        let mut parser_options = Options::empty();
        if opts.enable_tables {
            parser_options.insert(Options::ENABLE_TABLES);
        }
        if opts.enable_footnotes {
            parser_options.insert(Options::ENABLE_FOOTNOTES);
        }
        if opts.enable_strikethrough {
            parser_options.insert(Options::ENABLE_STRIKETHROUGH);
        }
        
        // Parse markdown
        let parser = Parser::new_ext(content, parser_options);
        
        // Convert to HTML
        let mut html_output = String::new();
        html::push_html(&mut html_output, parser);
        
        // Post-process if needed
        if opts.add_anchors {
            html_output = self.add_heading_anchors(&html_output);
        }
        
        Ok(html_output)
    }
    
    /// Add anchors to headings
    fn add_heading_anchors(&self, html: &str) -> String {
        // Simple regex-based anchor addition
        let heading_regex = regex::Regex::new(r"<h([1-6])>([^<]+)</h[1-6]>").unwrap();
        
        heading_regex.replace_all(html, |caps: &regex::Captures| {
            let level = &caps[1];
            let text = &caps[2];
            let anchor = text.to_lowercase().replace(' ', '-');
            
            format!(
                r#"<h{} id="{}">{}<a href="#{}" class="anchor">#</a></h{}>"#,
                level, anchor, text, anchor, level
            )
        }).to_string()
    }
}

#[derive(Default, Deserialize)]
struct MarkdownOptions {
    enable_tables: bool,
    enable_footnotes: bool,
    enable_strikethrough: bool,
    add_anchors: bool,
}

#[derive(Deserialize)]
struct PluginConfig {
    // Plugin configuration
}
```

## Event-Driven Plugin

```rust
/// SEO optimization plugin
use reed_plugin_api::*;
use async_trait::async_trait;

pub struct SeoPlugin {
    api: Option<PluginApi>,
    sitemap_generator: SitemapGenerator,
}

#[async_trait]
impl ReedPlugin for SeoPlugin {
    async fn init(&mut self, api: PluginApi) -> PluginResult<()> {
        self.api = Some(api.clone());
        
        // Subscribe to content events
        self.subscribe_to_events(&api).await?;
        
        // Register SEO hooks
        self.register_seo_hooks(&api).await?;
        
        // Schedule sitemap generation
        self.schedule_sitemap_generation(&api).await?;
        
        Ok(())
    }
    
    fn info(&self) -> PluginInfo {
        PluginInfo {
            name: "SEO Plugin".to_string(),
            version: "0.1.0".to_string(),
            author: "ReedCMS Team".to_string(),
            description: "SEO optimization and sitemap generation".to_string(),
            license: "MIT".to_string(),
            homepage: None,
            repository: None,
            keywords: vec!["seo".to_string(), "sitemap".to_string()],
            categories: vec!["optimization".to_string()],
            api_version: "1.0.0".to_string(),
        }
    }
}

impl SeoPlugin {
    /// Subscribe to content events
    async fn subscribe_to_events(&self, api: &PluginApi) -> PluginResult<()> {
        // Content created
        api.events().subscribe(
            "content.created",
            |event: PluginEvent| async move {
                // Update sitemap when content is created
                if let Some(content_id) = event.data.get("id").and_then(|v| v.as_str()) {
                    self.add_to_sitemap(content_id).await?;
                }
                Ok(())
            }
        ).await?;
        
        // Content updated
        api.events().subscribe(
            "content.updated",
            |event: PluginEvent| async move {
                // Update sitemap entry
                if let Some(content_id) = event.data.get("id").and_then(|v| v.as_str()) {
                    self.update_sitemap_entry(content_id).await?;
                }
                Ok(())
            }
        ).await?;
        
        // Content deleted
        api.events().subscribe(
            "content.deleted",
            |event: PluginEvent| async move {
                // Remove from sitemap
                if let Some(content_id) = event.data.get("id").and_then(|v| v.as_str()) {
                    self.remove_from_sitemap(content_id).await?;
                }
                Ok(())
            }
        ).await?;
        
        Ok(())
    }
    
    /// Register SEO hooks
    async fn register_seo_hooks(&self, api: &PluginApi) -> PluginResult<()> {
        // Add meta tags to rendered pages
        api.hooks().register(
            "after_render",
            90,
            |mut data: RenderedContent| async move {
                // Inject SEO meta tags
                let meta_tags = self.generate_meta_tags(&data).await?;
                
                // Insert into head section
                if let Some(head_pos) = data.html.find("</head>") {
                    data.html.insert_str(head_pos, &meta_tags);
                }
                
                Ok(data)
            }
        ).await?;
        
        // Validate SEO fields
        api.hooks().register(
            "validate_data",
            60,
            |data: ValidationData| async move {
                let mut errors = vec![];
                
                // Check meta description length
                if let Some(desc) = data.fields.get("meta_description").and_then(|v| v.as_str()) {
                    if desc.len() > 160 {
                        errors.push(ValidationError {
                            field: "meta_description".to_string(),
                            message: "Meta description should be under 160 characters".to_string(),
                        });
                    }
                }
                
                // Check title length
                if let Some(title) = data.fields.get("title").and_then(|v| v.as_str()) {
                    if title.len() > 60 {
                        errors.push(ValidationError {
                            field: "title".to_string(),
                            message: "Title should be under 60 characters for optimal SEO".to_string(),
                        });
                    }
                }
                
                Ok(ValidationResult {
                    valid: errors.is_empty(),
                    errors,
                })
            }
        ).await?;
        
        Ok(())
    }
    
    /// Generate meta tags
    async fn generate_meta_tags(&self, content: &RenderedContent) -> PluginResult<String> {
        let mut tags = String::new();
        
        // Basic meta tags
        if let Some(description) = content.metadata.get("description").and_then(|v| v.as_str()) {
            tags.push_str(&format!(
                r#"<meta name="description" content="{}">\n"#,
                html_escape(description)
            ));
        }
        
        // Open Graph tags
        if let Some(title) = content.metadata.get("title").and_then(|v| v.as_str()) {
            tags.push_str(&format!(
                r#"<meta property="og:title" content="{}">\n"#,
                html_escape(title)
            ));
        }
        
        // Twitter Card tags
        tags.push_str(r#"<meta name="twitter:card" content="summary_large_image">\n"#);
        
        // Canonical URL
        if let Some(url) = content.metadata.get("url").and_then(|v| v.as_str()) {
            tags.push_str(&format!(
                r#"<link rel="canonical" href="{}">\n"#,
                url
            ));
        }
        
        Ok(tags)
    }
}

/// Sitemap generator
struct SitemapGenerator {
    // Implementation
}
```

## HTTP API Extension Plugin

```rust
/// Weather widget plugin that fetches external data
use reed_plugin_api::*;
use async_trait::async_trait;

pub struct WeatherPlugin {
    api: Option<PluginApi>,
    api_key: String,
}

#[async_trait]
impl ReedPlugin for WeatherPlugin {
    async fn init(&mut self, api: PluginApi) -> PluginResult<()> {
        self.api = Some(api.clone());
        
        // Get API key from config
        self.api_key = api.config()
            .get_value::<String>("weather_api_key")
            .await?
            .ok_or_else(|| PluginError::InvalidArgument(
                "Weather API key not configured".to_string()
            ))?;
        
        // Register template function
        api.templates().register_function(
            "weather_widget",
            move |args: TemplateFunctionArgs| {
                let api_key = self.api_key.clone();
                async move {
                    let location = args.get("location")
                        .and_then(|v| v.as_str())
                        .unwrap_or("London");
                    
                    // Fetch weather data
                    let weather_data = self.fetch_weather(location, &api_key).await?;
                    
                    // Render widget HTML
                    let html = self.render_weather_widget(&weather_data);
                    
                    Ok(json!(html))
                }
            }
        ).await?;
        
        // Register shortcode
        api.content().register_shortcode(
            "weather",
            |params: ShortcodeParams| async move {
                let location = params.get("location").unwrap_or("London");
                format!("{{{{ weather_widget location='{}' }}}}", location)
            }
        ).await?;
        
        Ok(())
    }
    
    fn info(&self) -> PluginInfo {
        PluginInfo {
            name: "Weather Widget Plugin".to_string(),
            version: "0.1.0".to_string(),
            author: "ReedCMS Team".to_string(),
            description: "Display weather information in your content".to_string(),
            license: "MIT".to_string(),
            homepage: None,
            repository: None,
            keywords: vec!["weather".to_string(), "widget".to_string()],
            categories: vec!["widgets".to_string()],
            api_version: "1.0.0".to_string(),
        }
    }
}

impl WeatherPlugin {
    /// Fetch weather data from API
    async fn fetch_weather(
        &self,
        location: &str,
        api_key: &str,
    ) -> PluginResult<WeatherData> {
        let api = self.api.as_ref().unwrap();
        
        // Build API URL
        let url = format!(
            "https://api.openweathermap.org/data/2.5/weather?q={}&appid={}&units=metric",
            urlencoding::encode(location),
            api_key
        );
        
        // Make HTTP request
        let response = api.http().get(&url, None).await?;
        
        if response.status != 200 {
            return Err(PluginError::Internal(
                format!("Weather API returned status {}", response.status)
            ));
        }
        
        // Parse response
        let weather_data: WeatherData = serde_json::from_value(response.body)?;
        
        // Cache result for 10 minutes
        let cache_key = format!("weather:{}", location);
        api.cache().set(&cache_key, &weather_data, Some(600)).await?;
        
        Ok(weather_data)
    }
    
    /// Render weather widget HTML
    fn render_weather_widget(&self, data: &WeatherData) -> String {
        format!(
            r#"
            <div class="weather-widget">
                <h3>{}</h3>
                <div class="weather-info">
                    <span class="temperature">{:.1}°C</span>
                    <span class="description">{}</span>
                </div>
                <div class="weather-details">
                    <span>Feels like: {:.1}°C</span>
                    <span>Humidity: {}%</span>
                    <span>Wind: {:.1} m/s</span>
                </div>
            </div>
            "#,
            data.name,
            data.main.temp,
            data.weather[0].description,
            data.main.feels_like,
            data.main.humidity,
            data.wind.speed
        )
    }
}

#[derive(Debug, Deserialize)]
struct WeatherData {
    name: String,
    main: MainData,
    weather: Vec<WeatherInfo>,
    wind: WindData,
}

#[derive(Debug, Deserialize)]
struct MainData {
    temp: f64,
    feels_like: f64,
    humidity: u32,
}

#[derive(Debug, Deserialize)]
struct WeatherInfo {
    description: String,
}

#[derive(Debug, Deserialize)]
struct WindData {
    speed: f64,
}
```

## Admin UI Extension Plugin

```rust
/// Plugin that extends the admin interface
use reed_plugin_api::*;
use async_trait::async_trait;

pub struct AnalyticsPlugin {
    api: Option<PluginApi>,
}

#[async_trait]
impl ReedPlugin for AnalyticsPlugin {
    async fn init(&mut self, api: PluginApi) -> PluginResult<()> {
        self.api = Some(api.clone());
        
        // Register admin routes
        api.admin().register_menu_item(AdminMenuItem {
            id: "analytics".to_string(),
            label: "Analytics".to_string(),
            icon: "chart-line".to_string(),
            url: "/admin/analytics".to_string(),
            parent: None,
            position: 50,
            permissions: vec!["analytics.view".to_string()],
        }).await?;
        
        // Register admin page
        api.admin().register_page(
            "/admin/analytics",
            |request: AdminPageRequest| async move {
                // Generate analytics dashboard
                let stats = self.collect_statistics().await?;
                
                Ok(AdminPageResponse {
                    title: "Analytics Dashboard".to_string(),
                    content: self.render_analytics_dashboard(&stats),
                    scripts: vec!["/plugins/analytics/dashboard.js".to_string()],
                    styles: vec!["/plugins/analytics/dashboard.css".to_string()],
                })
            }
        ).await?;
        
        // Register API endpoints
        api.admin().register_api_endpoint(
            "GET",
            "/api/analytics/stats",
            |request: ApiRequest| async move {
                let period = request.query.get("period").unwrap_or("week");
                let stats = self.get_stats_for_period(period).await?;
                Ok(json!(stats))
            }
        ).await?;
        
        // Track page views
        api.hooks().register(
            "page_view",
            100,
            |data: PageViewData| async move {
                self.track_page_view(&data).await?;
                Ok(data)
            }
        ).await?;
        
        Ok(())
    }
    
    fn info(&self) -> PluginInfo {
        PluginInfo {
            name: "Analytics Plugin".to_string(),
            version: "0.1.0".to_string(),
            author: "ReedCMS Team".to_string(),
            description: "Analytics and statistics for ReedCMS".to_string(),
            license: "MIT".to_string(),
            homepage: None,
            repository: None,
            keywords: vec!["analytics".to_string(), "statistics".to_string()],
            categories: vec!["admin".to_string()],
            api_version: "1.0.0".to_string(),
        }
    }
}

impl AnalyticsPlugin {
    /// Render analytics dashboard
    fn render_analytics_dashboard(&self, stats: &AnalyticsStats) -> String {
        format!(
            r#"
            <div class="analytics-dashboard">
                <div class="stats-grid">
                    <div class="stat-card">
                        <h3>Page Views</h3>
                        <div class="stat-value">{}</div>
                        <div class="stat-change">+{}%</div>
                    </div>
                    <div class="stat-card">
                        <h3>Unique Visitors</h3>
                        <div class="stat-value">{}</div>
                        <div class="stat-change">+{}%</div>
                    </div>
                    <div class="stat-card">
                        <h3>Avg. Session Duration</h3>
                        <div class="stat-value">{}</div>
                    </div>
                    <div class="stat-card">
                        <h3>Bounce Rate</h3>
                        <div class="stat-value">{:.1}%</div>
                    </div>
                </div>
                
                <div class="chart-container">
                    <canvas id="analytics-chart"></canvas>
                </div>
                
                <div class="top-pages">
                    <h3>Top Pages</h3>
                    <table>
                        <thead>
                            <tr>
                                <th>Page</th>
                                <th>Views</th>
                                <th>Avg. Time</th>
                            </tr>
                        </thead>
                        <tbody>
                            {}
                        </tbody>
                    </table>
                </div>
            </div>
            "#,
            stats.page_views,
            stats.page_views_change,
            stats.unique_visitors,
            stats.unique_visitors_change,
            format_duration(stats.avg_session_duration),
            stats.bounce_rate,
            self.render_top_pages(&stats.top_pages)
        )
    }
}

#[derive(Debug, Serialize)]
struct AnalyticsStats {
    page_views: u64,
    page_views_change: f64,
    unique_visitors: u64,
    unique_visitors_change: f64,
    avg_session_duration: u64,
    bounce_rate: f64,
    top_pages: Vec<PageStats>,
}

#[derive(Debug, Serialize)]
struct PageStats {
    url: String,
    views: u64,
    avg_time: u64,
}
```

## Plugin Testing Example

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use reed_plugin_api::testing::*;
    
    #[tokio::test]
    async fn test_plugin_initialization() {
        // Create test environment
        let env = TestEnvironment::new();
        let api = env.create_plugin_api("test-plugin");
        
        // Create and initialize plugin
        let mut plugin = GalleryPlugin::default();
        let result = plugin.init(api.clone()).await;
        
        assert!(result.is_ok());
        
        // Verify schema was registered
        let schema = api.storage()
            .get("schema:gallery")
            .await
            .unwrap();
        
        assert!(schema.is_some());
    }
    
    #[tokio::test]
    async fn test_gallery_validation() {
        let env = TestEnvironment::new();
        let api = env.create_plugin_api("test-plugin");
        
        let mut plugin = GalleryPlugin::default();
        plugin.init(api.clone()).await.unwrap();
        
        // Test validation hook
        let hook_result = api.hooks()
            .execute::<ValidationData, ValidationResult>(
                "validate_data",
                ValidationData {
                    content_type: "gallery".to_string(),
                    fields: json!({
                        "title": "Test Gallery",
                        "images": []
                    }),
                }
            )
            .await
            .unwrap();
        
        assert!(!hook_result.valid);
        assert_eq!(hook_result.errors.len(), 1);
        assert_eq!(hook_result.errors[0].field, "images");
    }
}
```

## Summary

Diese Plugin Examples zeigen:
- **Content Extensions** - Gallery Plugin mit Custom Content Type
- **WASM Plugins** - Markdown Processor mit Sandboxing
- **Event-Driven** - SEO Plugin mit Event Subscriptions
- **External APIs** - Weather Widget mit HTTP Requests
- **Admin Extensions** - Analytics Dashboard Plugin
- **Testing** - Plugin Testing Best Practices

Praktische Beispiele für verschiedene Plugin Use Cases.
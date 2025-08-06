# ReedCMS-06-Templates.md

## Template System Philosophy

**Web Components + Tera + Redis = Zero Mental Load**

ReedCMS template system eliminates the complexity of traditional template engines by using Web Components as the bridge between server-side Tera templates and client-side ES2025+ JavaScript, with all data flowing through the 4-layer dataflow architecture.

**4k Demo Scene Principle:** Every template operation has a clear, single purpose with predictable behavior.

## Architecture Overview

### Three-Layer Template Stack
```
┌─────────────────────────────────────────────────────────┐
│                 Browser Layer                           │
│   Web Components + ES2025+ + Vanilla CSS               │
│   JSON attributes → Component initialization            │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                Asset Pipeline Layer                     │
│   Rust-based bundling + CSS optimization               │
│   ES2025+ modules + Tree-shaking + Minification        │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                Server Template Layer                    │
│   Tera templates + 4-layer dataflow integration        │
│   HTML generation with Web Component integration       │
└─────────────────────────────────────────────────────────┘
```

## Tera Template Integration

### Template Structure with Context Scoping
```tera
{# page.tera - renders complete page with scoped snippets #}
<!DOCTYPE html>
<html lang="{{ locale }}">
<head>
    <title>{{ page.content.title }}</title>
    <meta name="description" content="{{ page.content.meta_description }}">
    <link rel="stylesheet" href="/assets/bundle-{{ bundle_hash }}.css">
</head>
<body>
    {% for child in page.children %}
        {% include "snippets(" ~ scope ~ ")" with snippet=child %}
    {% endfor %}
    
    <script type="module" src="/assets/bundle-{{ bundle_hash }}.js"></script>
</body>
</html>
```

### Context Scoping System (Detailed)

**Core Innovation:** ReedCMS templates support multi-dimensional context scoping with intelligent fallback chains, enabling location/season/audience specific content without template duplication.

#### Scoping Syntax
```tera
{# Multi-dimensional context scoping #}
{% include "snippets(berlin.christmas.b2b)" with snippet=child %}
{# Fallback chain: berlin.christmas.b2b → berlin.christmas → berlin.default → default #}

{# Location-specific content #}
{% include "snippets(berlin)" with snippet=child %}
{% include "snippets(münchen)" with snippet=child %}
{% include "snippets(hamburg)" with snippet=child %}

{# Seasonal variations #}
{% include "snippets(christmas)" with snippet=child %}
{% include "snippets(black-friday)" with snippet=child %}
{% include "snippets(summer-sale)" with snippet=child %}

{# Target audience contexts #}
{% include "snippets(b2b)" with snippet=child %}
{% include "snippets(b2c)" with snippet=child %}
{% include "snippets(enterprise)" with snippet=child %}

{# Complex multi-dimensional scoping #}
{% include "snippets(münchen.black-friday.enterprise)" with snippet=child %}
{# Full fallback: münchen.black-friday.enterprise → münchen.black-friday → münchen.default → black-friday.enterprise → black-friday → enterprise → default #}
```

#### Fallback Resolution Algorithm
```rust
pub struct ContextScopingResolver {
    template_cache: HashMap<String, String>,
    scope_hierarchy: Vec<String>,
}

impl ContextScopingResolver {
    pub fn resolve_template_path(&self, snippet_type: &str, scope: &str) -> String {
        let scope_parts: Vec<&str> = scope.split('.').collect();
        
        // Generate fallback chain from most specific to most general
        let mut fallback_paths = Vec::new();
        
        // Full scope first
        fallback_paths.push(format!("snippets/{}/{}.{}.tera", snippet_type, snippet_type, scope));
        
        // Progressive scope reduction
        for i in (1..scope_parts.len()).rev() {
            let partial_scope = scope_parts[..i].join(".");
            fallback_paths.push(format!("snippets/{}/{}.{}.tera", snippet_type, snippet_type, partial_scope));
        }
        
        // Default template last
        fallback_paths.push(format!("snippets/{}/{}.tera", snippet_type, snippet_type));
        
        // Return first existing template
        for path in fallback_paths {
            if self.template_exists(&path) {
                return path;
            }
        }
        
        // Fallback to base template
        format!("snippets/{}/{}.tera", snippet_type, snippet_type)
    }
    
    fn template_exists(&self, path: &str) -> bool {
        // Check filesystem or template cache
        std::path::Path::new(path).exists()
    }
}
```

#### File Structure for Scoped Templates
```
themes/default/snippets/hero-banner/
├── hero-banner.tera                    # Default template
├── hero-banner.berlin.tera             # Berlin-specific
├── hero-banner.christmas.tera          # Christmas theme
├── hero-banner.b2b.tera               # B2B audience
├── hero-banner.berlin.christmas.tera   # Berlin + Christmas
├── hero-banner.berlin.b2b.tera        # Berlin + B2B
├── hero-banner.christmas.b2b.tera     # Christmas + B2B
├── hero-banner.berlin.christmas.b2b.tera # Full context
├── hero-banner.js                      # JavaScript component
└── hero-banner.css                     # Base styles
```

#### Scoped CSS Integration
```css
/* Base hero-banner styles */
hero-banner {
    padding: 2rem;
    background: #f0f0f0;
}

/* Location-specific styles */
hero-banner[data-scope*="berlin"] {
    background: #ff6b6b; /* Berlin brand color */
}

hero-banner[data-scope*="münchen"] {
    background: #4ecdc4; /* Munich brand color */
}

/* Seasonal styles */
hero-banner[data-scope*="christmas"] {
    border: 2px solid #c41e3a;
    background-image: url('/assets/christmas-pattern.png');
}

/* Audience-specific styles */
hero-banner[data-scope*="b2b"] {
    font-family: 'Corporate', sans-serif;
    color: #2c3e50;
}

/* Multi-dimensional combinations */
hero-banner[data-scope="berlin.christmas.b2b"] {
    /* Combines all three contexts */
    background: linear-gradient(45deg, #ff6b6b, #c41e3a);
    font-family: 'Corporate', sans-serif;
}
```

#### Context Detection and Auto-Scoping
```rust
pub struct ContextDetector {
    geoip_service: GeoIPService,
    user_agent_parser: UserAgentParser,
    session_store: SessionStore,
}

impl ContextDetector {
    pub async fn detect_context(&self, request: &HttpRequest) -> String {
        let mut context_parts = Vec::new();
        
        // Location detection
        if let Some(location) = self.detect_location(request).await {
            context_parts.push(location);
        }
        
        // Seasonal detection
        if let Some(season) = self.detect_season().await {
            context_parts.push(season);
        }
        
        // Audience detection (from session/cookies)
        if let Some(audience) = self.detect_audience(request).await {
            context_parts.push(audience);
        }
        
        context_parts.join(".")
    }
    
    async fn detect_location(&self, request: &HttpRequest) -> Option<String> {
        // IP-based location detection
        let client_ip = request.connection_info().remote_addr()?;
        let geo_info = self.geoip_service.lookup(client_ip).await.ok()?;
        
        match geo_info.city.to_lowercase().as_str() {
            "berlin" => Some("berlin".to_string()),
            "munich" | "münchen" => Some("münchen".to_string()),
            "hamburg" => Some("hamburg".to_string()),
            _ => None
        }
    }
    
    async fn detect_season(&self) -> Option<String> {
        use chrono::{Datelike, Local};
        let now = Local::now();
        let month = now.month();
        
        match month {
            12 | 1 | 2 => Some("christmas".to_string()),
            6 | 7 | 8 => Some("summer-sale".to_string()),
            11 => Some("black-friday".to_string()),
            _ => None
        }
    }
    
    async fn detect_audience(&self, request: &HttpRequest) -> Option<String> {
        // Check session data, referrer, or URL parameters
        if let Some(audience) = request.headers().get("X-Audience-Type") {
            return audience.to_str().ok().map(|s| s.to_string());
        }
        
        // Check URL parameters
        if let Some(query) = request.uri().query() {
            if query.contains("utm_source=enterprise") {
                return Some("enterprise".to_string());
            }
            if query.contains("utm_source=b2b") {
                return Some("b2b".to_string());
            }
        }
        
        None
    }
}
```

#### Template Rendering with Context
```rust
pub async fn render_scoped_template(
    snippet_type: &str, 
    snippet_data: &Value,
    context: &str
) -> Result<String, TemplateError> {
    let resolver = ContextScopingResolver::new();
    let template_path = resolver.resolve_template_path(snippet_type, context);
    
    // Create Tera context with scoping information
    let mut tera_context = Context::new();
    tera_context.insert("snippet", snippet_data);
    tera_context.insert("scope", context);
    tera_context.insert("scope_parts", &context.split('.').collect::<Vec<_>>());
    
    // Render with resolved template
    let rendered = TERA_ENGINE.render(&template_path, &tera_context)?;
    
    // Add scope data attributes for CSS targeting
    let rendered_with_scope = add_scope_attributes(rendered, context);
    
    Ok(rendered_with_scope)
}

fn add_scope_attributes(html: String, scope: &str) -> String {
    // Add data-scope attribute to root element
    if let Some(tag_end) = html.find('>') {
        let mut result = html;
        result.insert_str(tag_end, &format!(" data-scope=\"{}\"", scope));
        result
    } else {
        html
    }
}
```

### Snippet Template Pattern with CSS Controls
```tera
{# snippets/hero-banner/hero-banner.tera #}
<hero-banner 
    scope="{{ scope }}"
    timeframe="{{ snippet.content.timeframe }}"
    style="{% if snippet.css %}{{ snippet.css.to_style_string }}{% endif %}"
    title="{{ snippet.content.title | escape }}"
    subtitle="{{ snippet.content.subtitle | escape }}"
    background-url="{{ snippet.content.background_url | escape }}">
</hero-banner>
```

### Advanced CSS Control Template
```tera
{# snippets/text-with-image/text-with-image.tera #}
<text-with-image 
    scope="{{ scope }}"
    image-position="{{ snippet.content.image_position }}"
    style="{% if snippet.css -%}
        {% if snippet.css.margin_top %}margin-top: {{ snippet.css.margin_top }}; {% endif -%}
        {% if snippet.css.padding %}padding: {{ snippet.css.padding }}; {% endif -%}
        {% if snippet.css.border_radius %}border-radius: {{ snippet.css.border_radius }}; {% endif -%}
        {% if snippet.css.max_width %}max-width: {{ snippet.css.max_width }}; {% endif -%}
        {% if snippet.css.text_align %}text-align: {{ snippet.css.text_align }}; {% endif -%}
        {% if snippet.css.background_color %}background-color: {{ snippet.css.background_color }}; {% endif -%}
        {% if snippet.css.box_shadow %}box-shadow: {{ snippet.css.box_shadow }}; {% endif -%}
        {% if snippet.css.transform %}transform: {{ snippet.css.transform }}; {% endif -%}
        {% if snippet.css.opacity %}opacity: {{ snippet.css.opacity }}; {% endif -%}
        {% if snippet.css.z_index %}z-index: {{ snippet.css.z_index }}; {% endif -%}
    {%- endif %}">
    
    <div class="content-wrapper">
        <h2>{{ snippet.content.title }}</h2>
        <div class="text-content">{{ snippet.content.content }}</div>
        <img src="{{ snippet.content.image_url }}" alt="{{ snippet.content.title }}" />
    </div>
</text-with-image>
```

### CSS Control Context Population
```rust
// Updated template context building with CSS controls
impl TemplateContext {
    pub async fn build_from_route(route: &str, locale: &str) -> Result<Self, TemplateError> {
        // Step 1-4: Same as before (Redis structure, PostgreSQL content)
        let page_structure = resolve_page_structure_by_slug(route).await?;
        let hierarchy = load_snippet_hierarchy(&page_structure.id).await?;
        let content_ids: Vec<Uuid> = std::iter::once(page_structure.pg_id)
            .chain(hierarchy.iter().map(|h| h.pg_id))
            .collect();
        let content_data = load_content_batch(&content_ids, locale).await?;
        
        // Step 5: Merge structure with content AND CSS controls
        let page = PageWithContent::merge(page_structure, &content_data)?;
        let mut children = Vec::new();
        
        for hierarchy_item in hierarchy {
            let mut child = ChildWithContent::merge(hierarchy_item, &content_data)?;
            
            // Add CSS object based on registry definition and content
            if let Some(content) = content_data.get(&child.id) {
                let registry_def = registry.get_definition(&child.snippet_type)?;
                child.css = Some(registry_def.generate_css_object(&content.css_field_values));
            }
            
            children.push(child);
        }
        
        Ok(TemplateContext {
            page,
            children,
            bundle_hash: AssetPipeline::current_bundle_hash(),
            site_config: SiteConfig::load(),
            locale: locale.to_string(),
        })
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ChildWithContent {
    pub id: Uuid,
    pub snippet_type: String,
    pub semantic_name: Option<String>,
    pub content: ContentData,
    pub css: Option<CssObject>, // New: CSS controls
    pub children: Vec<ChildWithContent>,
}
```

### Complex Data Handling
```tera
{# snippets/accordeon/accordeon.tera #}
<accordeon-snippet 
    title="{{ snippet.content.title | escape }}"
    allow-multiple="{{ snippet.content.allow_multiple }}"
    items="{{ snippet.content.items | json_encode | escape }}">
</accordeon-snippet>
```

## Data Layer Integration

### 4-Layer Dataflow Architecture
ReedCMS uses the anti-bloat 4-layer system for optimal performance and maintainability:

#### Template Context Population
```rust
// Optimized template context building using 4-layer dataflow
pub struct TemplateContext {
    page: PageWithContent,
    children: Vec<ChildWithContent>,
    bundle_hash: String,
    site_config: SiteConfig,
    locale: String,
}

impl TemplateContext {
    pub async fn build_from_route(route: &str, locale: &str) -> Result<Self, TemplateError> {
        // Step 1: Redis - Get page structure (sub-millisecond)
        let page_structure = resolve_page_structure_by_slug(route).await?;
        
        // Step 2: Redis - Load complete hierarchy (single query)
        let hierarchy = load_snippet_hierarchy(&page_structure.id).await?;
        
        // Step 3: Collect all PostgreSQL content IDs
        let content_ids: Vec<Uuid> = std::iter::once(page_structure.pg_id)
            .chain(hierarchy.iter().map(|h| h.pg_id))
            .collect();
        
        // Step 4: PostgreSQL Main - Batch load all content (single query)
        let content_data = load_content_batch(&content_ids, locale).await?;
        
        // Step 5: Merge structure with content
        let page = PageWithContent::merge(page_structure, &content_data)?;
        let children = ChildWithContent::merge_vec(hierarchy, &content_data)?;
        
        Ok(TemplateContext {
            page,
            children,
            bundle_hash: AssetPipeline::current_bundle_hash(),
            site_config: SiteConfig::load(),
            locale: locale.to_string(),
        })
    }
}

// Batch content loading for performance
async fn load_content_batch(content_ids: &[Uuid], locale: &str) -> Result<HashMap<Uuid, ContentData>, DatabaseError> {
    let locale_suffix = locale.replace('-', "_");
    let title_column = format!("title_{}", locale_suffix);
    let content_column = format!("content_{}", locale_suffix);
    let meta_column = format!("meta_description_{}", locale_suffix);
    
    let query = format!(
        "SELECT id, snippet_type, semantic_name, {}, {}, {}, additional_data 
         FROM snippet_content 
         WHERE id = ANY($1)",
        title_column, content_column, meta_column
    );
    
    let rows = sqlx::query(&query)
        .bind(content_ids)
        .fetch_all(&pool)
        .await?;
    
    let mut content_map = HashMap::new();
    for row in rows {
        let content = ContentData {
            id: row.get("id"),
            snippet_type: row.get("snippet_type"),
            semantic_name: row.get("semantic_name"),
            title: row.get(&title_column),
            content: row.get(&content_column),
            meta_description: row.get(&meta_column),
            additional_data: row.get("additional_data"),
        };
        content_map.insert(content.id, content);
    }
    
    Ok(content_map)
}
```

## Web Components Integration

### Component Registration System
```javascript
// Auto-generated component loader
// File: /assets/components.js
import { HeroBannerSnippet } from './snippets/hero-banner/hero-banner.js';
import { AccordeonSnippet } from './snippets/accordeon/accordeon.js';

// Auto-register all snippet components
customElements.define('hero-banner', HeroBannerSnippet);
customElements.define('accordeon-snippet', AccordeonSnippet);
```

### Component Base Class
```javascript
// Base class for all snippet components
export class ReedSnippet extends HTMLElement {
    constructor() {
        super();
        this.snippetData = {};
    }
    
    connectedCallback() {
        this.parseAttributes();
        this.render();
        this.bindEvents();
    }
    
    parseAttributes() {
        // Parse JSON attributes to object
        for (const attr of this.attributes) {
            try {
                this.snippetData[attr.name] = JSON.parse(attr.value);
            } catch {
                this.snippetData[attr.name] = attr.value;
            }
        }
    }
    
    render() {
        // Override in child classes
        throw new Error('render() must be implemented by snippet component');
    }
    
    bindEvents() {
        // Override in child classes for event handling
    }
}
```

### Snippet Component Example
```javascript
// snippets/hero-banner/hero-banner.js
import { ReedSnippet } from '../../base/ReedSnippet.js';

export class HeroBannerSnippet extends ReedSnippet {
    render() {
        this.innerHTML = `
            <section class="hero-banner" style="background-image: url(${this.snippetData['background-url']})">
                <div class="hero-content">
                    <h1 class="hero-title">${this.snippetData.title}</h1>
                    <p class="hero-subtitle">${this.snippetData.subtitle}</p>
                </div>
            </section>
        `;
    }
    
    bindEvents() {
        // Hero-specific event handling
        const title = this.querySelector('.hero-title');
        if (title) {
            title.addEventListener('click', () => {
                console.log('Hero title clicked');
            });
        }
    }
}
```

## Asset Pipeline

### Rust-Based Build System
```rust
pub struct AssetPipeline {
    bundle_hash: String,
    css_files: Vec<PathBuf>,
    js_files: Vec<PathBuf>,
    output_dir: PathBuf,
}

impl AssetPipeline {
    pub async fn build_bundle(&mut self) -> Result<BuildResult, PipelineError> {
        // Collect all snippet assets
        let snippet_assets = self.discover_snippet_assets().await?;
        
        // Bundle CSS with tree-shaking
        let bundled_css = self.bundle_css(&snippet_assets.css_files).await?;
        
        // Bundle JavaScript ES2025+ modules
        let bundled_js = self.bundle_javascript(&snippet_assets.js_files).await?;
        
        // Generate content-based hash
        self.bundle_hash = self.generate_bundle_hash(&bundled_css, &bundled_js);
        
        // Write optimized bundles
        self.write_bundle_files(bundled_css, bundled_js).await?;
        
        Ok(BuildResult {
            bundle_hash: self.bundle_hash.clone(),
            css_size: bundled_css.len(),
            js_size: bundled_js.len(),
        })
    }
    
    async fn discover_snippet_assets(&self) -> Result<SnippetAssets, PipelineError> {
        let mut assets = SnippetAssets::new();
        
        // Scan all theme snippet directories
        let snippets_dir = self.output_dir.join("themes/default/snippets");
        let mut entries = fs::read_dir(snippets_dir).await?;
        
        while let Some(entry) = entries.next_entry().await? {
            if entry.file_type().await?.is_dir() {
                let snippet_name = entry.file_name().to_string_lossy().to_string();
                
                // Collect CSS file
                let css_path = entry.path().join(format!("{}.css", snippet_name));
                if css_path.exists() {
                    assets.css_files.push(css_path);
                }
                
                // Collect JS file
                let js_path = entry.path().join(format!("{}.js", snippet_name));
                if js_path.exists() {
                    assets.js_files.push(js_path);
                }
            }
        }
        
        Ok(assets)
    }
}
```

### CSS Bundle Optimization
```rust
impl AssetPipeline {
    async fn bundle_css(&self, css_files: &[PathBuf]) -> Result<String, PipelineError> {
        let mut bundled = String::new();
        
        for css_file in css_files {
            let content = fs::read_to_string(css_file).await?;
            
            // CSS optimization
            let optimized = self.optimize_css(&content)?;
            bundled.push_str(&optimized);
            bundled.push('\n');
        }
        
        // Global optimizations
        let minified = self.minify_css(&bundled)?;
        Ok(minified)
    }
    
    fn optimize_css(&self, css: &str) -> Result<String, PipelineError> {
        // Remove comments
        // Minimize whitespace
        // Remove unused CSS rules (tree-shaking)
        // Optimize color values
        Ok(css.to_string()) // Simplified for spec
    }
}
```

### JavaScript Bundle System
```rust
impl AssetPipeline {
    async fn bundle_javascript(&self, js_files: &[PathBuf]) -> Result<String, PipelineError> {
        let mut bundled = String::new();
        
        // Add base ReedSnippet class first
        bundled.push_str(&self.load_base_class().await?);
        
        // Add snippet components
        for js_file in js_files {
            let content = fs::read_to_string(js_file).await?;
            let processed = self.process_es_module(&content)?;
            bundled.push_str(&processed);
            bundled.push('\n');
        }
        
        // Add auto-registration code
        bundled.push_str(&self.generate_auto_registration()?);
        
        Ok(bundled)
    }
    
    fn process_es_module(&self, js_content: &str) -> Result<String, PipelineError> {
        // Convert ES2025+ modules to bundled format
        // Resolve imports
        // Tree-shake unused exports
        // Minify for production
        Ok(js_content.to_string()) // Simplified for spec
    }
}
```

## Template Rendering Engine

### Request Handling with 4-Layer Integration
```rust
pub async fn render_page_template(route: &str, locale: &str) -> Result<String, RenderError> {
    // Build template context from 4-layer dataflow
    let context = TemplateContext::build_from_route(route, locale).await?;
    
    // Get page template name from registry
    let template_name = format!("pages/{}.tera", context.page.template_type);
    
    // Render with Tera
    let rendered = TERA_ENGINE.render(&template_name, &context)?;
    
    Ok(rendered)
}
```

### Snippet Inclusion System
```rust
// Custom Tera function for snippet rendering
pub fn register_tera_functions(tera: &mut Tera) {
    tera.register_function("render_snippet", render_snippet_function);
    tera.register_filter("json_encode", json_encode_filter);
    tera.register_filter("escape_html_attr", escape_html_attr_filter);
}

fn render_snippet_function(args: &HashMap<String, Value>) -> Result<Value, Error> {
    let snippet_type = args.get("type").and_then(|v| v.as_str())
        .ok_or_else(|| Error::msg("Missing snippet type"))?;
    let snippet_data = args.get("data")
        .ok_or_else(|| Error::msg("Missing snippet data"))?;
    
    // Load snippet template
    let template_path = format!("snippets/{}/{}.tera", snippet_type, snippet_type);
    
    // Create context for snippet
    let mut context = Context::new();
    context.insert("snippet", snippet_data);
    
    // Render snippet template
    let rendered = TERA_ENGINE.render(&template_path, &context)?;
    Ok(Value::String(rendered))
}
```

## Development Workflow

### Hot Reload Development
```rust
pub struct DevServer {
    tera_watcher: notify::RecommendedWatcher,
    css_watcher: notify::RecommendedWatcher,
    js_watcher: notify::RecommendedWatcher,
}

impl DevServer {
    pub async fn start_with_hot_reload(&mut self) -> Result<(), DevServerError> {
        // Watch template files
        self.tera_watcher.watch("themes/", RecursiveMode::Recursive)?;
        
        // Watch CSS files  
        self.css_watcher.watch("themes/*/snippets/*/*.css", RecursiveMode::NonRecursive)?;
        
        // Watch JavaScript files
        self.js_watcher.watch("themes/*/snippets/*/*.js", RecursiveMode::NonRecursive)?;
        
        // WebSocket connection for browser refresh
        let ws_server = WebSocketServer::new("localhost:3001");
        ws_server.start().await?;
        
        Ok(())
    }
    
    async fn handle_file_change(&mut self, path: PathBuf) -> Result<(), DevServerError> {
        match path.extension().and_then(|ext| ext.to_str()) {
            Some("tera") => {
                self.reload_tera_templates().await?;
                self.broadcast_reload("template").await?;
            },
            Some("css") => {
                self.rebuild_css_bundle().await?;
                self.broadcast_reload("css").await?;
            },
            Some("js") => {
                self.rebuild_js_bundle().await?;
                self.broadcast_reload("javascript").await?;
            },
            _ => {}
        }
        Ok(())
    }
}
```

### Snippet File Generation (CSV-Driven)
```rust
pub struct SnippetGenerator {
    csv_store: CsvStore,
    template_engine: Tera,
}

impl SnippetGenerator {
    pub async fn generate_snippet_files(&self, snippet_name: &str) -> Result<(), GeneratorError> {
        // Load definition from CSV layer
        let definition = self.csv_store.get_snippet_definition(snippet_name)?;
        
        // Generate .tera template
        self.generate_tera_template(snippet_name, &definition).await?;
        
        // Generate .js component
        self.generate_js_component(snippet_name, &definition).await?;
        
        // Generate .css styles
        self.generate_css_styles(snippet_name, &definition).await?;
        
        // Generate .json field metadata
        self.generate_field_metadata(snippet_name, &definition).await?;
        
        Ok(())
    }
    
    async fn generate_tera_template(&self, name: &str, def: &SnippetDefinition) -> Result<(), GeneratorError> {
        let template_content = format!(
            r#"<{component_name}{attributes}>
</{component_name}>"#,
            component_name = name,
            attributes = def.fields.iter()
                .filter(|field| field.required)
                .map(|field| format!(r#" {}="{{{{ snippet.content.{} | escape }}}}""#, field.name, field.name))
                .collect::<Vec<_>>()
                .join("")
        );
        
        let file_path = format!("themes/default/snippets/{}/{}.tera", name, name);
        fs::write(file_path, template_content).await?;
        Ok(())
    }
    
    async fn generate_js_component(&self, name: &str, def: &SnippetDefinition) -> Result<(), GeneratorError> {
        let class_name = snippet_to_js_class(name);
        let component_content = format!(
            r#"import {{ ReedSnippet }} from '../../base/ReedSnippet.js';

export class {class_name} extends ReedSnippet {{
    render() {{
        this.innerHTML = `
            <div class="{name}">
                <!-- Auto-generated component template -->
                <h2>${{this.snippetData.title || 'Default Title'}}</h2>
            </div>
        `;
    }}
    
    bindEvents() {{
        // Custom event binding for {name}
    }}
}}
"#,
            class_name = class_name,
            name = name
        );
        
        let file_path = format!("themes/default/snippets/{}/{}.js", name, name);
        fs::write(file_path, component_content).await?;
        Ok(())
    }
}
```

This template system ensures ReedCMS delivers exceptional performance while maintaining developer productivity and content management flexibility through the power of Web Components + Tera + intelligent 4-layer dataflow integration.

---

**Next: ReedCMS-07-Search.md** - Learn how content discovery integrates seamlessly with the template system and 4-layer architecture.
# ReedCMS-13-Directory-Structure.md

## Directory Structure Philosophy

**KISS-Brain Organization:** Every file and directory has a clear, single purpose with predictable location patterns. The structure follows the separation of concerns principle while maintaining mental simplicity for developers.

**Core Principles:**
- **Snippets** contain content logic (theme-independent)
- **Themes** contain only visual overrides (scope-based)
- **Plugins** manage their own isolated ecosystems
- **UCG System** manages themes as entities with associations

## Root Project Structure

```
reed-cms/
├── src/                           # Rust source code
│   ├── main.rs                   # CLI entry point
│   ├── server/                   # Web server components
│   ├── ucg/                      # Universal Content Graph
│   ├── registry/                 # Snippet registry system
│   ├── templates/                # Template engine
│   ├── search/                   # Search system
│   └── plugins/                  # Plugin system
├── snippets/                     # Content logic (theme-independent)
│   ├── page/
│   │   ├── page.tera            # Template (required)
│   │   ├── page.js              # JavaScript component (required) 
│   │   └── page.css             # Base styling (required)
│   ├── hero-banner/
│   │   ├── hero-banner.tera
│   │   ├── hero-banner.js
│   │   └── hero-banner.css
│   └── product-card/
│       ├── product-card.tera
│       ├── product-card.js
│       └── product-card.css
├── themes/                       # Visual overrides only
│   ├── corporate/               # Base theme (user-defined at setup)
│   │   ├── hero-banner/
│   │   │   ├── hero-banner.css  # Override base styles
│   │   │   └── hero-banner.js   # Override base behavior
│   │   └── product-card/
│   │       └── product-card.css
│   ├── berlin/                  # Location-specific theme
│   │   └── hero-banner/
│   │       └── hero-banner.css
│   ├── christmas/               # Seasonal theme
│   │   ├── hero-banner/
│   │   │   └── hero-banner.css
│   │   └── product-card/
│   │       └── product-card.css
│   └── berlin.christmas/        # Multi-dimensional theme
│       └── hero-banner/
│           ├── hero-banner.css
│           └── hero-banner.js
├── plugins/                     # Plugin ecosystem
│   ├── ai-core/                # Rust Pro Plugin
│   │   ├── bin/
│   │   │   └── ai-core         # Plugin binary
│   │   ├── plugin.toml         # Plugin configuration
│   │   └── snippets/           # Plugin-owned snippets
│   │       └── ai-suggestion/
│   │           ├── ai-suggestion.tera
│   │           ├── ai-suggestion.js
│   │           └── ai-suggestion.css
│   └── simple-seo/             # LUA Plugin
│       ├── simple-seo.lua      # Plugin script
│       ├── plugin.toml
│       └── snippets/
│           └── seo-widget/
│               ├── seo-widget.tera
│               ├── seo-widget.js
│               └── seo-widget.css
├── config/                      # CSV-driven configuration
│   ├── snippets.csv            # Snippet type definitions
│   ├── themes.csv              # Theme registry
│   ├── associations.csv        # Site structure templates
│   ├── permissions.csv         # Permission definitions
│   ├── plugin-registry.csv     # Plugin registry
│   ├── search-words.csv        # Search index definitions
│   └── translations.csv        # i18n labels
├── build/                       # Generated files
│   ├── snippets/               # Auto-generated metadata
│   │   ├── hero-banner/
│   │   │   ├── corporate.json  # Theme-specific metadata
│   │   │   ├── berlin.json
│   │   │   └── christmas.json
│   │   └── product-card/
│   │       └── corporate.json
│   ├── assets/                 # Bundled assets
│   │   ├── bundle-{hash}.css   # CSS bundle
│   │   ├── bundle-{hash}.js    # JavaScript bundle
│   │   └── components.js       # Auto-registered components
│   └── themes/                 # Theme compilation results
│       ├── corporate/
│       └── berlin.christmas/
├── data/                        # Runtime data
│   ├── sites/
│   │   ├── production/         # Site-specific data
│   │   └── staging/
│   ├── postgres/               # PostgreSQL data directory
│   └── redis/                  # Redis data directory
└── tmp/                        # Runtime temporary files
    ├── sockets/                # Unix domain sockets
    │   ├── core.sock
    │   └── plugins/
    │       ├── ai-core.sock
    │       └── simple-seo.sock
    ├── fifos/                  # Plugin communication
    │   ├── ai-core→analytics
    │   └── analytics→ai-core
    └── pid/                    # Process IDs
        ├── core.pid
        └── plugins/
            ├── ai-core.pid
            └── simple-seo.pid
```

## File Resolution Patterns

### Snippet File Resolution

**Template Resolution (themes cannot override .tera files):**
```rust
// Templates are always resolved from snippets/ directory
pub fn resolve_snippet_template(snippet_name: &str) -> Option<PathBuf> {
    let template_path = Path::new("snippets")
        .join(snippet_name)
        .join(format!("{}.tera", snippet_name));
    
    if template_path.exists() {
        Some(template_path)
    } else {
        None
    }
}
```

**Asset Resolution (CSS/JS with theme overrides):**
```rust
pub fn resolve_snippet_assets(snippet_name: &str, theme_chain: &[String]) -> SnippetAssets {
    let snippet_dir = Path::new("snippets").join(snippet_name);
    
    // Base files (always included)
    let mut css_files = vec![snippet_dir.join(format!("{}.css", snippet_name))];
    let mut js_files = vec![snippet_dir.join(format!("{}.js", snippet_name))];
    
    // Theme overrides (in priority order)
    for theme in theme_chain {
        let theme_dir = Path::new("themes").join(theme).join(snippet_name);
        
        let theme_css = theme_dir.join(format!("{}.css", snippet_name));
        if theme_css.exists() {
            css_files.push(theme_css);
        }
        
        let theme_js = theme_dir.join(format!("{}.js", snippet_name));
        if theme_js.exists() {
            js_files.push(theme_js);
        }
    }
    
    SnippetAssets { css_files, js_files }
}
```

### Theme Resolution Chain

**Context-Based Theme Chain:**
```rust
pub async fn resolve_theme_chain(context: &str, base_theme: &str) -> Vec<String> {
    // Context: "berlin.christmas.b2b"
    // Base theme: "corporate"
    
    let mut theme_chain = vec![base_theme.to_string()];
    let context_parts: Vec<&str> = context.split('.').collect();
    
    // Progressive theme additions
    for i in 1..=context_parts.len() {
        let partial_context = context_parts[..i].join(".");
        let theme_name = format!("{}.{}", base_theme, partial_context);
        
        if theme_exists(&theme_name).await {
            theme_chain.push(theme_name);
        }
    }
    
    // Final multi-dimensional theme
    if context_parts.len() > 1 {
        let full_theme = format!("{}.{}", base_theme, context);
        if theme_exists(&full_theme).await {
            theme_chain.push(full_theme);
        }
    }
    
    theme_chain
}

// Example result for context "berlin.christmas" with base "corporate":
// ["corporate", "corporate.berlin", "corporate.christmas", "corporate.berlin.christmas"]
```

## Plugin Integration Patterns

### Plugin Installation

**Plugin-Owned Snippets:**
```bash
# Plugin installation copies snippets to main directory
plugins/ai-core/snippets/ai-suggestion/ → snippets/ai-suggestion/

# Plugin maintains ownership through registry
config/snippets.csv:
ai-suggestion,title,String,true,"AI Suggestion",ai-core-plugin
```

**Plugin Removal:**
```bash
# Clean removal of plugin-owned snippets
reed plugin uninstall "ai-core"
# Removes: snippets/ai-suggestion/ (if owned by ai-core)
# Removes: plugins/ai-core/
# Updates: config/snippets.csv (removes ai-suggestion entries)
```

### Plugin Directory Isolation

Each plugin maintains complete isolation:
```
plugins/plugin-name/
├── bin/                    # Rust binaries (for native plugins)
├── src/                    # Source code (development)
├── plugin.toml            # Plugin configuration
├── README.md              # Documentation
├── snippets/              # Plugin-owned snippets (before installation)
└── tests/                 # Plugin tests
```

## CLI Syntax Standards

### String Definitions vs References

**String Definitions (creating new entities):**
```bash
# Always use quotes for new string definitions
reed snippet create "hero-banner" /
  name $welcome_hero /
  with title "Welcome Message"

reed theme create "corporate" --base-theme
reed theme create "berlin" --context location:"berlin" --parent "corporate"

reed registry define "product-showcase" /
  routable true /
  description "E-commerce product showcase"
```

**References (using existing entities):**
```bash
# Always use $ prefix for existing entity references
reed snippet update $welcome_hero /
  set field "title" to "Updated Welcome"

reed schema bond /
  parent-type "theme" parent $corporate /
  child-type "theme" child $berlin

reed theme scaffold $berlin "hero-banner" --override css
```

### FreeBSD-Style Command Chaining

**Syntax Pattern:**
```bash
# Base pattern: command "target" / modifier value / modifier value
reed command "target" /
  modifier value /
  modifier value /
  modifier value

# Real examples:
reed snippet create "hero-banner" /
  name $welcome_hero /
  with title "Welcome" subtitle "Modern CMS" /
  associate with page $homepage at path "home.hero"

reed theme create "berlin.christmas" /
  context multi:"location:berlin;season:christmas" /
  parent "corporate" /
  description "Berlin Christmas theme"
```

### Command Categories

**Entity Management:**
- `reed snippet` - Snippet operations
- `reed theme` - Theme management  
- `reed plugin` - Plugin lifecycle
- `reed schema` - UCG associations
- `reed registry` - Meta-snippet definitions

**System Operations:**
- `reed setup` - Initial system configuration
- `reed build` - Asset compilation
- `reed serve` - Development server
- `reed deploy` - Production deployment

## Build System Integration

### Auto-Generation Patterns

**Generated Metadata:**
```
build/snippets/{snippet-name}/{theme-name}.json
```

**Asset Bundles:**
```
build/assets/bundle-{content-hash}.css
build/assets/bundle-{content-hash}.js
build/assets/components.js
```

**Build Triggers:**
- File system changes in `snippets/` or `themes/`
- CSV registry updates in `config/`
- Plugin installation/removal
- Explicit `reed build` command

### Development vs Production

**Development Structure:**
```
# Hot-reload enabled, source maps included
build/dev/
├── snippets/           # Unminified metadata
├── assets/             # Source maps included
└── themes/             # Development theme data
```

**Production Structure:**
```
# Optimized, minified, content-hashed
build/prod/
├── snippets/           # Minified metadata
├── assets/             # Optimized bundles
└── themes/             # Production theme data
```

## UCG Integration

### Themes as UCG Entities

**Redis UCG Storage:**
```redis
# Theme definitions
HSET entity:theme:corporate type "theme" data '{"name":"Corporate Theme","base_theme":true}'
HSET entity:theme:berlin type "theme" data '{"name":"Berlin Location","context":"location:berlin"}'

# Theme hierarchies
HSET assoc:theme.1 parent_id "null" entity_type "theme" entity_id "corporate" path "theme.1"
HSET assoc:theme.1.1 parent_id "corporate" entity_type "theme" entity_id "berlin" path "theme.1.1"
```

**CSV Sync:**
```csv
# themes.csv - synced to PostgreSQL UCG schema
theme_name,parent_theme,context_type,context_value,active,description
corporate,,base,,true,"Base corporate theme"
berlin,corporate,location,berlin,true,"Berlin-specific styling"
christmas,corporate,season,christmas,true,"Christmas seasonal theme"
```

## Performance Considerations

### File System Caching

**Template Resolution Cache:**
```rust
// Cache resolved file paths to avoid repeated filesystem checks
pub struct FilePathCache {
    template_cache: HashMap<String, Option<PathBuf>>,
    asset_cache: HashMap<String, SnippetAssets>,
    theme_cache: HashMap<String, Vec<String>>,
}
```

### Build Optimization

**Incremental Builds:**
- Track file modification times
- Only rebuild changed snippets
- Smart asset bundling based on usage

**Production Optimizations:**
- CSS/JS minification
- Content-based file hashing
- Tree-shaking unused components
- Compressed theme data

This directory structure ensures predictable file organization, clear separation of concerns, and optimal performance while maintaining the KISS-brain development philosophy throughout the ReedCMS ecosystem.
# ReedCMS-05-CLI.md

## CLI Philosophy

**Universal Content Graph CLI Interface:** All operations work on the Universal Content Graph - a single association system handling snippets, registry definitions, menus, categories, and any entity type through unified patterns.

**KISS-Brain Adaptation:** Human-readable commands for complex graph operations that route intelligently to the 4-layer architecture (CSV + PostgreSQL Main + PostgreSQL UCG + Redis).

## Command Philosophy

### FreeBSD-Style Chainable Commands
- **Readable as sentences:** Commands flow like natural language
- **Slash-separated chains:** Visual flow indicators with proper spacing
- **One operation per line:** Clear atomic operations
- **Quoted strings:** All names and values consistently quoted
- **Semantic references:** `$variable` syntax for existing entities

### Syntax Rules
```bash
# Base pattern: command "target" / modifier value / modifier value
reed snippet create "hero-banner" /
  name $welcome_hero /
  with title "Welcome Message" /
  associate with page $homepage at path "home.hero"

# Spacing: word SPACE / (FreeBSD convention)
# Quotes: ALL names and string values quoted
# References: $semantic_name for existing entities
```

## Core Commands

### Snippet Management

#### Create Snippet
```bash
# Simple snippet creation
reed snippet create "hero-banner" /
  name $welcome_hero /
  with title "Welcome to ReedCMS" subtitle "Modern CMS with Rust"

# Composite snippet (registry auto-creates children)
reed snippet create "text-with-picture" /
  name $intro_section /
  with layout "side-by-side" text_position "left"

# Page snippet (routable)
reed snippet create "page" /
  name $homepage /
  with title "Home" slug "home" meta_description "Welcome page"
```

#### Update Snippet
```bash
# Field updates
reed snippet update $welcome_hero /
  set field "title" to "New Welcome Message" /
  set field "subtitle" to "Updated subtitle"

# Bulk field updates
reed snippet update $homepage /
  set fields '{"title": "New Home", "meta_description": "Updated page"}'

# Add/remove from composite snippets
reed snippet update $intro_section /
  add child "call-to-action" at position "bottom" /
  remove child $old_cta
```

#### Query Snippets
```bash
# List snippets by type
reed snippet list --type "page" --format json
reed snippet list --type "hero-banner" --include-data

# Query by registry properties
reed snippet list --routable --format table
reed snippet list --navigable --depth 2

# Get specific snippet
reed snippet get $homepage --include-schema --format yaml
reed snippet get --slug "about" --full-hierarchy
```

#### Delete Snippet
```bash
# Safe deletion with confirmation
reed snippet delete $old_hero --confirm

# Cascade deletion (removes associations)
reed snippet delete $old_page --cascade --dry-run
reed snippet delete $old_page --cascade --force
```

### Schema Management (Universal Associations)

#### Create Schema Bonds
```bash
# Basic parent-child relationship (any entity types)
reed schema bond /
  parent-type "snippet" parent $homepage /
  child-type "snippet" child $welcome_hero /
  at path "content.1.1" /
  weight 0

# Registry composition (meta-snippet definition)
reed schema bond /
  parent-type "registry" parent $text_with_picture_def /
  child-type "snippet" child $text_snippet /
  at path "registry.1.1" /
  weight 0

# Menu structure
reed schema bond /
  parent-type "menu" parent $main_nav /
  child-type "menu-item" child $about_link /
  at path "menu.1.2" /
  weight 10

# Cross-entity associations
reed schema bond /
  parent-type "snippet" parent $product_page /
  child-type "category" child $electronics_cat /
  at path "taxonomy.1.5"
```

#### Universal Schema Queries
```bash
# Get any entity hierarchy
reed schema tree /
  root-type "snippet" root $homepage /
  depth 3 /
  include-types "snippet" "menu-item"

# Query by entity type
reed schema list /
  entity-type "registry" /
  depth 2 /
  format json

# Cross-type relationships
reed schema find /
  where parent-type "snippet" /
  and child-type "category" /
  format table
```

### Registry Management

#### Define Meta-Snippet Types
```bash
# Built-in style meta-snippet
reed registry define "product-showcase" /
  routable true /
  navigable true /
  required "title" "category" /
  indexable "title" "category" "price_range" /
  max-depth 2 /
  description "E-commerce product showcase page"

# Composite meta-snippet
reed registry define "feature-card" /
  composed-of "icon-snippet" "text-snippet" "link-snippet" /
  layout "vertical" /
  required "layout" /
  description "Feature showcase with icon and text"
```

#### Registry Queries
```bash
# List all meta-snippet types
reed registry list --format table
reed registry list --routable --format json

# Get specific definition
reed registry get "page" --include-examples
reed registry get "text-with-picture" --show-composition

# Validate registry consistency
reed registry validate --check-circular --check-fields
```

### Site Management

#### Site Operations
```bash
# Create new site
reed site create "my-blog" /
  with domain "blog.example.com" /
  theme "default" /
  locale "en_US"

# Site-scoped operations
reed site "my-blog" snippet create "page" /
  name $blog_home /
  with title "My Blog" slug "home"

# Cross-site operations
reed site copy snippet $hero_template /
  from "staging" /
  to "production" /
  update-references true
```

## Advanced Operations

### Bulk Operations
```bash
# Import from CSV/JSON
reed import snippets from "content-export.json" /
  site "production" /
  update-existing true /
  create-missing-parents true

# Export site content
reed export site "production" /
  to "backup.json" /
  include-schema true /
  include-registry true

# Batch updates
reed snippet batch-update /
  where type "hero-banner" /
  set field "version" to "2.0" /
  dry-run true
```

### Development Helpers
```bash
# Generate snippet templates
reed generate template "custom-snippet" /
  with fields "title" "content" "image_url" /
  output-dir "themes/default/snippets/"

# Validate site structure
reed validate site "production" /
  check-broken-links true /
  check-missing-templates true /
  check-registry-consistency true

# Performance analysis
reed analyze site "production" /
  check-query-performance true /
  suggest-indexes true /
  report-slow-paths true
```

## Intelligent Layer Routing

### Command Execution Flow
```rust
// CLI command execution with intelligent routing
pub async fn execute_cli_command(cmd: &CliCommand) -> Result<(), Error> {
    match cmd {
        CliCommand::CreateSnippet { snippet_type, fields, .. } => {
            // Route to PostgreSQL main schema (live content)
            let snippet_id = database.postgres_main
                .insert_snippet_content(snippet_type, fields)
                .await?;
            
            // Background: Update search index if searchable type
            if registry.is_searchable(snippet_type) {
                tokio::spawn(async move {
                    database.redis.update_search_index(&snippet_id).await.ok();
                });
            }
            
            println!("✓ Created snippet {} ({})", semantic_name, snippet_type);
        },
        
        CliCommand::DefineRegistry { snippet_name, fields, .. } => {
            // Route to CSV layer (configuration)
            csv_store.update_snippet_definition(snippet_name, fields).await?;
            
            // Background: Sync to PostgreSQL UCG schema
            tokio::spawn(async move {
                background_sync.sync_csv_to_ucg().await.ok();
            });
            
            println!("✓ Defined snippet type {}", snippet_name);
        },
        
        CliCommand::Search { query, .. } => {
            // Route to Redis search index
            let results = database.redis.search_words(query).await?;
            display_search_results(results);
        }
    }
}
```

### Layer Abstraction for Users
**90% of users don't need to understand layer routing:**
- Commands "just work" 
- Performance is automatic
- Data appears where expected

**10% of power users can learn via documentation:**
- Understanding helps with troubleshooting
- Advanced workflows benefit from layer knowledge
- System architecture courses available

## Output Formats

### Format Options
```bash
# Table format (default for lists)
reed snippet list --format table
┌─────────────────┬──────────────┬─────────────┬─────────────┐
│ Name            │ Type         │ Created     │ Path        │
├─────────────────┼──────────────┼─────────────┼─────────────┤
│ $homepage       │ page         │ 2025-01-15  │ home        │
│ $welcome_hero   │ hero-banner  │ 2025-01-15  │ home.hero   │
└─────────────────┴──────────────┴─────────────┴─────────────┘

# JSON format (machine-readable)
reed snippet get $homepage --format json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "snippet_type": "page",
  "semantic_name": "homepage",
  "data": {
    "title": "Home",
    "slug": "home",
    "meta_description": "Welcome page"
  },
  "schema": {
    "children": ["$welcome_hero", "$content_section"],
    "path": "home"
  }
}

# YAML format (config-friendly)
reed registry get "page" --format yaml
name: page
is_routable: true
is_navigable: false
required_fields:
  - title
  - slug
indexable_fields:
  - slug
  - title
  - meta_description
```

### Error Handling
```bash
# Clear error messages with suggestions
$ reed snippet create "unknown-type"
Error: Meta-snippet type "unknown-type" not found in registry.

Available types:
  - page (routable)
  - hero-banner (content-block)  
  - navigation-item (navigable)

Run: reed registry list --format table

# Validation errors with context
$ reed snippet create "page" name $test
Error: Missing required fields for meta-snippet "page":
  - title (string)
  - slug (string)

Example:
  reed snippet create "page" /
    name $test /
    with title "Test Page" slug "test"
```

## Interactive Mode

### REPL-Style Interface
```bash
# Start interactive session
$ reed
ReedCMS> snippet create "page" /
       > name $testpage /
       > with title "Test" slug "test"
✓ Created snippet $testpage (page)

ReedCMS> schema bond parent $testpage child $hero at "test.hero"
✓ Created schema bond: $testpage -> $hero

ReedCMS> exit
```

### Command History & Replay
```bash
# Save command sequence
ReedCMS> save session to "page-setup.reed"
✓ Saved 5 commands to page-setup.reed

# Replay commands
$ reed replay "page-setup.reed" --site "staging"
✓ Executed 5 commands successfully

# Dry-run mode
$ reed replay "page-setup.reed" --dry-run
→ Would create snippet $testpage (page)
→ Would create schema bond: $testpage -> $hero
```

## Integration & Automation

### Pipe-Friendly Design
```bash
# Chain with standard Unix tools
reed snippet list --type "page" --format json | jq '.[] | select(.data.slug | contains("blog"))'

# Export/import workflows
reed export site "production" --format json | \
  reed import --site "staging" --update-existing

# Validation pipelines
reed validate site "production" --format json | \
  jq '.errors[] | select(.severity == "critical")'
```

### Configuration Files
```bash
# .reedconfig support
$ cat .reedconfig
site = "production"
format = "json"
confirm_deletes = true

# Override config per command
reed snippet create "page" --site "staging" /
  name $override_test /
  with title "Override Test"
```

## Performance Considerations

### Efficient Operations
```bash
# Registry-optimized queries
reed snippet list --routable    # Only queries routable meta-snippet types
reed snippet find --slug "about" # Uses slug index, routable types only

# Batch operations minimize database round-trips  
reed schema bond-many /
  parent $homepage /
  children $hero $content $footer /
  paths "home.hero" "home.content" "home.footer"

# Lazy loading for large hierarchies
reed schema tree $homepage --lazy --depth 2  # Only loads visible levels
```

### Memory Management
```bash
# Stream large exports
reed export site "large-site" --stream --chunk-size 1000 > backup.jsonl

# Pagination for large lists
reed snippet list --limit 50 --offset 100 --type "product"
```

This CLI API provides a clean, powerful interface that maintains the simplicity of the universal snippet system while exposing all the performance benefits of the registry layer and intelligent 4-layer routing.

---

**Next: ReedCMS-06-Templates.md** - Learn how Web Components and Tera work together to render content with optimal performance.
# ReedCMS-03-Standards.md

## Core Principle

**Zero Mental Translation** - One naming pattern used consistently across all system layers eliminates cognitive overhead and ensures KISS-brain development.

**4k Demo Scene Philosophy:** Every naming decision has a clear purpose and follows predictable transformation rules with zero exceptions.

## Universal Snippet Naming

### Primary Pattern: kebab-case Everywhere
```
snippet-name → ALL system representations derive from this
```

### Transformation Rules (Mandatory)

#### 1. Snippet Type Definition
```
Input: "hero-banner"
Usage: CLI, Registry, Database, File system
Rule: Always lowercase kebab-case, alphanumeric + hyphens only
```

#### 2. JavaScript Class Names
```
Transform: kebab-case → PascalCase + "Snippet" suffix
Example: "hero-banner" → "HeroBannerSnippet"
Rule: Split on hyphens, capitalize each word, append "Snippet"
```

#### 3. Web Component Tags  
```
Transform: Keep kebab-case exactly as snippet name
Example: "hero-banner" → <hero-banner>
Rule: No transformation - direct usage in HTML
```

#### 4. CSS Selectors
```
Transform: Keep kebab-case exactly as snippet name
Example: "hero-banner" → hero-banner { }
Rule: No transformation - direct CSS scoping
```

#### 5. File Naming
```
Transform: snippet-name + file extension
Example: "hero-banner" → hero-banner.tera, hero-banner.js, hero-banner.css
Rule: Exact snippet name as file prefix
```

#### 6. Semantic References
```
Transform: kebab-case → snake_case with $ prefix
Example: "hero-banner" → $hero_banner_instance
Rule: Replace hyphens with underscores, add $ prefix for CLI usage
```

## Implementation Functions

### Rust Transformation Functions
```rust
// Mandatory transformations - no variations allowed
pub fn snippet_to_js_class(snippet_name: &str) -> String {
    snippet_name
        .split('-')
        .map(|word| format!("{}{}", 
            word.chars().next().unwrap().to_uppercase(), 
            &word[1..].to_lowercase()))
        .collect::<String>() + "Snippet"
}

pub fn snippet_to_semantic_name(snippet_name: &str) -> String {
    format!("${}", snippet_name.replace('-', "_"))
}

pub fn validate_snippet_name(name: &str) -> Result<(), NamingError> {
    if !name.chars().all(|c| c.is_ascii_lowercase() || c.is_ascii_digit() || c == '-') {
        return Err(NamingError::InvalidCharacters);
    }
    if name.starts_with('-') || name.ends_with('-') || name.contains("--") {
        return Err(NamingError::InvalidHyphenUsage);
    }
    if name.len() < 2 || name.len() > 50 {
        return Err(NamingError::InvalidLength);
    }
    Ok(())
}
```

## Naming Examples Table

| Snippet Name | JS Class | Web Component | CSS Selector | Files | Semantic Ref |
|--------------|----------|---------------|--------------|-------|--------------|
| `hero-banner` | `HeroBannerSnippet` | `<hero-banner>` | `hero-banner` | `hero-banner.*` | `$hero_banner` |
| `text-with-image` | `TextWithImageSnippet` | `<text-with-image>` | `text-with-image` | `text-with-image.*` | `$text_with_image` |
| `call-to-action` | `CallToActionSnippet` | `<call-to-action>` | `call-to-action` | `call-to-action.*` | `$call_to_action` |
| `product-showcase` | `ProductShowcaseSnippet` | `<product-showcase>` | `product-showcase` | `product-showcase.*` | `$product_showcase` |
| `navigation-item` | `NavigationItemSnippet` | `<navigation-item>` | `navigation-item` | `navigation-item.*` | `$navigation_item` |

## Reserved Names

### Built-in Meta-Snippets (Cannot be overridden)
- `page` → `PageSnippet`
- `menu` → `MenuSnippet`  
- `navigation-item` → `NavigationItemSnippet`

### System Reserved Patterns
- No snippet names starting with `reed-*` (reserved for system components)
- No snippet names starting with `admin-*` (reserved for admin interface)
- No snippet names starting with `api-*` (reserved for API endpoints)

## File Structure Conventions

### Snippet Directory Structure (Mandatory)
```
themes/{theme-name}/snippets/{snippet-name}/
├── {snippet-name}.tera    # Template (required)
├── {snippet-name}.js      # JavaScript component (required)
├── {snippet-name}.css     # Styling (required)
└── {snippet-name}.json    # Field definition (auto-generated)
```

### Registry CSV Naming
```csv
# File: snippets.csv (always plural)
snippet_name,field_name,field_type,required,default_value
hero-banner,title,String,true,"Hero Title"
hero-banner,subtitle,String,false,""
```

### CLI Command Patterns
```bash
# Always use quoted snippet names in CLI
reed snippet create "hero-banner" name $welcome_hero
reed registry define "product-showcase" routable true
reed schema bond parent $homepage child $hero_banner
```

## Data Layer Standards

### CSV Field Naming
```csv
# Use snake_case for CSV column names
snippet_name,field_name,field_type,is_required,default_value
hero-banner,background_image,String,false,""
hero-banner,animation_duration,Number,false,300
```

### PostgreSQL Naming
```sql
-- Table names: snake_case
CREATE TABLE snippet_content (...);
CREATE TABLE user_sessions (...);

-- Column names: snake_case  
CREATE TABLE snippet_content (
    snippet_type VARCHAR(255),
    semantic_name VARCHAR(255),
    title_de_DE TEXT,
    created_at TIMESTAMP
);
```

### Redis Key Patterns
```redis
# Entity keys: type:entity_type:entity_id
HSET entity:snippet:homepage ...
HSET entity:menu:main_nav ...

# Association keys: assoc:path
HSET assoc:content.1.1 ...
HSET assoc:menu.1.2 ...

# Search keys: word:term
SADD word:modern ...
SADD word:cms ...

# Session keys: session:user_id  
HSET session:user123 ...
```

## Error Messages

### Consistent Error Terminology
```rust
#[derive(Debug)]
pub enum NamingError {
    InvalidCharacters,          // "Snippet name must contain only lowercase letters, numbers, and hyphens"
    InvalidHyphenUsage,         // "Hyphens cannot be at start, end, or consecutive"
    InvalidLength,              // "Snippet name must be 2-50 characters"
    ReservedName(String),       // "Snippet name 'reed-admin' is reserved for system use"
    AlreadyExists(String),      // "Snippet type 'hero-banner' already exists"
}

impl fmt::Display for NamingError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            NamingError::InvalidCharacters => 
                write!(f, "Snippet name must contain only lowercase letters, numbers, and hyphens"),
            NamingError::InvalidHyphenUsage => 
                write!(f, "Hyphens cannot be at start, end, or consecutive"),
            NamingError::InvalidLength => 
                write!(f, "Snippet name must be 2-50 characters"),
            NamingError::ReservedName(name) => 
                write!(f, "Snippet name '{}' is reserved for system use", name),
            NamingError::AlreadyExists(name) => 
                write!(f, "Snippet type '{}' already exists", name),
        }
    }
}
```

## Validation Rules

### Snippet Name Requirements
1. **Length:** 2-50 characters
2. **Characters:** Lowercase letters (a-z), numbers (0-9), hyphens (-) only
3. **Hyphens:** Not at start/end, not consecutive
4. **Reserved:** Cannot start with `reed-`, `admin-`, `api-`
5. **Uniqueness:** Must be unique across all snippet types in registry

### Auto-Generated Name Validation
```rust
// System validates all auto-generated names at startup
pub fn validate_system_consistency() -> Result<(), SystemError> {
    let registry = MetaSnippetRegistry::load()?;
    
    for snippet_type in registry.get_all_types() {
        // Verify all transformations are valid
        validate_snippet_name(&snippet_type)?;
        
        // Verify files exist with correct names
        verify_snippet_files_exist(&snippet_type)?;
        
        // Verify JS class name is valid identifier
        verify_js_class_name_valid(&snippet_to_js_class(&snippet_type))?;
    }
    
    Ok(())
}
```

## Language Standards

### Code and Technical Documentation
- **Language:** English only
- **Comments:** English, concise, explain WHY not WHAT
- **Variable names:** English, descriptive but concise
- **Function names:** Verb-noun pattern (`create_snippet`, `validate_name`)

### User-Facing Content
- **Primary Language:** German for ReedCMS project documentation
- **CLI Help:** German with English technical terms
- **Error Messages:** German for user errors, English for developer errors

## Code Style Standards

### Rust Code Standards
```rust
// Use explicit types for public APIs
pub fn create_snippet(name: &str, fields: &[FieldDefinition]) -> Result<Snippet, Error> {
    // Implementation
}

// Prefer explicit error handling
match result {
    Ok(value) => process_value(value),
    Err(e) => return Err(Error::ProcessingFailed(e)),
}

// Use descriptive variable names
let snippet_definition = registry.get_definition(&snippet_type)?;
let content_with_metadata = merge_content_and_metadata(content, metadata)?;
```

### JavaScript Code Standards  
```javascript
// Use PascalCase for classes
export class HeroBannerSnippet extends HTMLElement {
    // Use camelCase for methods and properties
    initializeInteractivity() {
        this.bindEventHandlers();
    }
    
    // Use descriptive method names
    bindEventHandlers() {
        // Implementation
    }
}
```

### CSS Standards
```css
/* Use snippet name as base selector */
hero-banner {
    /* Component-scoped styles */
}

/* Direct child targeting - no BEM needed */
hero-banner > .title {
    /* Child element styles */
}

hero-banner > .subtitle {
    /* Another child element */
}

/* Nested snippet styling */
hero-banner > content-section {
    /* Nested snippet styles */
}

hero-banner > content-section > .text-block {
    /* Deeply nested elements */
}

/* State modifiers with utility classes */
hero-banner.loading {
    /* Loading state */
}

hero-banner .hide {
    /* Utility class usage */
}
```

## Documentation Standards

### Code Comments
```rust
// Good: Explains WHY
// Validate snippet name to prevent Redis key conflicts
if !is_valid_redis_key(&snippet_name) {
    return Err(Error::InvalidSnippetName);
}

// Bad: Explains WHAT (obvious from code)
// Check if snippet name is valid
if !is_valid_snippet_name(&snippet_name) {
    return Err(Error::InvalidSnippetName);
}
```

### Function Documentation
```rust
/// Creates a new snippet instance with validated content.
/// 
/// # Arguments
/// * `snippet_type` - Must be registered in meta-snippet registry
/// * `content` - Field values matching snippet type definition
/// 
/// # Returns
/// * `Ok(Snippet)` - Successfully created snippet with generated ID
/// * `Err(Error::UnknownSnippetType)` - Snippet type not in registry
/// * `Err(Error::ValidationFailed)` - Content doesn't match schema
/// 
/// # Example
/// ```rust
/// let snippet = create_snippet("hero-banner", &content_fields)?;
/// ```
pub fn create_snippet(snippet_type: &str, content: &ContentFields) -> Result<Snippet, Error> {
    // Implementation
}
```

## Performance Standards

### Memory Efficiency
- No unnecessary allocations in hot paths
- Prefer `&str` over `String` for temporary data
- Use `Vec::with_capacity()` when size is known
- Reuse buffers where possible

### Error Handling Standards
```rust
// Use specific error types, not generic strings
#[derive(Debug, thiserror::Error)]
pub enum SnippetError {
    #[error("Snippet type '{0}' not found in registry")]
    UnknownType(String),
    
    #[error("Field '{field}' is required but missing")]
    MissingRequiredField { field: String },
    
    #[error("Invalid field value for '{field}': {reason}")]
    InvalidFieldValue { field: String, reason: String },
}
```

## Testing Standards

### Unit Test Naming
```rust
// Pattern: test_[function]_[scenario]_[expected_result]
#[test]
fn test_create_snippet_with_valid_data_succeeds() {
    // Test implementation
}

#[test]
fn test_create_snippet_with_invalid_type_returns_error() {
    // Test implementation
}

#[test]
fn test_snippet_to_js_class_transforms_correctly() {
    assert_eq!(snippet_to_js_class("hero-banner"), "HeroBannerSnippet");
    assert_eq!(snippet_to_js_class("call-to-action"), "CallToActionSnippet");
}
```

These standards ensure consistency across all ReedCMS components and make the system predictable for developers at all levels.

---

**Next: ReedCMS-04-Registry.md** - Learn how to define and manage snippet types and their field schemas.
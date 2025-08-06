# Redis Key Naming Convention

## Overview

Konsistente Key-Naming-Convention für alle Redis-Operationen. Klare Struktur für UCG, Cache und Session-Daten.

## Key Categories

```rust
/// Redis key builder with consistent naming
pub struct RedisKeyBuilder {
    prefix: String,
}

impl RedisKeyBuilder {
    pub fn new(prefix: &str) -> Self {
        Self {
            prefix: prefix.to_string(),
        }
    }
    
    /// Build full key with prefix
    fn build(&self, key: &str) -> String {
        format!("{}{}", self.prefix, key)
    }
}
```

## UCG Structure Keys (No TTL - Protected)

```rust
impl RedisKeyBuilder {
    /// Entity storage key
    /// Format: entity:{type}:{id}
    /// Example: entity:snippet:550e8400-e29b-41d4-a716-446655440000
    pub fn entity(&self, entity_type: &str, id: &Uuid) -> String {
        self.build(&format!("entity:{}:{}", entity_type, id))
    }
    
    /// Semantic name mapping
    /// Format: semantic:{type}:{name}
    /// Example: semantic:snippet:$welcome_hero
    pub fn semantic(&self, entity_type: &str, name: &str) -> String {
        self.build(&format!("semantic:{}:{}", entity_type, name))
    }
    
    /// Association storage
    /// Format: assoc:{path}
    /// Example: assoc:content.1.1
    pub fn association(&self, path: &str) -> String {
        self.build(&format!("assoc:{}", path))
    }
    
    /// Children set for entity
    /// Format: children:{parent_id}
    /// Example: children:550e8400-e29b-41d4-a716-446655440000
    pub fn children(&self, parent_id: &Uuid) -> String {
        self.build(&format!("children:{}", parent_id))
    }
    
    /// Parent mapping
    /// Format: parent:{child_id}
    /// Example: parent:550e8400-e29b-41d4-a716-446655440000
    pub fn parent(&self, child_id: &Uuid) -> String {
        self.build(&format!("parent:{}", child_id))
    }
}
```

## Search Index Keys (No TTL - Protected)

```rust
impl RedisKeyBuilder {
    /// Word index for search
    /// Format: word:{word}
    /// Example: word:modern
    pub fn word_index(&self, word: &str) -> String {
        self.build(&format!("word:{}", word.to_lowercase()))
    }
    
    /// Entity words for reverse lookup
    /// Format: entity_words:{entity_id}
    /// Example: entity_words:550e8400-e29b-41d4-a716-446655440000
    pub fn entity_words(&self, entity_id: &Uuid) -> String {
        self.build(&format!("entity_words:{}", entity_id))
    }
    
    /// Type index
    /// Format: type_index:{entity_type}
    /// Example: type_index:snippet
    pub fn type_index(&self, entity_type: &str) -> String {
        self.build(&format!("type_index:{}", entity_type))
    }
    
    /// Status index
    /// Format: status_index:{status}
    /// Example: status_index:published
    pub fn status_index(&self, status: &str) -> String {
        self.build(&format!("status_index:{}", status))
    }
}
```

## Cache Keys (With TTL - Evictable)

```rust
impl RedisKeyBuilder {
    /// Page cache
    /// Format: page:cache:{path}:{locale}:{theme}
    /// Example: page:cache:/about:de_DE:corporate.berlin
    pub fn page_cache(&self, path: &str, locale: &str, theme: &str) -> String {
        self.build(&format!("page:cache:{}:{}:{}", 
            path.replace('/', "_"), locale, theme))
    }
    
    /// Template cache
    /// Format: template:cache:{name}:{theme}
    /// Example: template:cache:hero-banner:corporate.berlin
    pub fn template_cache(&self, name: &str, theme: &str) -> String {
        self.build(&format!("template:cache:{}:{}", name, theme))
    }
    
    /// Asset bundle cache
    /// Format: asset:bundle:{type}:{hash}
    /// Example: asset:bundle:css:a1b2c3d4
    pub fn asset_bundle(&self, bundle_type: &str, hash: &str) -> String {
        self.build(&format!("asset:bundle:{}:{}", bundle_type, hash))
    }
    
    /// Query result cache
    /// Format: query:cache:{query_hash}
    /// Example: query:cache:5d41402abc4b2a76b9719d911017c592
    pub fn query_cache(&self, query_hash: &str) -> String {
        self.build(&format!("query:cache:{}", query_hash))
    }
}
```

## Session Keys (With TTL - Evictable)

```rust
impl RedisKeyBuilder {
    /// User session
    /// Format: session:{session_id}
    /// Example: session:abc123def456
    pub fn session(&self, session_id: &str) -> String {
        self.build(&format!("session:{}", session_id))
    }
    
    /// CSRF token
    /// Format: csrf:{token}
    /// Example: csrf:xyz789ghi012
    pub fn csrf_token(&self, token: &str) -> String {
        self.build(&format!("csrf:{}", token))
    }
    
    /// Rate limit tracking
    /// Format: rate:{action}:{identifier}
    /// Example: rate:login:192.168.1.1
    pub fn rate_limit(&self, action: &str, identifier: &str) -> String {
        self.build(&format!("rate:{}:{}", action, identifier))
    }
}
```

## Temporary Keys (With Short TTL)

```rust
impl RedisKeyBuilder {
    /// Lock for distributed operations
    /// Format: lock:{resource}
    /// Example: lock:snippet:550e8400-e29b-41d4-a716-446655440000
    pub fn lock(&self, resource: &str) -> String {
        self.build(&format!("lock:{}", resource))
    }
    
    /// Task queue
    /// Format: queue:{queue_name}
    /// Example: queue:indexing
    pub fn queue(&self, queue_name: &str) -> String {
        self.build(&format!("queue:{}", queue_name))
    }
    
    /// Temporary computation result
    /// Format: temp:{operation}:{id}
    /// Example: temp:export:job123
    pub fn temp(&self, operation: &str, id: &str) -> String {
        self.build(&format!("temp:{}:{}", operation, id))
    }
}
```

## Key Patterns for Operations

```rust
/// Common key patterns for batch operations
pub struct RedisKeyPatterns;

impl RedisKeyPatterns {
    /// Pattern for all entities of a type
    pub fn entities_by_type(entity_type: &str) -> String {
        format!("entity:{}:*", entity_type)
    }
    
    /// Pattern for all cache entries
    pub fn all_cache() -> String {
        "page:cache:*".to_string()
    }
    
    /// Pattern for all sessions
    pub fn all_sessions() -> String {
        "session:*".to_string()
    }
    
    /// Pattern for UCG structure
    pub fn ucg_structure() -> String {
        "entity:*".to_string()
    }
    
    /// Pattern for search indexes
    pub fn search_indexes() -> String {
        "word:*".to_string()
    }
}
```

## TTL Management

```rust
/// TTL constants for different key types
pub struct RedisTTL;

impl RedisTTL {
    /// Page cache TTL (5 minutes)
    pub const PAGE_CACHE: Duration = Duration::from_secs(300);
    
    /// Template cache TTL (15 minutes)
    pub const TEMPLATE_CACHE: Duration = Duration::from_secs(900);
    
    /// Session TTL (24 hours)
    pub const SESSION: Duration = Duration::from_secs(86400);
    
    /// CSRF token TTL (1 hour)
    pub const CSRF_TOKEN: Duration = Duration::from_secs(3600);
    
    /// Rate limit window (15 minutes)
    pub const RATE_LIMIT: Duration = Duration::from_secs(900);
    
    /// Lock TTL (30 seconds)
    pub const LOCK: Duration = Duration::from_secs(30);
    
    /// Temporary data TTL (10 minutes)
    pub const TEMP: Duration = Duration::from_secs(600);
}
```

## Key Usage Examples

```rust
/// Example usage of key builder
pub async fn example_key_usage(redis: &RedisManager) -> Result<()> {
    let keys = RedisKeyBuilder::new("reed:");
    
    // Store entity (no TTL - protected)
    let entity_key = keys.entity("snippet", &Uuid::new_v4());
    redis.hset(&entity_key, "type", "hero-banner").await?;
    redis.hset(&entity_key, "title", "Welcome").await?;
    
    // Store cache (with TTL - evictable)
    let cache_key = keys.page_cache("/", "de_DE", "corporate");
    redis.set(&cache_key, "<html>...</html>", Some(RedisTTL::PAGE_CACHE)).await?;
    
    // Search index (no TTL - protected)
    let word_key = keys.word_index("welcome");
    redis.sadd(&word_key, &[&entity_key]).await?;
    
    // Session (with TTL - evictable)
    let session_key = keys.session("abc123");
    redis.set(&session_key, r#"{"user_id":"123"}"#, Some(RedisTTL::SESSION)).await?;
    
    Ok(())
}
```

## Key Documentation

```rust
/// Generate key documentation for debugging
pub fn document_keys() -> String {
    let mut doc = String::new();
    
    doc.push_str("# Redis Key Structure\n\n");
    
    doc.push_str("## Protected Keys (No TTL)\n");
    doc.push_str("- entity:{type}:{id} - Entity data\n");
    doc.push_str("- semantic:{type}:{name} - Name mappings\n");
    doc.push_str("- assoc:{path} - Associations\n");
    doc.push_str("- children:{id} - Child relationships\n");
    doc.push_str("- word:{word} - Search index\n\n");
    
    doc.push_str("## Cache Keys (With TTL)\n");
    doc.push_str("- page:cache:{path}:{locale}:{theme} - Rendered pages\n");
    doc.push_str("- template:cache:{name}:{theme} - Compiled templates\n");
    doc.push_str("- session:{id} - User sessions\n");
    doc.push_str("- rate:{action}:{id} - Rate limiting\n");
    
    doc
}
```

## Summary

Diese Key-Naming-Convention bietet:
- **Klare Kategorisierung** zwischen UCG und Cache
- **TTL-basierte Protection** durch Key-Struktur
- **Effiziente Patterns** für Batch-Operationen
- **Konsistente Namensgebung** im gesamten System
- **Selbstdokumentierende Keys** für Debugging

Jeder Key hat einen klaren Zweck und Lifecycle.
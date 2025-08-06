# ReedCMS-02-Architecture.md

## Architecture Philosophy

**Anti-Bloat by Design:** ReedCMS prevents the typical database explosion that plagues traditional CMSs through a carefully designed multi-layer architecture where each layer serves a specific, focused purpose.

**KISS-Brain Principle:** Every data operation should be mentally predictable - developers always know where data lives and how it flows.

## System Overview

ReedCMS uses a **4-layer data architecture** that prevents feature creep and maintains performance:

```
┌─────────────────────────────────────────────────────────┐
│                   CSV Layer                             │
│   Source of Truth: Schema + Configuration              │
│   Git-versionable, Human-readable, Deployment-friendly │ 
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL Main Schema                     │
│   Live Data: Content + Users + Business Logic          │
│   ACID transactions, i18n columns, Performance         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL UCG Schema                      │
│   Structure Backup: Associations + Schema cache        │
│   Compressed, Recovery-only, Synced from CSV           │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                   Redis Layer                           │
│   Performance: Search Index + Sessions + UCG Cache     │
│   Sub-millisecond queries, Ephemeral data              │
└─────────────────────────────────────────────────────────┘
```

## Data Layers Detail

### CSV Layer - Configuration Source of Truth

**Purpose:** Schema definitions, site structure templates, i18n labels
**Characteristics:** Static, rarely changed, Git-versionable, human-readable

```csv
# snippets.csv - Snippet type definitions
snippet_name,field_name,field_type,required,default_value
hero-banner,title,String,true,"Default Title"
hero-banner,subtitle,String,false,""
accordeon,items,Array,"title:String;content:String;open:Boolean",true,"[]"

# associations.csv - Site structure templates  
parent_path,entity_type,entity_id,weight,path
,page,homepage,0,content.1
homepage,hero-banner,welcome_hero,0,content.1.1
homepage,content-section,main_content,1,content.1.2

# translations.csv - i18n labels
snippet_name,field_name,label_de_DE,label_en_US
hero-banner,title,"Titel","Title"
hero-banner,subtitle,"Untertitel","Subtitle"

# search-words.csv - Search index definitions
word,content_ids
modern,"snippet:hero1,snippet:about2,snippet:tech3"
cms,"snippet:hero1,snippet:features4"
rust,"snippet:tech3,snippet:performance5"
```

**Growth Pattern:** Logarithmic - new snippet types and vocabulary grow slowly
**Size:** ~50KB for 1,000 snippets, ~800KB for 100,000 snippets

### PostgreSQL Main Schema - Live Business Data

**Purpose:** Content, users, commerce data, anything that changes frequently
**Characteristics:** ACID transactions, high performance, focused schema

```sql
-- Content with i18n columns
CREATE TABLE snippet_content (
    id UUID PRIMARY KEY,
    snippet_type VARCHAR(255) NOT NULL,
    semantic_name VARCHAR(255),
    
    -- i18n content columns (automatically added per language)
    title_de_DE TEXT,
    title_en_US TEXT,
    content_de_DE TEXT,
    content_en_US TEXT,
    meta_description_de_DE TEXT,
    meta_description_en_US TEXT,
    
    -- Flexible additional data
    additional_data JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- User management
CREATE TABLE users (
    id UUID PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_admin BOOLEAN DEFAULT FALSE,
    last_login TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Content editing locks
CREATE TABLE content_locks (
    snippet_id UUID PRIMARY KEY REFERENCES snippet_content(id),
    locked_by UUID REFERENCES users(id),
    locked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

**Anti-Bloat Design:** No UCG pollution, no search tables, no configuration mixing

### PostgreSQL UCG Schema - Structure Backup

**Purpose:** Backup and recovery for CSV-defined structure
**Characteristics:** Compressed, read-only during normal operation, sync target

```sql
-- UCG structure backup (synced from CSV)
CREATE TABLE ucg_associations (
    id UUID PRIMARY KEY,
    parent_id UUID,
    entity_type VARCHAR(255),
    entity_id UUID,
    path TEXT,
    weight INTEGER,
    data JSONB,
    synced_from_csv_at TIMESTAMP WITH TIME ZONE,
    csv_checksum VARCHAR(64)
) WITH (compression = lz4);

-- Schema definitions backup (synced from CSV)
CREATE TABLE ucg_snippet_definitions (
    snippet_name VARCHAR(255) PRIMARY KEY,
    fields JSONB,
    synced_from_csv_at TIMESTAMP WITH TIME ZONE,
    csv_checksum VARCHAR(64)
) WITH (compression = lz4);
```

**Compression Ratio:** ~80-90% for repetitive CSV data
**Usage:** Recovery scenarios, database imports, consistency verification

### Redis Layer - Performance & Ephemeral Data

**Purpose:** Search index, sessions, UCG performance cache
**Characteristics:** Sub-millisecond queries, memory-resident, rebuildable

```redis
# Search index (for searchable snippet types only)
SADD word:modern "snippet:hero1" "snippet:about2" "snippet:tech3"
SADD word:cms "snippet:hero1" "snippet:features4"
SADD word:rust "snippet:tech3" "snippet:performance5"

# UCG structure cache (rebuilt from CSV on startup)
HSET entity:snippet:homepage type "page" data '{"title":"Home","slug":"home"}'
SADD children:content.1 "content.1.1" "content.1.2" "content.1.3"

# User sessions (ephemeral)
HSET session:user123 "logged_in_at" "2025-08-04T10:30:00Z"
HSET session:user123 "cart_id" "cart456"

# Content locks cache (mirrors PostgreSQL)
SETEX lock:snippet:hero1 1800 "user123"
```

## Universal Content Graph Pattern

### Core Data Structure

Everything in ReedCMS is either an **Entity** or an **Association**:

```rust
// Universal entity - can be snippet, registry definition, menu, category
pub struct Entity {
    pub id: String,                    // UUID for internal operations
    pub entity_type: String,           // "snippet", "registry", "menu", "category"
    pub semantic_name: Option<String>, // Human-readable ($homepage, $main_nav)
    pub data: serde_json::Value,       // Type-specific data payload
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Universal association - single relationship pattern for all use cases
pub struct Association {
    pub id: Uuid,
    pub parent_id: Option<Uuid>,       // Null = root level
    pub entity_type: String,           // Type of child entity
    pub entity_id: String,             // Reference to child entity
    pub weight: i32,                   // Drupal-style ordering (lower first)
    pub depth: u8,                     // Performance optimization cache
    pub path: String,                  // Materialized path for fast queries
    pub created_at: DateTime<Utc>,
}
```

### Universal Patterns in Action

**Content Hierarchy:**
```redis
# Page with nested content structure
HSET entity:snippet:homepage type "page" data '{"title":"Home","slug":"home"}'
HSET assoc:content.1 parent_id "null" entity_type "snippet" entity_id "homepage" weight 0 depth 0 path "content.1"

# Child snippets in weighted order
HSET assoc:content.1.1 parent_id "homepage" entity_type "snippet" entity_id "hero-banner" weight 0 depth 1 path "content.1.1"
HSET assoc:content.1.2 parent_id "homepage" entity_type "snippet" entity_id "content-section" weight 10 depth 1 path "content.1.2"
```

**Menu Structures:**
```redis
# Navigation menu as entity graph
HSET entity:menu:main-nav type "menu" data '{"name":"Main Navigation"}'
HSET assoc:menu.1 parent_id "null" entity_type "menu" entity_id "main-nav" weight 0 depth 0 path "menu.1"

# Menu items with nested submenus
HSET entity:menu-item:about type "menu-item" data '{"label":"About","url":"/about"}'
HSET assoc:menu.1.1 parent_id "main-nav" entity_type "menu-item" entity_id "about" weight 10 depth 1 path "menu.1.1"
```

**Search Integration:**
```redis
# Search words become entities in the graph
SADD word:modern "snippet:homepage" "snippet:about" "snippet:tech-stack"
SADD word:rust "snippet:homepage" "snippet:architecture"

# Word positions for relevance scoring
ZADD entity:snippet:homepage:words 1 "welcome" 3 "modern" 7 "rust" 9 "cms"
```

## Data Flows

### Configuration Flow (Rare, Administrative)

```
CLI/Admin Panel → CSV Files → PostgreSQL UCG Schema (background)
```

**Trigger:** Schema changes, new snippet types, site restructuring
**Frequency:** Weekly to monthly
**Performance:** Can be slow (background sync)

### Content Flow (Frequent, User-facing)

```
CLI/Admin Panel → PostgreSQL Main Schema (immediate)
```

**Trigger:** Content creation, user updates, business transactions
**Frequency:** Continuous
**Performance:** Must be fast (1-2ms)

### Search Index Flow (Background, Intelligent)

```
PostgreSQL Main Schema → Redis Search Index (background, selective)
```

**Trigger:** Content changes on searchable snippet types
**Frequency:** Continuous background
**Performance:** Non-blocking, gentle

## Performance Characteristics

### Query Performance by Type

```
UCG Structure Queries:    0.1ms   (Redis key lookups)
Content Queries:          1-2ms   (PostgreSQL, no JOINs)
Search Queries:           0.5ms   (Redis set operations)
Combined Queries:         1.5ms   (Redis + PostgreSQL optimized)
```

### Memory Usage

```
Redis Cache:              ~10MB typical site
PostgreSQL Connections:   ~5MB per connection  
CSV Cache:                ~1MB loaded in memory
Total Application:        ~50MB including Rust runtime
```

### Storage Requirements

```
Small Site (1K snippets):
- CSV files: ~50KB
- PostgreSQL: ~15MB  
- Redis: ~5MB
- Total: ~20MB

Large Site (100K snippets):  
- CSV files: ~800KB
- PostgreSQL: ~1.2GB
- Redis: ~100MB  
- Total: ~1.3GB
```

### Scaling Properties

```
Entity Count    Redis Memory    Query Speed    CLI Response
1,000          15MB            <1ms           <5ms
10,000         125MB           <5ms           <15ms
100,000        1.2GB           <25ms          <100ms
1,000,000      12GB            <100ms         <500ms

# Linear scaling with no performance cliffs
# No query optimization needed - Redis handles everything
```

## Anti-Bloat Design Benefits

### Traditional CMS Evolution (Prevented)

Without layer separation, typical CMS horror story:

```sql
-- Year 1: Simple start
CREATE TABLE content (...);  -- 10 columns

-- Year 2: Feature creep begins  
ALTER TABLE content ADD COLUMN workflow_state_id UUID;
CREATE TABLE content_permissions (...);
CREATE TABLE content_tags (...);
-- 25 tables total

-- Year 3: Enterprise requirements
CREATE TABLE content_audit_logs (...);
CREATE TABLE content_versions (...);  
CREATE TABLE content_relations (...);
-- 50 tables total

-- Year 4: Performance crisis
-- Queries with 15+ JOINs taking 500ms
-- Need caching layer, search engine, CDN
-- 100+ tables, architecture unrecognizable
```

### ReedCMS Anti-Bloat Measures

**Layer Isolation:** Each concern stays in its optimal layer
```
- UCG bloat → Impossible (CSV-constrained)
- Search bloat → Impossible (Redis-ephemeral)  
- Config bloat → Impossible (CSV-readable)
- Content bloat → Minimized (focused schema)
```

**Predictable Performance:** No query optimization guesswork
```
UCG queries:      0.1ms (Redis)
Content queries:  1-2ms (PostgreSQL, no JOINs)
Search queries:   0.5ms (Redis sets)
Config queries:   N/A (CSV preloaded)
```

## System Recovery Scenarios

### CSV Corruption Recovery
```bash
$ reed recover csv-from-postgres
✓ Restored snippets.csv (47 snippet types)
✓ Restored associations.csv (1,247 associations)  
✓ Verified checksums - all CSV files recovered
```

### Complete System Recovery
```bash
$ reed recover full-restore /
  from-postgres-backup "site-backup-20250804.sql" /
  warm-cache true

✓ Restored PostgreSQL main + UCG schemas  
✓ Generated CSV files from UCG backup
✓ Warmed Redis cache from CSV + content
✓ Full system restored - ready to serve
```

## Implementation Guidelines

### Database Facade Pattern
```rust
pub struct DatabaseFacade {
    postgres_main: PostgresClient,
    postgres_ucg: PostgresClient,
    redis: RedisClient,
    csv_store: CsvStore,
}

impl DatabaseFacade {
    // Unified API hides layer complexity
    pub async fn create_snippet(&self, spec: &SnippetSpec) -> Result<String, Error> {
        // Route to appropriate layers
        let content_id = self.postgres_main.insert_content(spec).await?;
        
        // Background search index update (if searchable)
        if self.is_searchable(&spec.snippet_type) {
            tokio::spawn(async move {
                self.update_search_index(&content_id).await.ok();
            });
        }
        
        Ok(content_id)
    }
}
```

### Smart Query Routing
```rust
impl DatabaseFacade {
    pub async fn execute_query(&self, query: &ParsedQuery) -> Result<QueryResult, Error> {
        match query.query_type {
            QueryType::Structure => {
                // Pure UCG → Redis only (0.1ms)
                self.redis.execute_structure_query(query).await
            },
            QueryType::Content => {
                // Pure content → PostgreSQL only (1ms)
                self.postgres_main.execute_content_query(query).await
            },
            QueryType::Search => {
                // Search → Redis search index (0.5ms)
                self.redis.execute_search_query(query).await
            },
            QueryType::Combined => {
                // Multi-layer → optimized combination (1.5ms total)
                let structure = self.redis.get_structure(query).await?;
                let content = self.postgres_main
                    .get_content_for_structure(&structure, query)
                    .await?;
                Ok(self.merge_results(structure, content))
            }
        }
    }
}
```

This architecture ensures ReedCMS remains performant, maintainable, and bloat-free while providing all the functionality of enterprise CMSs through intelligent layer separation and focused responsibilities.

---

**Next: ReedCMS-03-Standards.md** - Learn the naming conventions and coding standards that keep the system consistent and KISS-brain friendly.
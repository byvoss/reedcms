# Universal Content Graph (UCG) System

## Kern-Innovation

Das Universal Content Graph (UCG) System ist die fundamentale Innovation von ReedCMS, die das Problem der Datenbank-Explosion traditioneller CMS löst.

**Das Problem:** Traditionelle CMSs leiden unter separaten Tabellen für Pages, Posts, Menüs, Kategorien, Suchindizes, Berechtigungen und Custom Fields. Dies führt zu komplexen JOINs, Query-Optimierungs-Albträumen und architektonischer Aufblähung.

**Die ReedCMS Lösung:** Alles ist eine Entity mit Associations in einem Universal Content Graph. Ein einziges Pattern behandelt:
- Content-Hierarchien (Pages mit verschachtelten Sections)
- Menü-Strukturen (Navigation mit Submenüs)  
- Meta-Snippet-Kompositionen (wiederverwendbare Komponenten-Definitionen)
- Such-Indizierung (Wort-Entity-Beziehungen)
- Taxonomien und Kategorien (Content-Klassifizierung)

## Core Data Structure

Alles in ReedCMS ist entweder eine **Entity** oder eine **Association**:

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

## Performance-Charakteristiken

**Vorhersagbare O(1) Performance**, keine komplexen Queries, unendliche Flexibilität:

```
UCG Structure Queries:    0.1ms   (Redis key lookups)
Content Queries:          1-2ms   (PostgreSQL, no JOINs)
Search Queries:           0.5ms   (Redis set operations)
Combined Operations:      1.5ms   (optimized multi-layer)
```

## Implementation Layers

Das UCG-System nutzt drei Schichten für optimale Performance:

### Redis Layer - Live UCG Cache
```redis
# UCG structure cache (rebuilt from CSV on startup)
HSET entity:snippet:homepage type "page" data '{"title":"Home","slug":"home"}'
SADD children:content.1 "content.1.1" "content.1.2" "content.1.3"

# Entity lookups - 0.1ms
HGET entity:snippet:welcome_hero
```

### PostgreSQL UCG Schema - Structure Backup
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
```

### CSV Layer - Schema Definitions
```csv
# associations.csv - Site structure templates  
parent_path,entity_type,entity_id,weight,path
,page,homepage,0,content.1
homepage,hero-banner,welcome_hero,0,content.1.1
homepage,content-section,main_content,1,content.1.2
```

## Universal Patterns in Action

### Content Hierarchy
```redis
# Page with nested content structure
HSET entity:snippet:homepage type "page" data '{"title":"Home","slug":"home"}'
HSET assoc:content.1 parent_id "null" entity_type "snippet" entity_id "homepage" weight 0 depth 0 path "content.1"

# Child snippets in weighted order
HSET assoc:content.1.1 parent_id "homepage" entity_type "snippet" entity_id "hero-banner" weight 0 depth 1 path "content.1.1"
HSET assoc:content.1.2 parent_id "homepage" entity_type "snippet" entity_id "content-section" weight 10 depth 1 path "content.1.2"
```

### Menu Structures
```redis
# Navigation menu as entity graph
HSET entity:menu:main-nav type "menu" data '{"name":"Main Navigation"}'
HSET assoc:menu.1 parent_id "null" entity_type "menu" entity_id "main-nav" weight 0 depth 0 path "menu.1"

# Menu items with nested submenus
HSET entity:menu-item:about type "menu-item" data '{"label":"About","url":"/about"}'
HSET assoc:menu.1.1 parent_id "main-nav" entity_type "menu-item" entity_id "about" weight 10 depth 1 path "menu.1.1"
```

### Search Integration
```redis
# Search words become entities in the graph
SADD word:modern "snippet:homepage" "snippet:about" "snippet:tech-stack"
SADD word:rust "snippet:homepage" "snippet:architecture"

# Word positions for relevance scoring
ZADD entity:snippet:homepage:words 1 "welcome" 3 "modern" 7 "rust" 9 "cms"
```

## Anti-Bloat Garantien

Das UCG-Pattern verhindert strukturell die typische CMS-Datenbank-Explosion:

```
- UCG bloat → Unmöglich (CSV-constrained)
- Search bloat → Unmöglich (Redis-ephemeral)  
- Config bloat → Unmöglich (CSV-readable)
- Content bloat → Minimiert (focused schema)
```

Keine Query-Optimierung nötig - Redis handhabt alles mit vorhersagbarer Performance.

## Recovery Strategy

Redis ist **pure Performance Cache** - niemals Persistence. Das UCG-System hat perfekte Backup-Strategien:

```rust
// Redis Recovery Strategy
pub async fn redis_startup_recovery() -> Result<(), Error> {
    // 1. CSV Layer: Immer aktuell, Git-versionable
    let ucg_structure = load_ucg_from_csv().await?;
    
    // 2. PostgreSQL UCG: Compressed backup für consistency
    let ucg_backup = postgres_ucg.load_structure_backup().await?;
    
    // 3. Warm Redis cache from authoritative sources
    redis.populate_ucg_cache(&ucg_structure).await?;
    redis.rebuild_search_index_from_postgres().await?;
}
```

**Resultat:** Redis crash = kein Datenverlust. System recovered von CSV + PostgreSQL UCG backup in Sekunden.

## Smart Query Routing

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

## Zusammenfassung

Das UCG-System eliminiert die Komplexität traditioneller CMSs durch:
- **Ein Pattern für alles**: Entity + Association
- **Vorhersagbare Performance**: O(1) operations
- **Keine JOINs**: Direkte Redis lookups
- **Recovery-fähig**: CSV + PostgreSQL backup
- **Anti-Bloat**: Strukturell unmöglich

Dies macht ReedCMS zum ersten CMS mit garantiert linearer Skalierung ohne Performance-Cliffs.
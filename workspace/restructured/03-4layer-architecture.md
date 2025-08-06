# ReedCMS 4-Layer Architecture

## Anti-Bloat by Design

ReedCMS verhindert die typische Datenbank-Explosion traditioneller CMS durch eine sorgfältig gestaltete Multi-Layer-Architektur, in der jede Schicht einen spezifischen, fokussierten Zweck erfüllt.

**KISS-Brain Prinzip:** Jede Datenoperation soll mental vorhersehbar sein - Entwickler wissen immer, wo Daten leben und wie sie fließen.

## System-Übersicht

ReedCMS nutzt eine **4-Layer Datenarchitektur**, die Feature Creep verhindert und Performance erhält:

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

## Detaillierte Layer-Beschreibung

### CSV Layer - Configuration Source of Truth

**Zweck:** Schema-Definitionen, Site-Struktur-Templates, i18n Labels

**Dateien:**
```bash
snippets.csv            # Snippet definitions & validation
associations.csv        # UCG entity relationships  
fields.csv             # Field specifications
translations.{locale}.csv # Translation strings per language
```

**Eigenschaften:**
- Git-versionierbar
- Human-readable
- Deployment-freundlich
- Source of Truth für alle Struktur-Definitionen

**Größe:** ~50KB für 1.000 Snippets, ~800KB für 100.000 Snippets

### PostgreSQL Main Schema - Live Business Data

**Zweck:** Content, Users, Commerce-Daten, alles was sich häufig ändert

**Schema Design:**
```sql
-- Flat content tables with i18n columns
CREATE TABLE snippet_content (
    id UUID PRIMARY KEY,
    snippet_name VARCHAR(255) NOT NULL,
    title_de_DE TEXT,
    title_en_US TEXT,
    title_fr_FR TEXT,
    content_de_DE JSONB,
    content_en_US JSONB,
    content_fr_FR JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Anti-Flatterhaft Regel:** "Je flatter die PostgreSQL, um so weniger flatterhaft ist ihr Verhalten"

**Anti-Bloat Design:** Keine UCG-Verschmutzung, keine Suchtabellen, keine Konfigurations-Vermischung

### PostgreSQL UCG Schema - Structure Backup

**Zweck:** Backup und Recovery für CSV-definierte Struktur
**Eigenschaften:** Komprimiert, read-only während normalem Betrieb, Sync-Ziel

```sql
-- UCG structure backup (synced from CSV)
CREATE TABLE ucg_associations (
    id UUID PRIMARY KEY,
    entity_id VARCHAR(255) NOT NULL,
    parent_id VARCHAR(255),
    child_ids TEXT[], 
    association_type VARCHAR(50),
    metadata JSONB,
    checksum VARCHAR(64)
) WITH (compression = lz4);

-- Schema definitions backup (synced from CSV)
CREATE TABLE ucg_snippet_definitions (
    snippet_name VARCHAR(255) PRIMARY KEY,
    fields JSONB,
    validation_rules JSONB,
    checksum VARCHAR(64)
) WITH (compression = lz4);
```

**Kompressionsrate:** ~80-90% für repetitive CSV-Daten
**Nutzung:** Recovery-Szenarien, Datenbank-Importe, Konsistenz-Verifikation

### Redis Layer - Performance & Ephemeral Data

**Zweck:** Search Index, Sessions, UCG Performance Cache

```bash
# Search index (rebuilt from PostgreSQL content)
SADD keyword:rust "snippet:tech3" "snippet:performance5"
SADD word:rust "snippet:tech3" "snippet:performance5"

# UCG structure cache (rebuilt from CSV on startup)
HSET entity:snippet:homepage type "page" data '{"title":"Home","slug":"home"}'
SADD children:content.1 "content.1.1" "content.1.2" "content.1.3"

# Session storage
SET session:abc123 '{"user_id":"user1","locale":"de_DE"}' EX 3600
```

**Prinzip:** Redis ist **pure Performance Cache** - niemals Persistence

## Data Flow Patterns

### Schema Updates (CSV → PostgreSQL UCG)
```
CLI/Admin Panel → CSV Files → PostgreSQL UCG Schema (background)
```

### Content Operations (Direct → PostgreSQL Main)
```
CLI/Admin Panel → PostgreSQL Main Schema (immediate)
```

### Search Index Updates (PostgreSQL → Redis)
```
PostgreSQL Main Schema → Redis Search Index (background, selective)
```

## Performance-Charakteristiken

```
UCG Structure Queries:    0.1ms   (Redis key lookups)
Content Queries:          1-2ms   (PostgreSQL, no JOINs)
Search Queries:           0.5ms   (Redis set operations)
Combined Queries:         1.5ms   (Redis + PostgreSQL optimized)
```

## Memory Usage

```
Redis Cache:              ~10MB typical site
PostgreSQL Connections:   ~5MB per connection  
CSV Cache:                ~1MB loaded in memory
Total Application:        ~50MB including Rust runtime
```

## Skalierungs-Profile

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

## Anti-Bloat Maßnahmen

**Layer-Isolation:** Jede Anforderung bleibt in ihrer optimalen Schicht
```
- UCG bloat → Unmöglich (CSV-begrenzt)
- Search bloat → Unmöglich (Redis-ephemeral)  
- Config bloat → Unmöglich (CSV-lesbar)
- Content bloat → Minimiert (fokussiertes Schema)
```

**Vorhersagbare Performance:** Kein Query-Optimierungs-Ratespiel
```
UCG queries:      0.1ms (Redis)
Content queries:  1-2ms (PostgreSQL, no JOINs)
Search queries:   0.5ms (Redis sets)
Config queries:   N/A (CSV preloaded)
```

## System Recovery Szenarien

### CSV Corruption Recovery
```bash
$ reed recover csv-from-postgres
✓ Restored snippets.csv (47 snippet types)
✓ Restored associations.csv (1,247 associations)  
✓ Verified checksums - all CSV files recovered
```

### Full System Recovery
```bash
$ reed recover full-system --from-backup backup.tar.gz
✓ Extracted backup archive
✓ Restored PostgreSQL main + UCG schemas  
✓ Generated CSV files from UCG backup
✓ Warmed Redis cache from CSV + content
✓ Full system restored - ready to serve
```

## Implementation Interface

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
        
        // Update search index in background
        tokio::spawn(async move {
            self.redis.update_search_index(content_id).await
        });
        
        Ok(content_id)
    }
}
```

Diese Architektur stellt sicher, dass ReedCMS performant, wartbar und bloat-frei bleibt, während es alle Funktionalitäten von Enterprise-CMS durch intelligente Layer-Trennung und fokussierte Verantwortlichkeiten bereitstellt.
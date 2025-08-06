# Database Schema Definition

## Overview

PostgreSQL Schema für Main und UCG Datenbanken. Flaches Schema-Design für optimale Performance.

## Main Database Schema

```sql
-- Main database stores actual content
-- Keep it flat for performance (no complex JOINs)

-- Snippet content storage
CREATE TABLE snippet_content (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snippet_type VARCHAR(100) NOT NULL,
    semantic_name VARCHAR(255),
    
    -- Locale-specific content fields (dynamic based on configured locales)
    title_de_DE TEXT,
    title_en_US TEXT,
    title_fr_FR TEXT,
    title_es_ES TEXT,
    title_eu TEXT,  -- Basque
    
    content_de_DE TEXT,
    content_en_US TEXT,
    content_fr_FR TEXT,
    content_es_ES TEXT,
    content_eu TEXT,
    
    -- JSON fields for flexible data
    fields_de_DE JSONB DEFAULT '{}',
    fields_en_US JSONB DEFAULT '{}',
    fields_fr_FR JSONB DEFAULT '{}',
    fields_es_ES JSONB DEFAULT '{}',
    fields_eu JSONB DEFAULT '{}',
    
    -- Metadata
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by VARCHAR(255) NOT NULL,
    updated_by VARCHAR(255) NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    
    -- Indexes for performance
    CONSTRAINT snippet_content_status_check 
        CHECK (status IN ('draft', 'published', 'archived', 'deleted'))
);

-- Indexes for fast lookups
CREATE INDEX idx_snippet_content_type ON snippet_content(snippet_type);
CREATE INDEX idx_snippet_content_semantic_name ON snippet_content(semantic_name);
CREATE INDEX idx_snippet_content_status ON snippet_content(status);
CREATE INDEX idx_snippet_content_created_at ON snippet_content(created_at);

-- Update trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_snippet_content_updated_at 
    BEFORE UPDATE ON snippet_content
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- User management (simple and flat)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    
    -- User details
    display_name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'editor',
    active BOOLEAN NOT NULL DEFAULT true,
    
    -- Metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,
    
    CONSTRAINT users_role_check 
        CHECK (role IN ('admin', 'editor', 'viewer'))
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);

CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- Session storage (for authentication)
CREATE TABLE sessions (
    id VARCHAR(255) PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    data JSONB NOT NULL DEFAULT '{}',
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);

-- Cleanup expired sessions periodically
CREATE OR REPLACE FUNCTION cleanup_expired_sessions()
RETURNS void AS $$
BEGIN
    DELETE FROM sessions WHERE expires_at < NOW();
END;
$$ language 'plpgsql';
```

```sql
-- Asset tracking (for bundled CSS/JS)
CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_type VARCHAR(20) NOT NULL,
    theme_scope VARCHAR(255) NOT NULL,
    content_hash VARCHAR(64) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    size_bytes INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT assets_bundle_type_check 
        CHECK (bundle_type IN ('css', 'js', 'image'))
);

CREATE UNIQUE INDEX idx_assets_hash ON assets(content_hash);
CREATE INDEX idx_assets_theme_scope ON assets(theme_scope);
```

## UCG Database Schema

```sql
-- UCG database stores structure (compressed backup)
-- This is recovery-only, not for live queries

-- Entity storage (compressed)
CREATE TABLE ucg_entities (
    id UUID PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
    semantic_name VARCHAR(255),
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    
    -- Compression for backup efficiency
    compressed_data BYTEA
);

CREATE INDEX idx_ucg_entities_type ON ucg_entities(entity_type);
CREATE INDEX idx_ucg_entities_semantic_name ON ucg_entities(semantic_name);

-- Association storage (compressed)
CREATE TABLE ucg_associations (
    id UUID PRIMARY KEY,
    parent_id UUID NOT NULL,
    child_id UUID NOT NULL,
    association_type VARCHAR(50) NOT NULL,
    path VARCHAR(500) NOT NULL,
    weight INTEGER NOT NULL DEFAULT 0,
    data JSONB DEFAULT '{}',
    
    -- Compression
    compressed_data BYTEA
);

CREATE INDEX idx_ucg_associations_parent ON ucg_associations(parent_id);
CREATE INDEX idx_ucg_associations_child ON ucg_associations(child_id);
CREATE INDEX idx_ucg_associations_path ON ucg_associations(path);

-- Backup metadata
CREATE TABLE ucg_backups (
    id SERIAL PRIMARY KEY,
    backup_type VARCHAR(50) NOT NULL,
    entity_count INTEGER NOT NULL,
    association_count INTEGER NOT NULL,
    size_bytes BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'running',
    
    CONSTRAINT ucg_backups_status_check 
        CHECK (status IN ('running', 'completed', 'failed'))
);
```

## Migration Support Tables

```sql
-- Track schema versions
CREATE TABLE schema_migrations (
    version INTEGER PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audit log for important changes
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50),
    entity_id UUID,
    old_data JSONB,
    new_data JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_user_id ON audit_log(user_id);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at);
```

## Schema Management Functions

```rust
use sqlx::PgPool;

/// Schema initialization and management
pub struct SchemaManager {
    main_pool: PgPool,
    ucg_pool: PgPool,
}

impl SchemaManager {
    pub fn new(main_pool: PgPool, ucg_pool: PgPool) -> Self {
        Self { main_pool, ucg_pool }
    }
    
    /// Initialize main database schema
    pub async fn init_main_schema(&self) -> Result<()> {
        sqlx::query(include_str!("sql/main_schema.sql"))
            .execute(&self.main_pool)
            .await?;
        
        tracing::info!("Main database schema initialized");
        Ok(())
    }
    
    /// Initialize UCG database schema
    pub async fn init_ucg_schema(&self) -> Result<()> {
        sqlx::query(include_str!("sql/ucg_schema.sql"))
            .execute(&self.ucg_pool)
            .await?;
        
        tracing::info!("UCG database schema initialized");
        Ok(())
    }
    
    /// Add locale-specific columns dynamically
    pub async fn add_locale_columns(&self, locale: &str) -> Result<()> {
        let queries = vec![
            format!("ALTER TABLE snippet_content ADD COLUMN IF NOT EXISTS title_{} TEXT", locale),
            format!("ALTER TABLE snippet_content ADD COLUMN IF NOT EXISTS content_{} TEXT", locale),
            format!("ALTER TABLE snippet_content ADD COLUMN IF NOT EXISTS fields_{} JSONB DEFAULT '{{}}'", locale),
        ];
        
        for query in queries {
            sqlx::query(&query)
                .execute(&self.main_pool)
                .await?;
        }
        
        tracing::info!("Added columns for locale: {}", locale);
        Ok(())
    }
    
    /// Verify schema integrity
    pub async fn verify_schema(&self) -> Result<SchemaStatus> {
        let main_tables = self.check_main_tables().await?;
        let ucg_tables = self.check_ucg_tables().await?;
        
        Ok(SchemaStatus {
            main_healthy: main_tables.len() == 5, // Expected number of tables
            ucg_healthy: ucg_tables.len() == 3,
            main_tables,
            ucg_tables,
        })
    }
    
    async fn check_main_tables(&self) -> Result<Vec<String>> {
        let tables: Vec<(String,)> = sqlx::query_as(
            "SELECT tablename FROM pg_tables WHERE schemaname = 'public'"
        )
        .fetch_all(&self.main_pool)
        .await?;
        
        Ok(tables.into_iter().map(|(name,)| name).collect())
    }
    
    async fn check_ucg_tables(&self) -> Result<Vec<String>> {
        let tables: Vec<(String,)> = sqlx::query_as(
            "SELECT tablename FROM pg_tables WHERE schemaname = 'public'"
        )
        .fetch_all(&self.ucg_pool)
        .await?;
        
        Ok(tables.into_iter().map(|(name,)| name).collect())
    }
}

#[derive(Debug, Clone)]
pub struct SchemaStatus {
    pub main_healthy: bool,
    pub ucg_healthy: bool,
    pub main_tables: Vec<String>,
    pub ucg_tables: Vec<String>,
}
```

## Performance Considerations

```sql
-- Maintenance commands for optimal performance

-- Update statistics
ANALYZE snippet_content;
ANALYZE users;
ANALYZE sessions;

-- Vacuum for space reclamation
VACUUM ANALYZE snippet_content;

-- Monitor table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Monitor slow queries
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
    query,
    calls,
    total_time,
    mean_time,
    stddev_time
FROM pg_stat_statements
WHERE mean_time > 10
ORDER BY mean_time DESC
LIMIT 20;
```

## Summary

Dieses Schema bietet:
- **Flache Struktur** ohne komplexe JOINs
- **Locale-spezifische Spalten** dynamisch erweiterbar
- **Performance-Indizes** auf allen wichtigen Feldern
- **UCG Backup** komprimiert für Recovery
- **Audit Trail** für Compliance

Das Design folgt dem KISS-Prinzip: PostgreSQL speichert dumm, UCG macht intelligent.
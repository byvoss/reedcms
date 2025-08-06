# Snippet Storage System

## Overview

PostgreSQL Storage Layer für Snippets. Optimiert für schnelle Queries mit Full-text Search und effizienter Indexierung.

## Database Schema

```sql
-- Snippets main table
CREATE TABLE snippets (
    id UUID PRIMARY KEY,
    snippet_type VARCHAR(255) NOT NULL,
    data JSONB NOT NULL,
    search_vector tsvector,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    created_by VARCHAR(255) NOT NULL
);

-- Indexes for performance
CREATE INDEX idx_snippets_type ON snippets(snippet_type);
CREATE INDEX idx_snippets_data_gin ON snippets USING GIN (data);
CREATE INDEX idx_snippets_search ON snippets USING GIN (search_vector);
CREATE INDEX idx_snippets_created ON snippets(created_at DESC);
CREATE INDEX idx_snippets_updated ON snippets(updated_at DESC);

-- Snippet fields (denormalized for fast queries)
CREATE TABLE snippet_fields (
    snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
    field_name VARCHAR(255) NOT NULL,
    field_value TEXT,
    field_type VARCHAR(50) NOT NULL,
    PRIMARY KEY (snippet_id, field_name)
);

CREATE INDEX idx_snippet_fields_name_value ON snippet_fields(field_name, field_value);
CREATE INDEX idx_snippet_fields_type ON snippet_fields(field_type);

-- Snippet versions for history
CREATE TABLE snippet_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by VARCHAR(255) NOT NULL,
    change_summary TEXT
);

CREATE INDEX idx_snippet_versions_snippet ON snippet_versions(snippet_id, version_number DESC);
CREATE INDEX idx_snippet_versions_created ON snippet_versions(created_at DESC);
```

## Storage Repository

```rust
use sqlx::{PgPool, Transaction, Postgres};
use uuid::Uuid;
use serde_json::Value;

/// PostgreSQL storage for snippets
pub struct SnippetStorage {
    pool: PgPool,
    field_extractor: Arc<FieldExtractor>,
}

impl SnippetStorage {
    pub fn new(pool: PgPool) -> Self {
        Self {
            pool,
            field_extractor: Arc::new(FieldExtractor::new()),
        }
    }
    
    /// Store snippet with denormalized fields
    pub async fn store(
        &self,
        entity: &Entity,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .ok_or_else(|| ReedError::InvalidData("Missing snippet_type".into()))?;
        
        // Insert main record
        let query = r#"
            INSERT INTO snippets (
                id, snippet_type, data, search_vector,
                created_at, updated_at, created_by
            ) VALUES ($1, $2, $3, to_tsvector('english', $4), $5, $6, $7)
        "#;
        
        let search_text = self.field_extractor.extract_search_text(&entity.data);
        
        sqlx::query(query)
            .bind(&entity.id)
            .bind(snippet_type)
            .bind(&entity.data)
            .bind(&search_text)
            .bind(&entity.created_at)
            .bind(&entity.updated_at)
            .bind(&entity.created_by)
            .execute(&mut **tx)
            .await?;
        
        // Store denormalized fields
        self.store_fields(&entity.id, &entity.data, tx).await?;
        
        // Create initial version
        self.create_version(entity, 1, "Initial creation", &entity.created_by, tx).await?;
        
        Ok(())
    }
    
    /// Update snippet
    pub async fn update(
        &self,
        entity: &Entity,
        change_summary: &str,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        // Get current version number
        let version_number = self.get_next_version_number(&entity.id, tx).await?;
        
        // Update main record
        let query = r#"
            UPDATE snippets
            SET data = $1,
                search_vector = to_tsvector('english', $2),
                updated_at = $3
            WHERE id = $4
        "#;
        
        let search_text = self.field_extractor.extract_search_text(&entity.data);
        
        sqlx::query(query)
            .bind(&entity.data)
            .bind(&search_text)
            .bind(&entity.updated_at)
            .bind(&entity.id)
            .execute(&mut **tx)
            .await?;
        
        // Update denormalized fields
        self.update_fields(&entity.id, &entity.data, tx).await?;
        
        // Create version
        self.create_version(entity, version_number, change_summary, &entity.created_by, tx).await?;
        
        Ok(())
    }
    
    /// Delete snippet
    pub async fn delete(
        &self,
        id: &Uuid,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        // Fields and versions are cascade deleted
        sqlx::query("DELETE FROM snippets WHERE id = $1")
            .bind(id)
            .execute(&mut **tx)
            .await?;
        
        Ok(())
    }
}
```

## Field Storage

```rust
impl SnippetStorage {
    /// Store denormalized fields for fast queries
    async fn store_fields(
        &self,
        snippet_id: &Uuid,
        data: &Value,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        if let Some(obj) = data.as_object() {
            for (field_name, field_value) in obj {
                // Skip system fields
                if field_name.starts_with('_') {
                    continue;
                }
                
                let (value_str, field_type) = self.extract_field_value(field_value);
                
                if let Some(value) = value_str {
                    let query = r#"
                        INSERT INTO snippet_fields (
                            snippet_id, field_name, field_value, field_type
                        ) VALUES ($1, $2, $3, $4)
                    "#;
                    
                    sqlx::query(query)
                        .bind(snippet_id)
                        .bind(field_name)
                        .bind(&value)
                        .bind(&field_type)
                        .execute(&mut **tx)
                        .await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Update denormalized fields
    async fn update_fields(
        &self,
        snippet_id: &Uuid,
        data: &Value,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        // Delete existing fields
        sqlx::query("DELETE FROM snippet_fields WHERE snippet_id = $1")
            .bind(snippet_id)
            .execute(&mut **tx)
            .await?;
        
        // Insert new fields
        self.store_fields(snippet_id, data, tx).await
    }
    
    /// Extract field value for storage
    fn extract_field_value(&self, value: &Value) -> (Option<String>, String) {
        match value {
            Value::String(s) => (Some(s.clone()), "string".to_string()),
            Value::Number(n) => (Some(n.to_string()), "number".to_string()),
            Value::Bool(b) => (Some(b.to_string()), "boolean".to_string()),
            Value::Null => (None, "null".to_string()),
            Value::Array(_) => (Some(value.to_string()), "array".to_string()),
            Value::Object(_) => (Some(value.to_string()), "object".to_string()),
        }
    }
}
```

## Query Builder

```rust
/// Build complex queries for snippets
pub struct SnippetQueryBuilder {
    base_query: String,
    conditions: Vec<String>,
    parameters: Vec<Box<dyn ToSql + Send + Sync>>,
    order_by: Option<String>,
    limit: Option<usize>,
    offset: Option<usize>,
}

impl SnippetQueryBuilder {
    pub fn new() -> Self {
        Self {
            base_query: "SELECT DISTINCT s.* FROM snippets s".to_string(),
            conditions: vec!["s.id IS NOT NULL".to_string()], // Always true condition
            parameters: Vec::new(),
            order_by: None,
            limit: None,
            offset: None,
        }
    }
    
    /// Filter by snippet type
    pub fn with_type(mut self, snippet_type: &str) -> Self {
        let param_index = self.parameters.len() + 1;
        self.conditions.push(format!("s.snippet_type = ${}", param_index));
        self.parameters.push(Box::new(snippet_type.to_string()));
        self
    }
    
    /// Filter by field value
    pub fn with_field(mut self, field_name: &str, field_value: &str) -> Self {
        // Join with fields table
        if !self.base_query.contains("snippet_fields") {
            self.base_query.push_str(" LEFT JOIN snippet_fields f ON s.id = f.snippet_id");
        }
        
        let param_index = self.parameters.len() + 1;
        self.conditions.push(format!(
            "(f.field_name = ${} AND f.field_value = ${})",
            param_index,
            param_index + 1
        ));
        self.parameters.push(Box::new(field_name.to_string()));
        self.parameters.push(Box::new(field_value.to_string()));
        self
    }
    
    /// Filter by JSON path
    pub fn with_json_path(mut self, path: &str, value: &Value) -> Self {
        let param_index = self.parameters.len() + 1;
        self.conditions.push(format!("s.data @> ${}::jsonb", param_index));
        
        let json_filter = serde_json::json!({
            path: value
        });
        self.parameters.push(Box::new(json_filter));
        self
    }
    
    /// Full text search
    pub fn with_search(mut self, query: &str) -> Self {
        let param_index = self.parameters.len() + 1;
        self.conditions.push(format!(
            "s.search_vector @@ plainto_tsquery('english', ${})",
            param_index
        ));
        self.parameters.push(Box::new(query.to_string()));
        
        // Order by relevance if no other order specified
        if self.order_by.is_none() {
            self.order_by = Some(format!(
                "ts_rank(s.search_vector, plainto_tsquery('english', ${})) DESC",
                param_index
            ));
        }
        
        self
    }
    
    /// Set ordering
    pub fn order_by(mut self, field: &str, desc: bool) -> Self {
        let direction = if desc { "DESC" } else { "ASC" };
        self.order_by = Some(format!("s.{} {}", field, direction));
        self
    }
    
    /// Set limit
    pub fn limit(mut self, limit: usize) -> Self {
        self.limit = Some(limit);
        self
    }
    
    /// Set offset
    pub fn offset(mut self, offset: usize) -> Self {
        self.offset = Some(offset);
        self
    }
    
    /// Build final query
    pub fn build(self) -> (String, Vec<Box<dyn ToSql + Send + Sync>>) {
        let mut query = self.base_query;
        
        // Add WHERE clause
        if !self.conditions.is_empty() {
            query.push_str(" WHERE ");
            query.push_str(&self.conditions.join(" AND "));
        }
        
        // Add ORDER BY
        if let Some(order) = self.order_by {
            query.push_str(" ORDER BY ");
            query.push_str(&order);
        }
        
        // Add LIMIT
        if let Some(limit) = self.limit {
            query.push_str(&format!(" LIMIT {}", limit));
        }
        
        // Add OFFSET
        if let Some(offset) = self.offset {
            query.push_str(&format!(" OFFSET {}", offset));
        }
        
        (query, self.parameters)
    }
}
```

## Version Management

```rust
impl SnippetStorage {
    /// Create version record
    async fn create_version(
        &self,
        entity: &Entity,
        version_number: i32,
        change_summary: &str,
        created_by: &str,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<()> {
        let query = r#"
            INSERT INTO snippet_versions (
                snippet_id, version_number, data,
                created_at, created_by, change_summary
            ) VALUES ($1, $2, $3, $4, $5, $6)
        "#;
        
        sqlx::query(query)
            .bind(&entity.id)
            .bind(version_number)
            .bind(&entity.data)
            .bind(&entity.updated_at)
            .bind(created_by)
            .bind(change_summary)
            .execute(&mut **tx)
            .await?;
        
        Ok(())
    }
    
    /// Get next version number
    async fn get_next_version_number(
        &self,
        snippet_id: &Uuid,
        tx: &mut Transaction<'_, Postgres>,
    ) -> Result<i32> {
        let query = r#"
            SELECT COALESCE(MAX(version_number), 0) + 1
            FROM snippet_versions
            WHERE snippet_id = $1
        "#;
        
        let row: (i32,) = sqlx::query_as(query)
            .bind(snippet_id)
            .fetch_one(&mut **tx)
            .await?;
        
        Ok(row.0)
    }
    
    /// Get version history
    pub async fn get_versions(
        &self,
        snippet_id: Uuid,
        limit: usize,
    ) -> Result<Vec<SnippetVersion>> {
        let query = r#"
            SELECT id, snippet_id, version_number, data,
                   created_at, created_by, change_summary
            FROM snippet_versions
            WHERE snippet_id = $1
            ORDER BY version_number DESC
            LIMIT $2
        "#;
        
        let versions = sqlx::query_as::<_, SnippetVersionRow>(query)
            .bind(&snippet_id)
            .bind(limit as i64)
            .fetch_all(&self.pool)
            .await?
            .into_iter()
            .map(|row| row.into())
            .collect();
        
        Ok(versions)
    }
    
    /// Restore from version
    pub async fn restore_version(
        &self,
        snippet_id: Uuid,
        version_number: i32,
        user: &UserContext,
    ) -> Result<Entity> {
        let query = r#"
            SELECT data FROM snippet_versions
            WHERE snippet_id = $1 AND version_number = $2
        "#;
        
        let row: (Value,) = sqlx::query_as(query)
            .bind(&snippet_id)
            .bind(version_number)
            .fetch_one(&self.pool)
            .await?;
        
        // Create entity from version data
        let mut entity = Entity {
            id: snippet_id,
            entity_type: EntityType::Snippet,
            semantic_name: None,
            created_at: Utc::now(), // Will be updated
            updated_at: Utc::now(),
            created_by: user.id.to_string(),
            data: row.0,
        };
        
        // Update with restore
        let mut tx = self.pool.begin().await?;
        
        self.update(
            &entity,
            &format!("Restored from version {}", version_number),
            &mut tx,
        ).await?;
        
        tx.commit().await?;
        
        Ok(entity)
    }
}
```

## Search Optimization

```rust
/// Extract and optimize search text
pub struct FieldExtractor {
    stop_words: HashSet<String>,
}

impl FieldExtractor {
    pub fn new() -> Self {
        let stop_words = vec![
            "a", "an", "and", "are", "as", "at", "be", "by", "for",
            "from", "has", "he", "in", "is", "it", "its", "of", "on",
            "that", "the", "to", "was", "will", "with"
        ].into_iter().map(String::from).collect();
        
        Self { stop_words }
    }
    
    /// Extract searchable text from JSON
    pub fn extract_search_text(&self, data: &Value) -> String {
        let mut text_parts = Vec::new();
        
        if let Some(obj) = data.as_object() {
            // Priority fields
            for field in &["title", "name", "headline", "summary"] {
                if let Some(Value::String(s)) = obj.get(*field) {
                    text_parts.push(s.clone());
                }
            }
            
            // Content fields
            for field in &["content", "body", "description", "text"] {
                if let Some(Value::String(s)) = obj.get(*field) {
                    // Strip HTML if present
                    let clean = self.strip_html(s);
                    text_parts.push(clean);
                }
            }
            
            // Additional text fields
            for (key, value) in obj {
                if !key.starts_with('_') && key.ends_with("_text") {
                    if let Value::String(s) = value {
                        text_parts.push(s.clone());
                    }
                }
            }
        }
        
        // Process and clean text
        let combined = text_parts.join(" ");
        self.clean_search_text(&combined)
    }
    
    /// Clean and optimize search text
    fn clean_search_text(&self, text: &str) -> String {
        text.split_whitespace()
            .filter(|word| {
                word.len() > 2 && !self.stop_words.contains(&word.to_lowercase())
            })
            .take(1000) // Limit to 1000 words
            .collect::<Vec<_>>()
            .join(" ")
    }
    
    /// Strip HTML tags
    fn strip_html(&self, html: &str) -> String {
        // Simple HTML stripping (use proper HTML parser in production)
        let re = regex::Regex::new(r"<[^>]+>").unwrap();
        re.replace_all(html, " ").to_string()
    }
}
```

## Storage Statistics

```rust
/// Collect storage statistics
pub struct StorageStats {
    pool: PgPool,
}

impl StorageStats {
    pub async fn get_stats(&self) -> Result<SnippetStorageStats> {
        let total_query = "SELECT COUNT(*) FROM snippets";
        let type_query = r#"
            SELECT snippet_type, COUNT(*) as count
            FROM snippets
            GROUP BY snippet_type
            ORDER BY count DESC
        "#;
        
        let (total,): (i64,) = sqlx::query_as(total_query)
            .fetch_one(&self.pool)
            .await?;
        
        let by_type: Vec<(String, i64)> = sqlx::query_as(type_query)
            .fetch_all(&self.pool)
            .await?;
        
        let storage_size_query = r#"
            SELECT 
                pg_size_pretty(pg_total_relation_size('snippets')),
                pg_size_pretty(pg_total_relation_size('snippet_fields')),
                pg_size_pretty(pg_total_relation_size('snippet_versions'))
        "#;
        
        let (main_size, fields_size, versions_size): (String, String, String) = 
            sqlx::query_as(storage_size_query)
                .fetch_one(&self.pool)
                .await?;
        
        Ok(SnippetStorageStats {
            total_snippets: total as usize,
            snippets_by_type: by_type.into_iter().collect(),
            storage_size: StorageSize {
                main_table: main_size,
                fields_table: fields_size,
                versions_table: versions_size,
            },
        })
    }
}

#[derive(Debug, Serialize)]
pub struct SnippetStorageStats {
    pub total_snippets: usize,
    pub snippets_by_type: HashMap<String, i64>,
    pub storage_size: StorageSize,
}

#[derive(Debug, Serialize)]
pub struct StorageSize {
    pub main_table: String,
    pub fields_table: String,
    pub versions_table: String,
}
```

## Summary

Dieses Storage System bietet:
- **Denormalized Fields** - Schnelle Queries ohne JSON parsing
- **Full-text Search** - PostgreSQL tsvector für Relevanz-Suche
- **Version History** - Komplette Änderungshistorie
- **Query Builder** - Flexible, type-safe Query Construction
- **Performance Indexes** - Optimiert für Read-heavy Workloads

PostgreSQL als Storage Engine nutzt die volle Power der Datenbank.
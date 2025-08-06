# Snippet Core Operations

## Overview

Snippet CRUD Operations für ReedCMS. Management von Content-Snippets mit UCG Integration und Registry Validation.

## Snippet Manager Core

```rust
use uuid::Uuid;
use std::collections::HashMap;
use serde_json::Value;

/// Core snippet management
pub struct SnippetManager {
    ucg_entity: Arc<UcgEntity>,
    ucg_association: Arc<UcgAssociation>,
    registry: Arc<RegistryManager>,
    validator: Arc<SnippetValidator>,
    cache: Arc<SnippetCache>,
    db: Arc<DatabaseConnection>,
}

impl SnippetManager {
    pub fn new(
        ucg_entity: Arc<UcgEntity>,
        ucg_association: Arc<UcgAssociation>,
        registry: Arc<RegistryManager>,
        validator: Arc<SnippetValidator>,
        cache: Arc<SnippetCache>,
        db: Arc<DatabaseConnection>,
    ) -> Self {
        Self {
            ucg_entity,
            ucg_association,
            registry,
            validator,
            cache,
            db,
        }
    }
    
    /// Create new snippet
    pub async fn create_snippet(
        &self,
        snippet_type: &str,
        data: HashMap<String, Value>,
        parent_id: Option<Uuid>,
        user: &UserContext,
    ) -> Result<Entity> {
        // Validate snippet type exists
        let definition = self.registry.get_definition(snippet_type).await?;
        
        // Prepare data with defaults
        let mut snippet_data = self.prepare_snippet_data(&definition, data)?;
        
        // Add snippet metadata
        snippet_data.insert("_type".to_string(), Value::String("snippet".to_string()));
        snippet_data.insert("snippet_type".to_string(), Value::String(snippet_type.to_string()));
        
        // Validate data
        let validation_result = self.validator
            .validate_snippet(snippet_type, &snippet_data)
            .await?;
        
        if !validation_result.is_valid {
            return Err(ReedError::Validation(ValidationError::FieldErrors(
                validation_result.errors
            )));
        }
        
        // Begin transaction
        let mut tx = self.db.begin().await?;
        
        // Create entity
        let entity = Entity {
            id: Uuid::new_v4(),
            entity_type: EntityType::Snippet,
            semantic_name: self.generate_semantic_name(&snippet_data),
            created_at: Utc::now(),
            updated_at: Utc::now(),
            created_by: user.id.to_string(),
            data: serde_json::to_value(snippet_data)?,
        };
        
        // Store in UCG
        self.ucg_entity.create(&entity, &mut tx).await?;
        
        // Create parent association if needed
        if let Some(pid) = parent_id {
            let association = Association {
                id: Uuid::new_v4(),
                parent_id: pid,
                child_id: entity.id,
                association_type: AssociationType::Contains,
                weight: self.get_next_weight(pid).await?,
                path: format!("{}.{}", pid, entity.id),
            };
            
            self.ucg_association.create(&association, &mut tx).await?;
        }
        
        // Store in main table
        self.store_snippet_record(&entity, &mut tx).await?;
        
        // Commit transaction
        tx.commit().await?;
        
        // Invalidate cache
        self.cache.invalidate_snippet(&entity.id).await?;
        if let Some(pid) = parent_id {
            self.cache.invalidate_children(pid).await?;
        }
        
        // Trigger hooks
        self.trigger_after_create(&entity).await?;
        
        Ok(entity)
    }
    
    /// Update existing snippet
    pub async fn update_snippet(
        &self,
        id: Uuid,
        updates: HashMap<String, Value>,
        user: &UserContext,
    ) -> Result<Entity> {
        // Get existing entity
        let mut entity = self.ucg_entity.get_by_id(id).await?
            .ok_or_else(|| ReedError::NotFound(format!("Snippet {} not found", id)))?;
        
        // Check permissions
        self.check_update_permission(&entity, user)?;
        
        // Get snippet type
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .ok_or_else(|| ReedError::InvalidData("Missing snippet_type".into()))?;
        
        // Get definition
        let definition = self.registry.get_definition(snippet_type).await?;
        
        // Merge updates
        let mut data = entity.data.as_object()
            .cloned()
            .unwrap_or_default();
        
        for (key, value) in updates {
            if key.starts_with('_') {
                continue; // Skip system fields
            }
            data.insert(key, value);
        }
        
        // Validate updated data
        let validation_result = self.validator
            .validate_snippet(snippet_type, &data)
            .await?;
        
        if !validation_result.is_valid {
            return Err(ReedError::Validation(ValidationError::FieldErrors(
                validation_result.errors
            )));
        }
        
        // Update entity
        entity.data = serde_json::to_value(data)?;
        entity.updated_at = Utc::now();
        
        // Begin transaction
        let mut tx = self.db.begin().await?;
        
        // Update in UCG
        self.ucg_entity.update(&entity, &mut tx).await?;
        
        // Update in main table
        self.update_snippet_record(&entity, &mut tx).await?;
        
        // Commit
        tx.commit().await?;
        
        // Invalidate cache
        self.cache.invalidate_snippet(&entity.id).await?;
        
        // Trigger hooks
        self.trigger_after_update(&entity).await?;
        
        Ok(entity)
    }
    
    /// Delete snippet
    pub async fn delete_snippet(
        &self,
        id: Uuid,
        user: &UserContext,
    ) -> Result<()> {
        // Get entity
        let entity = self.ucg_entity.get_by_id(id).await?
            .ok_or_else(|| ReedError::NotFound(format!("Snippet {} not found", id)))?;
        
        // Check permissions
        self.check_delete_permission(&entity, user)?;
        
        // Check if has children
        let children = self.ucg_association.get_children(id).await?;
        if !children.is_empty() {
            return Err(ReedError::HasChildren(id));
        }
        
        // Begin transaction
        let mut tx = self.db.begin().await?;
        
        // Delete associations
        self.ucg_association.delete_by_child(id, &mut tx).await?;
        
        // Delete from main table
        self.delete_snippet_record(id, &mut tx).await?;
        
        // Delete from UCG
        self.ucg_entity.delete(id, &mut tx).await?;
        
        // Commit
        tx.commit().await?;
        
        // Invalidate cache
        self.cache.invalidate_snippet(&id).await?;
        
        // Get parent for cache invalidation
        if let Some(parent_assoc) = self.ucg_association.get_parent(id).await? {
            self.cache.invalidate_children(parent_assoc.parent_id).await?;
        }
        
        // Trigger hooks
        self.trigger_after_delete(&entity).await?;
        
        Ok(())
    }
}
```

## Snippet Data Preparation

```rust
impl SnippetManager {
    /// Prepare snippet data with defaults and transformations
    fn prepare_snippet_data(
        &self,
        definition: &SnippetDefinition,
        mut data: HashMap<String, Value>,
    ) -> Result<HashMap<String, Value>> {
        let mut prepared = HashMap::new();
        
        // Process each field
        for field_def in &definition.fields {
            let value = if let Some(v) = data.remove(&field_def.name) {
                // Transform value based on field type
                self.transform_field_value(v, &field_def.field_type)?
            } else if let Some(default) = &field_def.default_value {
                default.clone()
            } else if field_def.required {
                return Err(ReedError::Validation(ValidationError::RequiredField(
                    field_def.name.clone()
                )));
            } else {
                Value::Null
            };
            
            prepared.insert(field_def.name.clone(), value);
        }
        
        // Check for unknown fields
        if !data.is_empty() {
            let unknown: Vec<String> = data.keys().cloned().collect();
            tracing::warn!("Unknown fields in snippet data: {:?}", unknown);
        }
        
        Ok(prepared)
    }
    
    /// Transform field value to correct type
    fn transform_field_value(&self, value: Value, field_type: &FieldType) -> Result<Value> {
        match field_type {
            FieldType::Text | FieldType::Textarea => {
                match value {
                    Value::String(s) => Ok(Value::String(s)),
                    _ => Ok(Value::String(value.to_string())),
                }
            }
            FieldType::Number => {
                match value {
                    Value::Number(n) => Ok(Value::Number(n)),
                    Value::String(s) => {
                        let num = s.parse::<f64>()
                            .map_err(|_| ReedError::InvalidData(
                                format!("Invalid number: {}", s)
                            ))?;
                        Ok(serde_json::Number::from_f64(num)
                            .map(Value::Number)
                            .unwrap_or(Value::Null))
                    }
                    _ => Err(ReedError::InvalidData("Expected number".into())),
                }
            }
            FieldType::Boolean => {
                match value {
                    Value::Bool(b) => Ok(Value::Bool(b)),
                    Value::String(s) => Ok(Value::Bool(s == "true" || s == "1")),
                    Value::Number(n) => Ok(Value::Bool(n.as_i64().unwrap_or(0) != 0)),
                    _ => Ok(Value::Bool(false)),
                }
            }
            FieldType::Date => {
                match value {
                    Value::String(s) => {
                        // Validate date format
                        chrono::DateTime::parse_from_rfc3339(&s)
                            .map_err(|_| ReedError::InvalidData(
                                format!("Invalid date format: {}", s)
                            ))?;
                        Ok(Value::String(s))
                    }
                    _ => Err(ReedError::InvalidData("Expected date string".into())),
                }
            }
            FieldType::Reference(entity_type) => {
                match value {
                    Value::String(s) => {
                        // Validate UUID
                        let uuid = Uuid::parse_str(&s)
                            .map_err(|_| ReedError::InvalidData(
                                format!("Invalid UUID: {}", s)
                            ))?;
                        
                        // Verify entity exists
                        self.verify_reference(uuid, entity_type).await?;
                        
                        Ok(Value::String(s))
                    }
                    _ => Err(ReedError::InvalidData("Expected UUID string".into())),
                }
            }
            _ => Ok(value),
        }
    }
    
    /// Generate semantic name from data
    fn generate_semantic_name(&self, data: &HashMap<String, Value>) -> Option<SemanticName> {
        // Try common fields
        for field in &["slug", "name", "title"] {
            if let Some(Value::String(s)) = data.get(*field) {
                if !s.is_empty() {
                    return SemanticName::new(s).ok();
                }
            }
        }
        
        None
    }
}
```

## Snippet Queries

```rust
impl SnippetManager {
    /// Get snippet by ID
    pub async fn get_by_id(&self, id: Uuid) -> Result<Option<Entity>> {
        // Check cache first
        if let Some(entity) = self.cache.get_snippet(id).await? {
            return Ok(Some(entity));
        }
        
        // Load from UCG
        if let Some(entity) = self.ucg_entity.get_by_id(id).await? {
            // Verify it's a snippet
            if entity.entity_type == EntityType::Snippet {
                // Cache it
                self.cache.store_snippet(&entity).await?;
                return Ok(Some(entity));
            }
        }
        
        Ok(None)
    }
    
    /// Get snippet by semantic name
    pub async fn get_by_name(&self, name: &str) -> Result<Option<Entity>> {
        self.ucg_entity.get_by_semantic_name(name).await
    }
    
    /// Get snippets by type
    pub async fn get_by_type(
        &self,
        snippet_type: &str,
        limit: usize,
        offset: usize,
    ) -> Result<Vec<Entity>> {
        let query = r#"
            SELECT e.*
            FROM entities e
            WHERE e.entity_type = 'snippet'
            AND e.data->>'snippet_type' = $1
            ORDER BY e.created_at DESC
            LIMIT $2 OFFSET $3
        "#;
        
        let entities = sqlx::query_as::<_, EntityRow>(query)
            .bind(snippet_type)
            .bind(limit as i64)
            .bind(offset as i64)
            .fetch_all(&*self.db.pool)
            .await?
            .into_iter()
            .map(|row| row.into())
            .collect();
        
        Ok(entities)
    }
    
    /// Search snippets
    pub async fn search(
        &self,
        query: &str,
        snippet_type: Option<&str>,
        limit: usize,
    ) -> Result<Vec<Entity>> {
        let mut sql = String::from(r#"
            SELECT e.*
            FROM entities e
            WHERE e.entity_type = 'snippet'
            AND e.search_vector @@ plainto_tsquery('english', $1)
        "#);
        
        let mut binds = vec![query.to_string()];
        let mut bind_index = 2;
        
        if let Some(st) = snippet_type {
            sql.push_str(&format!(" AND e.data->>'snippet_type' = ${}", bind_index));
            binds.push(st.to_string());
            bind_index += 1;
        }
        
        sql.push_str(&format!(" ORDER BY ts_rank(e.search_vector, plainto_tsquery('english', $1)) DESC LIMIT ${}", bind_index));
        binds.push(limit.to_string());
        
        // Execute dynamic query
        let mut query = sqlx::query_as::<_, EntityRow>(&sql);
        for bind in binds {
            query = query.bind(bind);
        }
        
        let entities = query
            .fetch_all(&*self.db.pool)
            .await?
            .into_iter()
            .map(|row| row.into())
            .collect();
        
        Ok(entities)
    }
}
```

## Snippet Storage

```rust
impl SnippetManager {
    /// Store snippet in main table
    async fn store_snippet_record(
        &self,
        entity: &Entity,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
    ) -> Result<()> {
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown");
        
        let query = r#"
            INSERT INTO snippets (
                id, snippet_type, data, search_vector,
                created_at, updated_at, created_by
            ) VALUES ($1, $2, $3, to_tsvector('english', $4), $5, $6, $7)
        "#;
        
        let search_text = self.extract_search_text(&entity.data);
        
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
        
        Ok(())
    }
    
    /// Update snippet in main table
    async fn update_snippet_record(
        &self,
        entity: &Entity,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
    ) -> Result<()> {
        let query = r#"
            UPDATE snippets
            SET data = $1,
                search_vector = to_tsvector('english', $2),
                updated_at = $3
            WHERE id = $4
        "#;
        
        let search_text = self.extract_search_text(&entity.data);
        
        sqlx::query(query)
            .bind(&entity.data)
            .bind(&search_text)
            .bind(&entity.updated_at)
            .bind(&entity.id)
            .execute(&mut **tx)
            .await?;
        
        Ok(())
    }
    
    /// Delete snippet from main table
    async fn delete_snippet_record(
        &self,
        id: Uuid,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
    ) -> Result<()> {
        sqlx::query("DELETE FROM snippets WHERE id = $1")
            .bind(&id)
            .execute(&mut **tx)
            .await?;
        
        Ok(())
    }
    
    /// Extract searchable text from data
    fn extract_search_text(&self, data: &Value) -> String {
        let mut text = String::new();
        
        if let Some(obj) = data.as_object() {
            for (key, value) in obj {
                // Skip system fields
                if key.starts_with('_') {
                    continue;
                }
                
                // Extract text from common fields
                if matches!(key.as_str(), "title" | "content" | "description" | "name") {
                    if let Some(s) = value.as_str() {
                        text.push_str(s);
                        text.push(' ');
                    }
                }
            }
        }
        
        text.trim().to_string()
    }
}
```

## Snippet Hooks

```rust
impl SnippetManager {
    /// Trigger after create hooks
    async fn trigger_after_create(&self, entity: &Entity) -> Result<()> {
        // Intelligence hook
        if let Some(provider) = &self.intelligence_provider {
            provider.on_snippet_created(entity).await?;
        }
        
        // Plugin hooks
        self.plugin_manager.trigger_event(
            "snippet.after_create",
            serde_json::to_value(entity)?,
        ).await?;
        
        Ok(())
    }
    
    /// Trigger after update hooks
    async fn trigger_after_update(&self, entity: &Entity) -> Result<()> {
        // Intelligence hook
        if let Some(provider) = &self.intelligence_provider {
            provider.on_snippet_updated(entity).await?;
        }
        
        // Plugin hooks
        self.plugin_manager.trigger_event(
            "snippet.after_update",
            serde_json::to_value(entity)?,
        ).await?;
        
        Ok(())
    }
    
    /// Trigger after delete hooks
    async fn trigger_after_delete(&self, entity: &Entity) -> Result<()> {
        // Intelligence hook
        if let Some(provider) = &self.intelligence_provider {
            provider.on_snippet_deleted(entity).await?;
        }
        
        // Plugin hooks
        self.plugin_manager.trigger_event(
            "snippet.after_delete",
            serde_json::to_value(entity)?,
        ).await?;
        
        Ok(())
    }
}
```

## Snippet Permissions

```rust
impl SnippetManager {
    /// Check update permission
    fn check_update_permission(&self, entity: &Entity, user: &UserContext) -> Result<()> {
        // Admin can always update
        if user.is_admin() {
            return Ok(());
        }
        
        // Check ownership
        if entity.created_by == user.id.to_string() {
            return Ok(());
        }
        
        // Check specific permissions
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown");
        
        let permission = format!("snippet.{}.update", snippet_type);
        
        if user.has_permission(&permission) {
            return Ok(());
        }
        
        Err(ReedError::PermissionDenied(permission))
    }
    
    /// Check delete permission
    fn check_delete_permission(&self, entity: &Entity, user: &UserContext) -> Result<()> {
        // Admin can always delete
        if user.is_admin() {
            return Ok(());
        }
        
        // Check ownership
        if entity.created_by == user.id.to_string() {
            return Ok(());
        }
        
        // Check specific permissions
        let snippet_type = entity.data.get("snippet_type")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown");
        
        let permission = format!("snippet.{}.delete", snippet_type);
        
        if user.has_permission(&permission) {
            return Ok(());
        }
        
        Err(ReedError::PermissionDenied(permission))
    }
}
```

## Summary

Dieses Snippet Core System bietet:
- **Full CRUD Operations** - Create, Read, Update, Delete mit Validation
- **UCG Integration** - Nahtlose Entity/Association Verwaltung
- **Registry Validation** - Type-safe Field Handling
- **Search Support** - Full-text Search mit PostgreSQL
- **Hook System** - Intelligence und Plugin Integration

Snippets sind die Content-Bausteine von ReedCMS.
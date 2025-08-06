# UCG Entity Management

## Overview

Universal Content Graph (UCG) Entity Management - das Herzstück von ReedCMS. Alles ist eine Entity mit Associations.

## Entity Manager

```rust
use std::collections::HashMap;
use uuid::Uuid;

/// Core UCG entity manager
pub struct EntityManager {
    db: Arc<DatabaseManager>,
    redis: Arc<RedisManager>,
    memory: Arc<MemoryManager>,
}

impl EntityManager {
    pub fn new(
        db: Arc<DatabaseManager>,
        redis: Arc<RedisManager>,
        memory: Arc<MemoryManager>,
    ) -> Self {
        Self { db, redis, memory }
    }
    
    /// Create new entity
    pub async fn create_entity(
        &self,
        entity_type: EntityType,
        semantic_name: Option<&str>,
        data: serde_json::Value,
        created_by: &str,
    ) -> Result<Entity> {
        // Validate semantic name if provided
        let semantic = if let Some(name) = semantic_name {
            Some(SemanticName::new(name)?)
        } else {
            None
        };
        
        // Check for duplicate semantic name
        if let Some(ref sem_name) = semantic {
            if self.exists_by_semantic_name(&entity_type, sem_name).await? {
                return Err(ReedError::EntityExists {
                    semantic_name: sem_name.as_str().to_string(),
                });
            }
        }
        
        // Create entity
        let entity = Entity {
            id: Uuid::new_v4(),
            entity_type,
            semantic_name: semantic,
            created_at: Utc::now(),
            updated_at: Utc::now(),
            created_by: created_by.to_string(),
            data,
        };
        
        // Store in UCG (Redis)
        self.memory.store_ucg_entity(&entity).await?;
        
        // Store in PostgreSQL for persistence
        self.persist_entity(&entity).await?;
        
        tracing::info!(
            "Created entity: {} ({:?})", 
            entity.id,
            entity.semantic_name
        );
        
        Ok(entity)
    }
    
    /// Get entity by ID
    pub async fn get_entity(&self, id: &Uuid) -> Result<Entity> {
        // Try Redis first
        if let Some(entity) = self.get_from_cache(id).await? {
            return Ok(entity);
        }
        
        // Fallback to PostgreSQL
        let entity = self.get_from_database(id).await?;
        
        // Cache for next time
        self.memory.store_ucg_entity(&entity).await?;
        
        Ok(entity)
    }
    
    /// Get entity by semantic name
    pub async fn get_by_semantic_name(
        &self,
        entity_type: &EntityType,
        semantic_name: &SemanticName,
    ) -> Result<Entity> {
        // Look up ID from semantic mapping
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let semantic_key = keys.semantic(&entity_type.to_string(), semantic_name.as_str());
        
        let id_str = self.redis.get(&semantic_key).await?
            .ok_or_else(|| ReedError::EntityNotFound {
                id: Uuid::nil(), // Use nil UUID for semantic lookups
            })?;
        
        let id = Uuid::parse_str(&id_str)?;
        self.get_entity(&id).await
    }
}
```

## Entity CRUD Operations

```rust
impl EntityManager {
    /// Update entity
    pub async fn update_entity(
        &self,
        id: &Uuid,
        data: serde_json::Value,
        updated_by: &str,
    ) -> Result<Entity> {
        let mut entity = self.get_entity(id).await?;
        
        // Update fields
        entity.data = data;
        entity.updated_at = Utc::now();
        entity.updated_by = updated_by.to_string();
        
        // Update in UCG
        self.memory.store_ucg_entity(&entity).await?;
        
        // Persist to database
        self.persist_entity(&entity).await?;
        
        Ok(entity)
    }
    
    /// Delete entity (soft delete)
    pub async fn delete_entity(&self, id: &Uuid, deleted_by: &str) -> Result<()> {
        let entity = self.get_entity(id).await?;
        
        // Check for children
        if self.has_children(id).await? {
            return Err(ReedError::UcgIntegrity {
                reason: "Cannot delete entity with children".to_string(),
            });
        }
        
        // Mark as deleted in data
        let mut data = entity.data.clone();
        data["deleted"] = serde_json::json!(true);
        data["deleted_at"] = serde_json::json!(Utc::now());
        data["deleted_by"] = serde_json::json!(deleted_by);
        
        self.update_entity(id, data, deleted_by).await?;
        
        // Remove from semantic mapping if exists
        if let Some(ref sem_name) = entity.semantic_name {
            let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
            let semantic_key = keys.semantic(&entity.entity_type.to_string(), sem_name.as_str());
            self.redis.del(&[&semantic_key]).await?;
        }
        
        Ok(())
    }
    
    /// List entities by type
    pub async fn list_by_type(
        &self,
        entity_type: &EntityType,
        limit: Option<usize>,
    ) -> Result<Vec<Entity>> {
        // Use type index
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let type_key = keys.type_index(&entity_type.to_string());
        
        let entity_ids = self.redis.smembers(&type_key).await?;
        
        let mut entities = Vec::new();
        let max_items = limit.unwrap_or(entity_ids.len());
        
        for id_str in entity_ids.iter().take(max_items) {
            if let Ok(id) = Uuid::parse_str(id_str) {
                if let Ok(entity) = self.get_entity(&id).await {
                    entities.push(entity);
                }
            }
        }
        
        // Sort by creation date
        entities.sort_by(|a, b| b.created_at.cmp(&a.created_at));
        
        Ok(entities)
    }
}
```

## Cache Operations

```rust
impl EntityManager {
    /// Get entity from Redis cache
    async fn get_from_cache(&self, id: &Uuid) -> Result<Option<Entity>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let entity_key = keys.entity("*", id); // Type wildcard for lookup
        
        // Check if exists
        if !self.redis.exists(&entity_key).await? {
            return Ok(None);
        }
        
        // Get all fields
        let data = self.redis.hgetall(&entity_key).await?;
        
        if data.is_empty() {
            return Ok(None);
        }
        
        // Reconstruct entity
        let entity = self.reconstruct_entity(id, data)?;
        Ok(Some(entity))
    }
    
    /// Reconstruct entity from Redis hash
    fn reconstruct_entity(
        &self,
        id: &Uuid,
        data: HashMap<String, String>,
    ) -> Result<Entity> {
        let entity_type = data.get("type")
            .ok_or_else(|| ReedError::Internal("Missing entity type".to_string()))?;
        
        let entity_data = data.get("data")
            .ok_or_else(|| ReedError::Internal("Missing entity data".to_string()))?;
        
        let semantic_name = data.get("semantic_name")
            .and_then(|s| SemanticName::new(s).ok());
        
        Ok(Entity {
            id: *id,
            entity_type: self.parse_entity_type(entity_type)?,
            semantic_name,
            created_at: Utc::now(), // Will be overridden from DB if needed
            updated_at: Utc::now(),
            created_by: data.get("created_by").unwrap_or(&"system".to_string()).clone(),
            data: serde_json::from_str(entity_data)?,
        })
    }
    
    fn parse_entity_type(&self, type_str: &str) -> Result<EntityType> {
        match type_str {
            "snippet" => Ok(EntityType::Snippet),
            "theme" => Ok(EntityType::Theme),
            "page" => Ok(EntityType::Page),
            "user" => Ok(EntityType::User),
            "plugin" => Ok(EntityType::Plugin),
            other => Ok(EntityType::Custom(other.to_string())),
        }
    }
}
```

## Database Persistence

```rust
impl EntityManager {
    /// Persist entity to PostgreSQL
    async fn persist_entity(&self, entity: &Entity) -> Result<()> {
        // For snippets, store in snippet_content table
        if entity.entity_type == EntityType::Snippet {
            self.persist_snippet(entity).await?;
        }
        
        // Store entity metadata in UCG backup
        self.backup_to_ucg(entity).await?;
        
        Ok(())
    }
    
    /// Get entity from database
    async fn get_from_database(&self, id: &Uuid) -> Result<Entity> {
        // Check UCG backup first
        let query = r#"
            SELECT entity_type, semantic_name, data, created_at
            FROM ucg_entities
            WHERE id = $1
        "#;
        
        let row: Option<(String, Option<String>, serde_json::Value, DateTime<Utc>)> = 
            sqlx::query_as(query)
                .bind(id)
                .fetch_optional(self.db.ucg_pool())
                .await?;
        
        let (type_str, sem_name, data, created_at) = row
            .ok_or_else(|| ReedError::EntityNotFound { id: *id })?;
        
        Ok(Entity {
            id: *id,
            entity_type: self.parse_entity_type(&type_str)?,
            semantic_name: sem_name.and_then(|s| SemanticName::new(&s).ok()),
            created_at,
            updated_at: created_at, // Will be updated if needed
            created_by: "system".to_string(), // Default
            data,
        })
    }
    
    /// Backup entity to UCG database
    async fn backup_to_ucg(&self, entity: &Entity) -> Result<()> {
        let query = r#"
            INSERT INTO ucg_entities (id, entity_type, semantic_name, data, created_at)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (id) DO UPDATE SET
                data = EXCLUDED.data,
                semantic_name = EXCLUDED.semantic_name
        "#;
        
        sqlx::query(query)
            .bind(&entity.id)
            .bind(&entity.entity_type.to_string())
            .bind(entity.semantic_name.as_ref().map(|s| s.as_str()))
            .bind(&entity.data)
            .bind(&entity.created_at)
            .execute(self.db.ucg_pool())
            .await?;
        
        Ok(())
    }
}
```

## Entity Relationships

```rust
impl EntityManager {
    /// Check if entity has children
    async fn has_children(&self, id: &Uuid) -> Result<bool> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let children_key = keys.children(id);
        
        let children = self.redis.smembers(&children_key).await?;
        Ok(!children.is_empty())
    }
    
    /// Get entity children
    pub async fn get_children(&self, parent_id: &Uuid) -> Result<Vec<Entity>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let children_key = keys.children(parent_id);
        
        let child_ids = self.redis.smembers(&children_key).await?;
        
        let mut children = Vec::new();
        for id_str in child_ids {
            if let Ok(id) = Uuid::parse_str(&id_str) {
                if let Ok(entity) = self.get_entity(&id).await {
                    children.push(entity);
                }
            }
        }
        
        Ok(children)
    }
    
    /// Get entity parent
    pub async fn get_parent(&self, child_id: &Uuid) -> Result<Option<Entity>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let parent_key = keys.parent(child_id);
        
        if let Some(parent_id_str) = self.redis.get(&parent_key).await? {
            let parent_id = Uuid::parse_str(&parent_id_str)?;
            Ok(Some(self.get_entity(&parent_id).await?))
        } else {
            Ok(None)
        }
    }
}
```

## Entity Search

```rust
impl EntityManager {
    /// Search entities by data field
    pub async fn search_entities(
        &self,
        entity_type: Option<&EntityType>,
        field: &str,
        value: &str,
    ) -> Result<Vec<Entity>> {
        // This is a simple implementation
        // Real search would use the search index
        
        let entities = if let Some(et) = entity_type {
            self.list_by_type(et, None).await?
        } else {
            // Get all entity types
            let mut all = Vec::new();
            for et in &[EntityType::Snippet, EntityType::Page, EntityType::Theme] {
                all.extend(self.list_by_type(et, None).await?);
            }
            all
        };
        
        // Filter by field value
        let filtered: Vec<Entity> = entities.into_iter()
            .filter(|e| {
                e.data.get(field)
                    .and_then(|v| v.as_str())
                    .map(|s| s.contains(value))
                    .unwrap_or(false)
            })
            .collect();
        
        Ok(filtered)
    }
}
```

## Summary

Dieses Entity Management bietet:
- **Unified Entity Model** - Alles ist eine Entity
- **Semantic Names** - Menschenlesbare Referenzen mit $prefix
- **Redis Caching** - Schneller Zugriff auf UCG Struktur
- **PostgreSQL Backup** - Persistente Speicherung
- **Relationship Management** - Parent/Child Beziehungen

Das UCG System eliminiert komplexe JOINs und bietet konsistente Performance.
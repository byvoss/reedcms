# UCG Association Handling

## Overview

Association Management für das Universal Content Graph System. Definiert Beziehungen zwischen Entities.

## Association Manager

```rust
use std::collections::HashMap;

/// Manages associations between entities in UCG
pub struct AssociationManager {
    db: Arc<DatabaseManager>,
    redis: Arc<RedisManager>,
    memory: Arc<MemoryManager>,
}

impl AssociationManager {
    pub fn new(
        db: Arc<DatabaseManager>,
        redis: Arc<RedisManager>,
        memory: Arc<MemoryManager>,
    ) -> Self {
        Self { db, redis, memory }
    }
    
    /// Create association between entities
    pub async fn create_association(
        &self,
        parent_id: Uuid,
        child_id: Uuid,
        association_type: AssociationType,
        weight: Option<i32>,
    ) -> Result<Association> {
        // Verify both entities exist
        self.verify_entities_exist(&parent_id, &child_id).await?;
        
        // Check for circular reference
        if self.would_create_cycle(&parent_id, &child_id).await? {
            return Err(ReedError::UcgIntegrity {
                reason: "Association would create circular reference".to_string(),
            });
        }
        
        // Generate path
        let path = self.generate_path(&parent_id, &child_id).await?;
        
        // Create association
        let association = Association {
            id: Uuid::new_v4(),
            parent_id,
            child_id,
            association_type,
            weight: weight.unwrap_or(0),
            path,
        };
        
        // Store in UCG
        self.memory.store_ucg_association(&association).await?;
        
        // Persist to database
        self.persist_association(&association).await?;
        
        tracing::info!(
            "Created association: {} -> {} ({})",
            parent_id,
            child_id,
            association.path
        );
        
        Ok(association)
    }
    
    /// Generate UCG path for association
    async fn generate_path(&self, parent_id: &Uuid, child_id: &Uuid) -> Result<String> {
        // Get parent's path or create root path
        let parent_path = if let Some(parent_assoc) = self.get_parent_association(parent_id).await? {
            parent_assoc.path
        } else {
            // This is a root entity
            "content".to_string()
        };
        
        // Count existing children to determine position
        let position = self.count_children(parent_id).await? + 1;
        
        // Build path like "content.1.2"
        Ok(format!("{}.{}", parent_path, position))
    }
}
```

## Association Queries

```rust
impl AssociationManager {
    /// Get association by path
    pub async fn get_by_path(&self, path: &str) -> Result<Association> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let assoc_key = keys.association(path);
        
        let data = self.redis.hgetall(&assoc_key).await?;
        
        if data.is_empty() {
            return Err(ReedError::InvalidPath {
                path: path.to_string(),
            });
        }
        
        self.reconstruct_association(path, data)
    }
    
    /// Get associations for parent
    pub async fn get_children_associations(&self, parent_id: &Uuid) -> Result<Vec<Association>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let children_key = keys.children(parent_id);
        
        let child_ids = self.redis.smembers(&children_key).await?;
        
        let mut associations = Vec::new();
        for child_id_str in child_ids {
            if let Ok(child_id) = Uuid::parse_str(&child_id_str) {
                if let Some(assoc) = self.find_association(parent_id, &child_id).await? {
                    associations.push(assoc);
                }
            }
        }
        
        // Sort by weight and position
        associations.sort_by_key(|a| (a.weight, a.path.clone()));
        
        Ok(associations)
    }
    
    /// Get parent association
    async fn get_parent_association(&self, child_id: &Uuid) -> Result<Option<Association>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let parent_key = keys.parent(child_id);
        
        if let Some(parent_id_str) = self.redis.get(&parent_key).await? {
            let parent_id = Uuid::parse_str(&parent_id_str)?;
            self.find_association(&parent_id, child_id).await
        } else {
            Ok(None)
        }
    }
    
    /// Find specific association
    async fn find_association(&self, parent_id: &Uuid, child_id: &Uuid) -> Result<Option<Association>> {
        // Search by pattern
        let pattern = format!("assoc:*");
        let mut cursor = 0;
        
        loop {
            let (new_cursor, keys): (u64, Vec<String>) = 
                self.redis.scan_match(&pattern, cursor, 100).await?;
            
            for key in keys {
                let data = self.redis.hgetall(&key).await?;
                if let Ok(assoc) = self.reconstruct_association(&key, data) {
                    if assoc.parent_id == *parent_id && assoc.child_id == *child_id {
                        return Ok(Some(assoc));
                    }
                }
            }
            
            cursor = new_cursor;
            if cursor == 0 {
                break;
            }
        }
        
        Ok(None)
    }
}
```

## Association Operations

```rust
impl AssociationManager {
    /// Update association weight
    pub async fn update_weight(
        &self,
        association_id: &Uuid,
        new_weight: i32,
    ) -> Result<()> {
        // Find association by ID
        let assoc = self.get_association(association_id).await?;
        
        // Update weight in Redis
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let assoc_key = keys.association(&assoc.path);
        
        self.redis.hset(&assoc_key, "weight", &new_weight.to_string()).await?;
        
        // Update in database
        self.update_association_weight_db(association_id, new_weight).await?;
        
        Ok(())
    }
    
    /// Remove association
    pub async fn remove_association(&self, association_id: &Uuid) -> Result<()> {
        let assoc = self.get_association(association_id).await?;
        
        // Remove from Redis
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        
        // Remove association
        let assoc_key = keys.association(&assoc.path);
        self.redis.del(&[&assoc_key]).await?;
        
        // Remove from parent's children set
        let children_key = keys.children(&assoc.parent_id);
        self.redis.srem(&children_key, &[&assoc.child_id.to_string()]).await?;
        
        // Remove child's parent reference
        let parent_key = keys.parent(&assoc.child_id);
        self.redis.del(&[&parent_key]).await?;
        
        // Remove from database
        self.delete_association_db(association_id).await?;
        
        tracing::info!("Removed association: {}", assoc.path);
        
        Ok(())
    }
    
    /// Move association (change parent)
    pub async fn move_association(
        &self,
        child_id: &Uuid,
        new_parent_id: &Uuid,
        new_weight: Option<i32>,
    ) -> Result<Association> {
        // Get current association
        let current = self.get_parent_association(child_id).await?
            .ok_or_else(|| ReedError::UcgIntegrity {
                reason: "Entity has no parent".to_string(),
            })?;
        
        // Remove old association
        self.remove_association(&current.id).await?;
        
        // Create new association
        self.create_association(
            *new_parent_id,
            *child_id,
            current.association_type,
            new_weight.or(Some(current.weight)),
        ).await
    }
}
```

## Cycle Detection

```rust
impl AssociationManager {
    /// Check if association would create cycle
    async fn would_create_cycle(&self, parent_id: &Uuid, child_id: &Uuid) -> Result<bool> {
        // If child is ancestor of parent, it would create cycle
        self.is_ancestor(child_id, parent_id).await
    }
    
    /// Check if entity is ancestor of another
    async fn is_ancestor(&self, potential_ancestor: &Uuid, descendant: &Uuid) -> Result<bool> {
        let mut current = *descendant;
        let mut visited = HashSet::new();
        
        loop {
            // Avoid infinite loops
            if !visited.insert(current) {
                return Ok(false);
            }
            
            // Get parent
            if let Some(parent_assoc) = self.get_parent_association(&current).await? {
                if parent_assoc.parent_id == *potential_ancestor {
                    return Ok(true);
                }
                current = parent_assoc.parent_id;
            } else {
                // Reached root
                return Ok(false);
            }
        }
    }
}
```

## Database Persistence

```rust
impl AssociationManager {
    /// Persist association to database
    async fn persist_association(&self, assoc: &Association) -> Result<()> {
        let query = r#"
            INSERT INTO ucg_associations 
            (id, parent_id, child_id, association_type, path, weight)
            VALUES ($1, $2, $3, $4, $5, $6)
            ON CONFLICT (id) DO UPDATE SET
                weight = EXCLUDED.weight,
                path = EXCLUDED.path
        "#;
        
        sqlx::query(query)
            .bind(&assoc.id)
            .bind(&assoc.parent_id)
            .bind(&assoc.child_id)
            .bind(&assoc.association_type.to_string())
            .bind(&assoc.path)
            .bind(&assoc.weight)
            .execute(self.db.ucg_pool())
            .await?;
        
        Ok(())
    }
    
    /// Update association weight in database
    async fn update_association_weight_db(&self, id: &Uuid, weight: i32) -> Result<()> {
        sqlx::query("UPDATE ucg_associations SET weight = $1 WHERE id = $2")
            .bind(weight)
            .bind(id)
            .execute(self.db.ucg_pool())
            .await?;
        
        Ok(())
    }
    
    /// Delete association from database
    async fn delete_association_db(&self, id: &Uuid) -> Result<()> {
        sqlx::query("DELETE FROM ucg_associations WHERE id = $1")
            .bind(id)
            .execute(self.db.ucg_pool())
            .await?;
        
        Ok(())
    }
}
```

## Helper Methods

```rust
impl AssociationManager {
    /// Verify entities exist
    async fn verify_entities_exist(&self, parent_id: &Uuid, child_id: &Uuid) -> Result<()> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        
        let parent_exists = self.redis.exists(&keys.entity("*", parent_id)).await?;
        let child_exists = self.redis.exists(&keys.entity("*", child_id)).await?;
        
        if !parent_exists {
            return Err(ReedError::EntityNotFound { id: *parent_id });
        }
        
        if !child_exists {
            return Err(ReedError::EntityNotFound { id: *child_id });
        }
        
        Ok(())
    }
    
    /// Count children of entity
    async fn count_children(&self, parent_id: &Uuid) -> Result<usize> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let children_key = keys.children(parent_id);
        
        let children = self.redis.smembers(&children_key).await?;
        Ok(children.len())
    }
    
    /// Reconstruct association from Redis data
    fn reconstruct_association(
        &self,
        path: &str,
        data: HashMap<String, String>,
    ) -> Result<Association> {
        let parent_id = data.get("parent_id")
            .and_then(|s| Uuid::parse_str(s).ok())
            .ok_or_else(|| ReedError::Internal("Missing parent_id".to_string()))?;
        
        let child_id = data.get("child_id")
            .and_then(|s| Uuid::parse_str(s).ok())
            .ok_or_else(|| ReedError::Internal("Missing child_id".to_string()))?;
        
        let association_type = data.get("type")
            .map(|s| self.parse_association_type(s))
            .unwrap_or(AssociationType::Contains);
        
        let weight = data.get("weight")
            .and_then(|s| s.parse::<i32>().ok())
            .unwrap_or(0);
        
        Ok(Association {
            id: Uuid::new_v4(), // Generated
            parent_id,
            child_id,
            association_type,
            weight,
            path: path.to_string(),
        })
    }
    
    fn parse_association_type(&self, type_str: &str) -> AssociationType {
        match type_str {
            "contains" => AssociationType::Contains,
            "references" => AssociationType::References,
            "extends" => AssociationType::Extends,
            other => AssociationType::Custom(other.to_string()),
        }
    }
}
```

## Summary

Dieses Association Management bietet:
- **Path-basierte Hierarchie** - content.1.1 Style
- **Cycle Detection** - Verhindert zirkuläre Referenzen
- **Weight-basierte Sortierung** - Für geordnete Listen
- **Flexible Association Types** - Contains, References, Extends
- **Integrity Checks** - Sichere Operationen

Associations sind das Bindeglied im UCG System.
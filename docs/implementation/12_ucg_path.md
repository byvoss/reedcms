# UCG Path Resolution

## Overview

Path Resolution für das Universal Content Graph System. Verarbeitet Pfade wie "content.1.1" für hierarchische Navigation.

## Path Resolver

```rust
use std::str::FromStr;

/// Resolves UCG paths to entities and associations
pub struct PathResolver {
    entity_manager: Arc<EntityManager>,
    association_manager: Arc<AssociationManager>,
    redis: Arc<RedisManager>,
}

impl PathResolver {
    pub fn new(
        entity_manager: Arc<EntityManager>,
        association_manager: Arc<AssociationManager>,
        redis: Arc<RedisManager>,
    ) -> Self {
        Self {
            entity_manager,
            association_manager,
            redis,
        }
    }
    
    /// Resolve path to entity
    pub async fn resolve_path(&self, path: &str) -> Result<PathResolution> {
        // Validate path format
        let parsed_path = UcgPath::parse(path)?;
        
        match parsed_path {
            UcgPath::Root => self.resolve_root().await,
            UcgPath::Direct(segments) => self.resolve_segments(&segments).await,
            UcgPath::Semantic(name) => self.resolve_semantic(name).await,
        }
    }
    
    /// Resolve root path
    async fn resolve_root(&self) -> Result<PathResolution> {
        // Root entities have no parent
        let entities = self.find_root_entities().await?;
        
        Ok(PathResolution {
            path: "content".to_string(),
            entity: None,
            children: entities,
            association: None,
        })
    }
    
    /// Resolve path segments
    async fn resolve_segments(&self, segments: &[PathSegment]) -> Result<PathResolution> {
        let mut current_path = "content".to_string();
        let mut current_parent: Option<Uuid> = None;
        let mut current_entity: Option<Entity> = None;
        let mut current_association: Option<Association> = None;
        
        for segment in segments {
            let (entity, assoc) = match segment {
                PathSegment::Index(idx) => {
                    self.resolve_by_index(&current_parent, *idx).await?
                }
                PathSegment::Name(name) => {
                    self.resolve_by_name(&current_parent, name).await?
                }
            };
            
            current_path = format!("{}.{}", current_path, segment);
            current_parent = Some(entity.id);
            current_entity = Some(entity);
            current_association = Some(assoc);
        }
        
        // Get children of final entity
        let children = if let Some(ref entity) = current_entity {
            self.entity_manager.get_children(&entity.id).await?
        } else {
            vec![]
        };
        
        Ok(PathResolution {
            path: current_path,
            entity: current_entity,
            children,
            association: current_association,
        })
    }
    
    /// Resolve semantic name
    async fn resolve_semantic(&self, name: SemanticName) -> Result<PathResolution> {
        // Try different entity types
        for entity_type in &[EntityType::Snippet, EntityType::Page, EntityType::Theme] {
            if let Ok(entity) = self.entity_manager
                .get_by_semantic_name(entity_type, &name)
                .await 
            {
                // Find path to this entity
                let path = self.find_path_to_entity(&entity.id).await?;
                let association = self.get_entity_association(&entity.id).await?;
                let children = self.entity_manager.get_children(&entity.id).await?;
                
                return Ok(PathResolution {
                    path,
                    entity: Some(entity),
                    children,
                    association,
                });
            }
        }
        
        Err(ReedError::EntityNotFound { id: Uuid::nil() })
    }
}
```

## Path Types

```rust
/// UCG path representation
#[derive(Debug, Clone, PartialEq)]
pub enum UcgPath {
    Root,
    Direct(Vec<PathSegment>),
    Semantic(SemanticName),
}

#[derive(Debug, Clone, PartialEq)]
pub enum PathSegment {
    Index(usize),
    Name(String),
}

impl UcgPath {
    /// Parse path string
    pub fn parse(path: &str) -> Result<Self> {
        if path.is_empty() || path == "content" {
            return Ok(UcgPath::Root);
        }
        
        if path.starts_with('$') {
            // Semantic name
            let name = SemanticName::new(path)?;
            return Ok(UcgPath::Semantic(name));
        }
        
        // Direct path
        let segments = Self::parse_segments(path)?;
        Ok(UcgPath::Direct(segments))
    }
    
    fn parse_segments(path: &str) -> Result<Vec<PathSegment>> {
        let mut segments = Vec::new();
        
        for part in path.split('.') {
            if part == "content" {
                continue; // Skip root
            }
            
            let segment = if let Ok(index) = part.parse::<usize>() {
                PathSegment::Index(index)
            } else {
                PathSegment::Name(part.to_string())
            };
            
            segments.push(segment);
        }
        
        if segments.is_empty() {
            return Err(ReedError::InvalidPath {
                path: path.to_string(),
            });
        }
        
        Ok(segments)
    }
}

impl fmt::Display for PathSegment {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PathSegment::Index(idx) => write!(f, "{}", idx),
            PathSegment::Name(name) => write!(f, "{}", name),
        }
    }
}
```

## Resolution Result

```rust
/// Result of path resolution
#[derive(Debug, Clone)]
pub struct PathResolution {
    pub path: String,
    pub entity: Option<Entity>,
    pub children: Vec<Entity>,
    pub association: Option<Association>,
}

impl PathResolution {
    /// Check if path exists
    pub fn exists(&self) -> bool {
        self.entity.is_some()
    }
    
    /// Get entity ID if resolved
    pub fn entity_id(&self) -> Option<Uuid> {
        self.entity.as_ref().map(|e| e.id)
    }
    
    /// Get child paths
    pub fn child_paths(&self) -> Vec<String> {
        self.children.iter().enumerate()
            .map(|(idx, _)| format!("{}.{}", self.path, idx + 1))
            .collect()
    }
}
```

## Resolution Methods

```rust
impl PathResolver {
    /// Resolve by position index
    async fn resolve_by_index(
        &self,
        parent_id: &Option<Uuid>,
        index: usize,
    ) -> Result<(Entity, Association)> {
        let associations = if let Some(pid) = parent_id {
            self.association_manager.get_children_associations(pid).await?
        } else {
            self.find_root_associations().await?
        };
        
        // Get by position (1-based index)
        let assoc = associations.get(index - 1)
            .ok_or_else(|| ReedError::InvalidPath {
                path: format!("index {}", index),
            })?;
        
        let entity = self.entity_manager.get_entity(&assoc.child_id).await?;
        
        Ok((entity, assoc.clone()))
    }
    
    /// Resolve by name
    async fn resolve_by_name(
        &self,
        parent_id: &Option<Uuid>,
        name: &str,
    ) -> Result<(Entity, Association)> {
        let children = if let Some(pid) = parent_id {
            self.entity_manager.get_children(pid).await?
        } else {
            self.find_root_entities().await?
        };
        
        // Find by name in data
        for child in children {
            if let Some(child_name) = child.data.get("name").and_then(|v| v.as_str()) {
                if child_name == name {
                    let assoc = self.get_entity_association(&child.id).await?
                        .ok_or_else(|| ReedError::UcgIntegrity {
                            reason: "Entity has no association".to_string(),
                        })?;
                    
                    return Ok((child, assoc));
                }
            }
        }
        
        Err(ReedError::InvalidPath {
            path: format!("name '{}'", name),
        })
    }
    
    /// Find path to entity
    async fn find_path_to_entity(&self, entity_id: &Uuid) -> Result<String> {
        let mut path_parts = Vec::new();
        let mut current_id = *entity_id;
        
        // Walk up the tree
        loop {
            if let Some(assoc) = self.get_entity_association(&current_id).await? {
                // Extract position from path
                let parts: Vec<&str> = assoc.path.split('.').collect();
                if let Some(last) = parts.last() {
                    path_parts.push(last.to_string());
                }
                
                current_id = assoc.parent_id;
            } else {
                // Reached root
                break;
            }
        }
        
        // Reverse to get top-down path
        path_parts.reverse();
        
        if path_parts.is_empty() {
            Ok("content".to_string())
        } else {
            Ok(format!("content.{}", path_parts.join(".")))
        }
    }
}
```

## Helper Methods

```rust
impl PathResolver {
    /// Find root entities (no parent)
    async fn find_root_entities(&self) -> Result<Vec<Entity>> {
        // Get all entities and filter those without parents
        let mut root_entities = Vec::new();
        
        for entity_type in &[EntityType::Page, EntityType::Snippet] {
            let entities = self.entity_manager.list_by_type(entity_type, None).await?;
            
            for entity in entities {
                if self.get_entity_association(&entity.id).await?.is_none() {
                    root_entities.push(entity);
                }
            }
        }
        
        // Sort by creation date
        root_entities.sort_by(|a, b| a.created_at.cmp(&b.created_at));
        
        Ok(root_entities)
    }
    
    /// Find root associations
    async fn find_root_associations(&self) -> Result<Vec<Association>> {
        let root_entities = self.find_root_entities().await?;
        let mut associations = Vec::new();
        
        for (idx, entity) in root_entities.iter().enumerate() {
            // Create virtual association for root entities
            associations.push(Association {
                id: Uuid::new_v4(),
                parent_id: Uuid::nil(), // No parent
                child_id: entity.id,
                association_type: AssociationType::Contains,
                weight: idx as i32,
                path: format!("content.{}", idx + 1),
            });
        }
        
        Ok(associations)
    }
    
    /// Get entity's association
    async fn get_entity_association(&self, entity_id: &Uuid) -> Result<Option<Association>> {
        let keys = RedisKeyBuilder::new(&self.redis.config.key_prefix);
        let parent_key = keys.parent(entity_id);
        
        if let Some(parent_id_str) = self.redis.get(&parent_key).await? {
            let parent_id = Uuid::parse_str(&parent_id_str)?;
            self.association_manager.find_association(&parent_id, entity_id).await
        } else {
            Ok(None)
        }
    }
}
```

## Path Navigation

```rust
/// Navigate between paths
pub struct PathNavigator {
    resolver: Arc<PathResolver>,
}

impl PathNavigator {
    pub fn new(resolver: Arc<PathResolver>) -> Self {
        Self { resolver }
    }
    
    /// Get parent path
    pub async fn parent(&self, path: &str) -> Result<Option<String>> {
        let parts: Vec<&str> = path.split('.').collect();
        
        if parts.len() <= 1 {
            return Ok(None);
        }
        
        let parent_parts = &parts[..parts.len() - 1];
        Ok(Some(parent_parts.join(".")))
    }
    
    /// Get sibling paths
    pub async fn siblings(&self, path: &str) -> Result<Vec<String>> {
        if let Some(parent_path) = self.parent(path).await? {
            let parent_resolution = self.resolver.resolve_path(&parent_path).await?;
            Ok(parent_resolution.child_paths())
        } else {
            // Root level siblings
            let root = self.resolver.resolve_path("content").await?;
            Ok(root.child_paths())
        }
    }
    
    /// Get next sibling
    pub async fn next_sibling(&self, path: &str) -> Result<Option<String>> {
        let siblings = self.siblings(path).await?;
        let current_idx = siblings.iter().position(|p| p == path);
        
        if let Some(idx) = current_idx {
            Ok(siblings.get(idx + 1).cloned())
        } else {
            Ok(None)
        }
    }
    
    /// Get previous sibling
    pub async fn previous_sibling(&self, path: &str) -> Result<Option<String>> {
        let siblings = self.siblings(path).await?;
        let current_idx = siblings.iter().position(|p| p == path);
        
        if let Some(idx) = current_idx {
            if idx > 0 {
                Ok(siblings.get(idx - 1).cloned())
            } else {
                Ok(None)
            }
        } else {
            Ok(None)
        }
    }
}
```

## Summary

Dieses Path Resolution System bietet:
- **Hierarchische Navigation** - content.1.1 Style Paths
- **Semantic Resolution** - $name zu Path Auflösung
- **Flexible Segmente** - Index oder Name basiert
- **Path Navigation** - Parent, Siblings, Next/Previous
- **Efficient Lookups** - Redis-backed Resolution

Das Path System macht UCG navigierbar und verständlich.
# UCG Query Operations

## Overview

Query Operations für das Universal Content Graph System. Ermöglicht effiziente Abfragen ohne komplexe JOINs.

## Query Engine

```rust
use std::collections::HashSet;

/// UCG query engine for efficient data retrieval
pub struct UcgQueryEngine {
    entity_manager: Arc<EntityManager>,
    association_manager: Arc<AssociationManager>,
    path_resolver: Arc<PathResolver>,
    redis: Arc<RedisManager>,
}

impl UcgQueryEngine {
    pub fn new(
        entity_manager: Arc<EntityManager>,
        association_manager: Arc<AssociationManager>,
        path_resolver: Arc<PathResolver>,
        redis: Arc<RedisManager>,
    ) -> Self {
        Self {
            entity_manager,
            association_manager,
            path_resolver,
            redis,
        }
    }
    
    /// Execute UCG query
    pub async fn query(&self, query: UcgQuery) -> Result<QueryResult> {
        let start = std::time::Instant::now();
        
        // Get base entities
        let mut entities = self.get_base_entities(&query).await?;
        
        // Apply filters
        if !query.filters.is_empty() {
            entities = self.apply_filters(entities, &query.filters).await?;
        }
        
        // Apply sorting
        if let Some(ref sort) = query.sort {
            entities = self.apply_sort(entities, sort);
        }
        
        // Apply pagination
        let total = entities.len();
        if let Some(ref pagination) = query.pagination {
            entities = self.apply_pagination(entities, pagination);
        }
        
        // Load associations if requested
        let mut result_entities = Vec::new();
        for entity in entities {
            let enriched = if query.include_children {
                self.enrich_with_children(entity).await?
            } else {
                EntityWithRelations {
                    entity,
                    children: vec![],
                    parent: None,
                    associations: vec![],
                }
            };
            result_entities.push(enriched);
        }
        
        let elapsed = start.elapsed();
        
        Ok(QueryResult {
            entities: result_entities,
            total,
            query_time: elapsed,
        })
    }
}
```

## Query Types

```rust
/// UCG query definition
#[derive(Debug, Clone)]
pub struct UcgQuery {
    pub base: QueryBase,
    pub filters: Vec<QueryFilter>,
    pub sort: Option<QuerySort>,
    pub pagination: Option<QueryPagination>,
    pub include_children: bool,
    pub include_parent: bool,
}

#[derive(Debug, Clone)]
pub enum QueryBase {
    All,
    Type(EntityType),
    Path(String),
    Parent(Uuid),
    Semantic(SemanticName),
}

#[derive(Debug, Clone)]
pub struct QueryFilter {
    pub field: FilterField,
    pub operator: FilterOperator,
    pub value: serde_json::Value,
}

#[derive(Debug, Clone)]
pub enum FilterField {
    Status,
    CreatedBy,
    UpdatedAt,
    DataField(String),
}

#[derive(Debug, Clone)]
pub enum FilterOperator {
    Equals,
    NotEquals,
    Contains,
    GreaterThan,
    LessThan,
    In,
}

#[derive(Debug, Clone)]
pub struct QuerySort {
    pub field: SortField,
    pub direction: SortDirection,
}

#[derive(Debug, Clone)]
pub enum SortField {
    CreatedAt,
    UpdatedAt,
    Weight,
    DataField(String),
}

#[derive(Debug, Clone)]
pub enum SortDirection {
    Ascending,
    Descending,
}

#[derive(Debug, Clone)]
pub struct QueryPagination {
    pub offset: usize,
    pub limit: usize,
}
```

## Query Execution

```rust
impl UcgQueryEngine {
    /// Get base entities for query
    async fn get_base_entities(&self, query: &UcgQuery) -> Result<Vec<Entity>> {
        match &query.base {
            QueryBase::All => {
                // Get all entities (use with caution)
                let mut all = Vec::new();
                for entity_type in &[EntityType::Snippet, EntityType::Page, EntityType::Theme] {
                    all.extend(self.entity_manager.list_by_type(entity_type, None).await?);
                }
                Ok(all)
            }
            
            QueryBase::Type(entity_type) => {
                self.entity_manager.list_by_type(entity_type, None).await
            }
            
            QueryBase::Path(path) => {
                let resolution = self.path_resolver.resolve_path(path).await?;
                if let Some(entity) = resolution.entity {
                    Ok(vec![entity])
                } else {
                    Ok(resolution.children)
                }
            }
            
            QueryBase::Parent(parent_id) => {
                self.entity_manager.get_children(parent_id).await
            }
            
            QueryBase::Semantic(name) => {
                // Try all entity types
                for entity_type in &[EntityType::Snippet, EntityType::Page, EntityType::Theme] {
                    if let Ok(entity) = self.entity_manager
                        .get_by_semantic_name(entity_type, name)
                        .await
                    {
                        return Ok(vec![entity]);
                    }
                }
                Ok(vec![])
            }
        }
    }
    
    /// Apply filters to entities
    async fn apply_filters(
        &self,
        entities: Vec<Entity>,
        filters: &[QueryFilter],
    ) -> Result<Vec<Entity>> {
        let mut filtered = entities;
        
        for filter in filters {
            filtered = filtered.into_iter()
                .filter(|entity| self.matches_filter(entity, filter))
                .collect();
        }
        
        Ok(filtered)
    }
    
    /// Check if entity matches filter
    fn matches_filter(&self, entity: &Entity, filter: &QueryFilter) -> bool {
        let value = match &filter.field {
            FilterField::Status => {
                entity.data.get("status").cloned()
                    .unwrap_or(serde_json::json!("draft"))
            }
            FilterField::CreatedBy => {
                serde_json::json!(&entity.created_by)
            }
            FilterField::UpdatedAt => {
                serde_json::json!(entity.updated_at.to_rfc3339())
            }
            FilterField::DataField(field) => {
                entity.data.get(field).cloned()
                    .unwrap_or(serde_json::Value::Null)
            }
        };
        
        match &filter.operator {
            FilterOperator::Equals => value == filter.value,
            FilterOperator::NotEquals => value != filter.value,
            FilterOperator::Contains => {
                if let (Some(str_val), Some(search)) = (value.as_str(), filter.value.as_str()) {
                    str_val.contains(search)
                } else {
                    false
                }
            }
            FilterOperator::GreaterThan => {
                self.compare_values(&value, &filter.value) > 0
            }
            FilterOperator::LessThan => {
                self.compare_values(&value, &filter.value) < 0
            }
            FilterOperator::In => {
                if let Some(array) = filter.value.as_array() {
                    array.contains(&value)
                } else {
                    false
                }
            }
        }
    }
}
```

## Sorting and Pagination

```rust
impl UcgQueryEngine {
    /// Apply sorting to entities
    fn apply_sort(&self, mut entities: Vec<Entity>, sort: &QuerySort) -> Vec<Entity> {
        entities.sort_by(|a, b| {
            let cmp = match &sort.field {
                SortField::CreatedAt => a.created_at.cmp(&b.created_at),
                SortField::UpdatedAt => a.updated_at.cmp(&b.updated_at),
                SortField::Weight => {
                    // Get weight from associations if available
                    std::cmp::Ordering::Equal
                }
                SortField::DataField(field) => {
                    let a_val = a.data.get(field);
                    let b_val = b.data.get(field);
                    self.compare_json_values(a_val, b_val)
                }
            };
            
            match sort.direction {
                SortDirection::Ascending => cmp,
                SortDirection::Descending => cmp.reverse(),
            }
        });
        
        entities
    }
    
    /// Apply pagination
    fn apply_pagination(
        &self,
        entities: Vec<Entity>,
        pagination: &QueryPagination,
    ) -> Vec<Entity> {
        entities.into_iter()
            .skip(pagination.offset)
            .take(pagination.limit)
            .collect()
    }
    
    /// Compare JSON values for sorting
    fn compare_json_values(
        &self,
        a: Option<&serde_json::Value>,
        b: Option<&serde_json::Value>,
    ) -> std::cmp::Ordering {
        match (a, b) {
            (None, None) => std::cmp::Ordering::Equal,
            (None, Some(_)) => std::cmp::Ordering::Less,
            (Some(_), None) => std::cmp::Ordering::Greater,
            (Some(a), Some(b)) => {
                // Compare based on type
                match (a, b) {
                    (serde_json::Value::Number(n1), serde_json::Value::Number(n2)) => {
                        n1.as_f64().partial_cmp(&n2.as_f64()).unwrap_or(std::cmp::Ordering::Equal)
                    }
                    (serde_json::Value::String(s1), serde_json::Value::String(s2)) => {
                        s1.cmp(s2)
                    }
                    _ => std::cmp::Ordering::Equal,
                }
            }
        }
    }
    
    /// Generic value comparison
    fn compare_values(&self, a: &serde_json::Value, b: &serde_json::Value) -> i8 {
        match self.compare_json_values(Some(a), Some(b)) {
            std::cmp::Ordering::Less => -1,
            std::cmp::Ordering::Equal => 0,
            std::cmp::Ordering::Greater => 1,
        }
    }
}
```

## Relationship Loading

```rust
impl UcgQueryEngine {
    /// Enrich entity with relationships
    async fn enrich_with_children(&self, entity: Entity) -> Result<EntityWithRelations> {
        let children = self.entity_manager.get_children(&entity.id).await?;
        let parent = if let Some(parent_assoc) = self.association_manager
            .get_parent_association(&entity.id).await? 
        {
            Some(self.entity_manager.get_entity(&parent_assoc.parent_id).await?)
        } else {
            None
        };
        
        let associations = self.association_manager
            .get_children_associations(&entity.id).await?;
        
        Ok(EntityWithRelations {
            entity,
            children,
            parent,
            associations,
        })
    }
}

/// Entity with loaded relationships
#[derive(Debug, Clone)]
pub struct EntityWithRelations {
    pub entity: Entity,
    pub children: Vec<Entity>,
    pub parent: Option<Entity>,
    pub associations: Vec<Association>,
}

/// Query result
#[derive(Debug, Clone)]
pub struct QueryResult {
    pub entities: Vec<EntityWithRelations>,
    pub total: usize,
    pub query_time: Duration,
}
```

## Query Builder

```rust
/// Fluent query builder
pub struct QueryBuilder {
    query: UcgQuery,
}

impl QueryBuilder {
    pub fn new() -> Self {
        Self {
            query: UcgQuery {
                base: QueryBase::All,
                filters: vec![],
                sort: None,
                pagination: None,
                include_children: false,
                include_parent: false,
            },
        }
    }
    
    pub fn from_type(mut self, entity_type: EntityType) -> Self {
        self.query.base = QueryBase::Type(entity_type);
        self
    }
    
    pub fn from_path(mut self, path: &str) -> Self {
        self.query.base = QueryBase::Path(path.to_string());
        self
    }
    
    pub fn from_parent(mut self, parent_id: Uuid) -> Self {
        self.query.base = QueryBase::Parent(parent_id);
        self
    }
    
    pub fn filter(mut self, field: FilterField, op: FilterOperator, value: serde_json::Value) -> Self {
        self.query.filters.push(QueryFilter {
            field,
            operator: op,
            value,
        });
        self
    }
    
    pub fn sort_by(mut self, field: SortField, direction: SortDirection) -> Self {
        self.query.sort = Some(QuerySort { field, direction });
        self
    }
    
    pub fn paginate(mut self, offset: usize, limit: usize) -> Self {
        self.query.pagination = Some(QueryPagination { offset, limit });
        self
    }
    
    pub fn with_children(mut self) -> Self {
        self.query.include_children = true;
        self
    }
    
    pub fn with_parent(mut self) -> Self {
        self.query.include_parent = true;
        self
    }
    
    pub fn build(self) -> UcgQuery {
        self.query
    }
}

/// Usage example
async fn example_queries(engine: &UcgQueryEngine) -> Result<()> {
    // Get all published snippets
    let query = QueryBuilder::new()
        .from_type(EntityType::Snippet)
        .filter(
            FilterField::Status,
            FilterOperator::Equals,
            serde_json::json!("published")
        )
        .sort_by(SortField::CreatedAt, SortDirection::Descending)
        .paginate(0, 20)
        .build();
    
    let result = engine.query(query).await?;
    println!("Found {} snippets in {:?}", result.total, result.query_time);
    
    Ok(())
}
```

## Summary

Dieses Query System bietet:
- **Flexible Queries** ohne komplexe JOINs
- **Efficient Filtering** mit verschiedenen Operatoren
- **Sorting & Pagination** für große Datenmengen
- **Relationship Loading** optional und performant
- **Fluent Query Builder** für einfache Nutzung

UCG Queries sind schnell und vorhersehbar - genau wie ReedCMS es verspricht.
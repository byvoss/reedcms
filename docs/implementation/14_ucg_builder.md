# UCG Graph Builder

## Overview

Graph Builder für das Universal Content Graph System. Erstellt und verwaltet die Graphstruktur effizient.

## Graph Builder Core

```rust
use std::collections::{HashMap, VecDeque};

/// Builds and manages the UCG graph structure
pub struct UcgGraphBuilder {
    entity_manager: Arc<EntityManager>,
    association_manager: Arc<AssociationManager>,
    path_resolver: Arc<PathResolver>,
    redis: Arc<RedisManager>,
}

impl UcgGraphBuilder {
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
    
    /// Build complete graph from root
    pub async fn build_graph(&self) -> Result<UcgGraph> {
        let start = std::time::Instant::now();
        
        // Find all root entities
        let roots = self.find_root_nodes().await?;
        
        // Build graph structure
        let mut graph = UcgGraph::new();
        
        for root in roots {
            let node = self.build_node_recursive(&root, 0).await?;
            graph.add_root(node);
        }
        
        // Build indices
        graph.build_indices();
        
        let elapsed = start.elapsed();
        tracing::info!(
            "Built UCG graph: {} nodes in {:?}",
            graph.node_count(),
            elapsed
        );
        
        Ok(graph)
    }
    
    /// Build node with all children recursively
    async fn build_node_recursive(
        &self,
        entity: &Entity,
        depth: usize,
    ) -> Result<GraphNode> {
        // Prevent infinite recursion
        const MAX_DEPTH: usize = 50;
        if depth > MAX_DEPTH {
            return Err(ReedError::UcgIntegrity {
                reason: format!("Max depth {} exceeded", MAX_DEPTH),
            });
        }
        
        // Get children
        let children_entities = self.entity_manager.get_children(&entity.id).await?;
        let mut children = Vec::new();
        
        for child_entity in children_entities {
            let child_node = Box::new(
                self.build_node_recursive(&child_entity, depth + 1).await?
            );
            children.push(child_node);
        }
        
        // Get associations for this node
        let associations = self.association_manager
            .get_children_associations(&entity.id).await?;
        
        Ok(GraphNode {
            entity: entity.clone(),
            children,
            associations,
            depth,
        })
    }
}
```

## Graph Structure

```rust
/// Complete UCG graph representation
#[derive(Debug, Clone)]
pub struct UcgGraph {
    roots: Vec<GraphNode>,
    node_index: HashMap<Uuid, NodeReference>,
    path_index: HashMap<String, Uuid>,
    semantic_index: HashMap<String, Uuid>,
}

/// Node in the graph
#[derive(Debug, Clone)]
pub struct GraphNode {
    pub entity: Entity,
    pub children: Vec<Box<GraphNode>>,
    pub associations: Vec<Association>,
    pub depth: usize,
}

/// Reference to node location in graph
#[derive(Debug, Clone)]
pub struct NodeReference {
    pub path: Vec<usize>,  // Path of indices to reach node
    pub depth: usize,
}

impl UcgGraph {
    pub fn new() -> Self {
        Self {
            roots: Vec::new(),
            node_index: HashMap::new(),
            path_index: HashMap::new(),
            semantic_index: HashMap::new(),
        }
    }
    
    /// Add root node
    pub fn add_root(&mut self, node: GraphNode) {
        self.roots.push(node);
    }
    
    /// Build indices for fast lookups
    pub fn build_indices(&mut self) {
        self.node_index.clear();
        self.path_index.clear();
        self.semantic_index.clear();
        
        for (idx, root) in self.roots.iter().enumerate() {
            self.index_node_recursive(root, vec![idx], "content");
        }
    }
    
    /// Index node recursively
    fn index_node_recursive(&mut self, node: &GraphNode, path: Vec<usize>, ucg_path: &str) {
        // Index by entity ID
        self.node_index.insert(node.entity.id, NodeReference {
            path: path.clone(),
            depth: node.depth,
        });
        
        // Index by UCG path
        self.path_index.insert(ucg_path.to_string(), node.entity.id);
        
        // Index by semantic name
        if let Some(ref name) = node.entity.semantic_name {
            self.semantic_index.insert(name.as_str().to_string(), node.entity.id);
        }
        
        // Index children
        for (idx, child) in node.children.iter().enumerate() {
            let mut child_path = path.clone();
            child_path.push(idx);
            
            let child_ucg_path = format!("{}.{}", ucg_path, idx + 1);
            self.index_node_recursive(child, child_path, &child_ucg_path);
        }
    }
    
    /// Get total node count
    pub fn node_count(&self) -> usize {
        self.node_index.len()
    }
}
```

## Graph Operations

```rust
impl UcgGraph {
    /// Find node by ID
    pub fn find_node(&self, id: &Uuid) -> Option<&GraphNode> {
        let reference = self.node_index.get(id)?;
        self.navigate_to_node(&reference.path)
    }
    
    /// Find node by path
    pub fn find_by_path(&self, path: &str) -> Option<&GraphNode> {
        let id = self.path_index.get(path)?;
        self.find_node(id)
    }
    
    /// Find node by semantic name
    pub fn find_by_semantic(&self, name: &str) -> Option<&GraphNode> {
        let id = self.semantic_index.get(name)?;
        self.find_node(id)
    }
    
    /// Navigate to node using path indices
    fn navigate_to_node(&self, path: &[usize]) -> Option<&GraphNode> {
        if path.is_empty() {
            return None;
        }
        
        let mut current = self.roots.get(path[0])?;
        
        for &idx in &path[1..] {
            current = current.children.get(idx)?.as_ref();
        }
        
        Some(current)
    }
    
    /// Get all descendants of a node
    pub fn get_descendants(&self, id: &Uuid) -> Vec<&Entity> {
        let mut descendants = Vec::new();
        
        if let Some(node) = self.find_node(id) {
            self.collect_descendants_recursive(node, &mut descendants);
        }
        
        descendants
    }
    
    fn collect_descendants_recursive<'a>(
        &self,
        node: &'a GraphNode,
        descendants: &mut Vec<&'a Entity>,
    ) {
        for child in &node.children {
            descendants.push(&child.entity);
            self.collect_descendants_recursive(child, descendants);
        }
    }
}
```

## Graph Traversal

```rust
/// Graph traversal utilities
pub struct GraphTraversal<'a> {
    graph: &'a UcgGraph,
}

impl<'a> GraphTraversal<'a> {
    pub fn new(graph: &'a UcgGraph) -> Self {
        Self { graph }
    }
    
    /// Breadth-first traversal
    pub fn bfs<F>(&self, start_id: &Uuid, mut visitor: F) -> Result<()>
    where
        F: FnMut(&GraphNode) -> bool,
    {
        let start_node = self.graph.find_node(start_id)
            .ok_or_else(|| ReedError::EntityNotFound { id: *start_id })?;
        
        let mut queue = VecDeque::new();
        queue.push_back(start_node);
        
        while let Some(node) = queue.pop_front() {
            if !visitor(node) {
                break;  // Visitor returned false, stop traversal
            }
            
            for child in &node.children {
                queue.push_back(child);
            }
        }
        
        Ok(())
    }
    
    /// Depth-first traversal
    pub fn dfs<F>(&self, start_id: &Uuid, mut visitor: F) -> Result<()>
    where
        F: FnMut(&GraphNode) -> bool,
    {
        let start_node = self.graph.find_node(start_id)
            .ok_or_else(|| ReedError::EntityNotFound { id: *start_id })?;
        
        self.dfs_recursive(start_node, &mut visitor);
        Ok(())
    }
    
    fn dfs_recursive<F>(&self, node: &GraphNode, visitor: &mut F) -> bool
    where
        F: FnMut(&GraphNode) -> bool,
    {
        if !visitor(node) {
            return false;
        }
        
        for child in &node.children {
            if !self.dfs_recursive(child, visitor) {
                return false;
            }
        }
        
        true
    }
    
    /// Find path between two nodes
    pub fn find_path(&self, from_id: &Uuid, to_id: &Uuid) -> Option<Vec<Uuid>> {
        let mut path = Vec::new();
        let mut visited = HashSet::new();
        
        if self.find_path_recursive(from_id, to_id, &mut path, &mut visited) {
            Some(path)
        } else {
            None
        }
    }
    
    fn find_path_recursive(
        &self,
        current_id: &Uuid,
        target_id: &Uuid,
        path: &mut Vec<Uuid>,
        visited: &mut HashSet<Uuid>,
    ) -> bool {
        if current_id == target_id {
            path.push(*current_id);
            return true;
        }
        
        if !visited.insert(*current_id) {
            return false;  // Already visited
        }
        
        path.push(*current_id);
        
        if let Some(node) = self.graph.find_node(current_id) {
            for child in &node.children {
                if self.find_path_recursive(&child.entity.id, target_id, path, visited) {
                    return true;
                }
            }
        }
        
        path.pop();  // Backtrack
        false
    }
}
```

## Graph Modifications

```rust
impl UcgGraphBuilder {
    /// Add node to existing graph
    pub async fn add_node(
        &self,
        parent_id: &Uuid,
        entity: Entity,
        weight: Option<i32>,
    ) -> Result<()> {
        // Create association
        let association = self.association_manager.create_association(
            *parent_id,
            entity.id,
            AssociationType::Contains,
            weight,
        ).await?;
        
        // Store entity
        self.entity_manager.create_entity(
            entity.entity_type,
            entity.semantic_name.as_ref().map(|n| n.as_str()),
            entity.data,
            &entity.created_by,
        ).await?;
        
        tracing::info!(
            "Added node {} to parent {}",
            entity.id,
            parent_id
        );
        
        Ok(())
    }
    
    /// Remove node and all descendants
    pub async fn remove_node(&self, node_id: &Uuid) -> Result<()> {
        // Get all descendants first
        let graph = self.build_graph().await?;
        let descendants = graph.get_descendants(node_id);
        
        // Remove in reverse order (children first)
        for entity in descendants.iter().rev() {
            self.entity_manager.delete_entity(&entity.id, "system").await?;
        }
        
        // Remove the node itself
        self.entity_manager.delete_entity(node_id, "system").await?;
        
        tracing::info!(
            "Removed node {} and {} descendants",
            node_id,
            descendants.len()
        );
        
        Ok(())
    }
    
    /// Move subtree to new parent
    pub async fn move_subtree(
        &self,
        node_id: &Uuid,
        new_parent_id: &Uuid,
        new_weight: Option<i32>,
    ) -> Result<()> {
        // Check for cycles
        let graph = self.build_graph().await?;
        let traversal = GraphTraversal::new(&graph);
        
        if traversal.find_path(node_id, new_parent_id).is_some() {
            return Err(ReedError::UcgIntegrity {
                reason: "Move would create cycle".to_string(),
            });
        }
        
        // Move the association
        self.association_manager.move_association(
            node_id,
            new_parent_id,
            new_weight,
        ).await?;
        
        tracing::info!(
            "Moved subtree {} to parent {}",
            node_id,
            new_parent_id
        );
        
        Ok(())
    }
}
```

## Graph Serialization

```rust
impl UcgGraph {
    /// Export graph to JSON
    pub fn to_json(&self) -> Result<serde_json::Value> {
        let nodes: Vec<serde_json::Value> = self.roots.iter()
            .map(|node| self.node_to_json(node))
            .collect();
        
        Ok(serde_json::json!({
            "version": "1.0",
            "nodes": nodes,
            "statistics": {
                "total_nodes": self.node_count(),
                "max_depth": self.max_depth(),
            }
        }))
    }
    
    fn node_to_json(&self, node: &GraphNode) -> serde_json::Value {
        serde_json::json!({
            "id": node.entity.id,
            "type": node.entity.entity_type,
            "semantic_name": node.entity.semantic_name,
            "data": node.entity.data,
            "children": node.children.iter()
                .map(|child| self.node_to_json(child))
                .collect::<Vec<_>>(),
        })
    }
    
    /// Calculate maximum depth
    fn max_depth(&self) -> usize {
        self.node_index.values()
            .map(|ref_| ref_.depth)
            .max()
            .unwrap_or(0)
    }
}
```

## Summary

Dieser Graph Builder bietet:
- **Efficient Graph Construction** - Rekursiver Aufbau mit Cycle Detection
- **Fast Lookups** - Indizierung nach ID, Path und Semantic Name
- **Graph Traversal** - BFS und DFS Algorithmen
- **Graph Modifications** - Add, Remove, Move Operations
- **Serialization** - Export für Debugging und Backup

Der UCG Graph ist das Herzstück für alle Content-Operationen.
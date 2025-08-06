# Snippet CLI Commands

## Overview

Snippet-spezifische CLI Commands für ReedCMS. CRUD Operations, Bulk Actions und erweiterte Snippet-Verwaltung über die Command Line.

## Snippet Command Implementation

```rust
use clap::{Args, Subcommand};
use uuid::Uuid;
use std::collections::HashMap;

/// Snippet command implementation
pub struct SnippetCommand {
    manager: Arc<SnippetManager>,
    formatter: Arc<OutputFormatter>,
    context: Arc<CliContext>,
}

impl SnippetCommand {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self {
            manager: context.snippet_manager.clone(),
            formatter: Arc::new(OutputFormatter::new(context.output_format)),
            context,
        }
    }
    
    /// Execute snippet subcommand
    pub async fn execute(&self, args: &SnippetArgs) -> Result<CommandOutput> {
        match &args.command {
            SnippetCommands::List(list_args) => self.handle_list(list_args).await,
            SnippetCommands::Get(get_args) => self.handle_get(get_args).await,
            SnippetCommands::Create(create_args) => self.handle_create(create_args).await,
            SnippetCommands::Update(update_args) => self.handle_update(update_args).await,
            SnippetCommands::Delete(delete_args) => self.handle_delete(delete_args).await,
            SnippetCommands::Search(search_args) => self.handle_search(search_args).await,
            SnippetCommands::Move(move_args) => self.handle_move(move_args).await,
            SnippetCommands::Copy(copy_args) => self.handle_copy(copy_args).await,
            SnippetCommands::Bulk(bulk_args) => self.handle_bulk(bulk_args).await,
            SnippetCommands::Export(export_args) => self.handle_export(export_args).await,
            SnippetCommands::Import(import_args) => self.handle_import(import_args).await,
        }
    }
}

/// Snippet command arguments
#[derive(Args)]
pub struct SnippetArgs {
    #[command(subcommand)]
    pub command: SnippetCommands,
}

#[derive(Subcommand)]
pub enum SnippetCommands {
    /// List snippets
    List(ListArgs),
    
    /// Get snippet details
    Get(GetArgs),
    
    /// Create new snippet
    Create(CreateArgs),
    
    /// Update existing snippet
    Update(UpdateArgs),
    
    /// Delete snippet
    Delete(DeleteArgs),
    
    /// Search snippets
    Search(SearchArgs),
    
    /// Move snippet to new parent
    Move(MoveArgs),
    
    /// Copy snippet
    Copy(CopyArgs),
    
    /// Bulk operations
    Bulk(BulkArgs),
    
    /// Export snippets
    Export(ExportArgs),
    
    /// Import snippets
    Import(ImportArgs),
}
```

## List Command

```rust
#[derive(Args)]
pub struct ListArgs {
    /// Filter by snippet type
    #[arg(short = 't', long)]
    pub r#type: Option<String>,
    
    /// Filter by parent ID
    #[arg(short, long)]
    pub parent: Option<Uuid>,
    
    /// Include children recursively
    #[arg(short, long)]
    pub recursive: bool,
    
    /// Maximum depth for recursive listing
    #[arg(long, default_value = "3")]
    pub max_depth: u32,
    
    /// Sort by field
    #[arg(short, long, default_value = "created_at")]
    pub sort: String,
    
    /// Sort in descending order
    #[arg(long)]
    pub desc: bool,
    
    /// Limit number of results
    #[arg(short, long, default_value = "50")]
    pub limit: usize,
    
    /// Skip first N results
    #[arg(long, default_value = "0")]
    pub offset: usize,
    
    /// Show all fields
    #[arg(short, long)]
    pub all: bool,
    
    /// Fields to display
    #[arg(short = 'f', long, value_delimiter = ',')]
    pub fields: Vec<String>,
}

impl SnippetCommand {
    async fn handle_list(&self, args: &ListArgs) -> Result<CommandOutput> {
        // Build query
        let mut query = SnippetQueryBuilder::new();
        
        if let Some(ref snippet_type) = args.r#type {
            query = query.with_type(snippet_type);
        }
        
        if let Some(parent_id) = args.parent {
            if args.recursive {
                let descendants = self.get_descendants_recursive(
                    parent_id,
                    args.max_depth,
                ).await?;
                return Ok(self.format_tree_output(descendants));
            } else {
                query = query.with_parent(parent_id);
            }
        }
        
        query = query
            .order_by(&args.sort, args.desc)
            .limit(args.limit)
            .offset(args.offset);
        
        // Execute query
        let snippets = self.manager.execute_query(query).await?;
        
        // Format output
        let output = if args.all || !args.fields.is_empty() {
            self.format_detailed_list(&snippets, &args.fields)
        } else {
            self.format_snippet_list(&snippets)
        };
        
        Ok(CommandOutput::Success(output))
    }
    
    /// Get descendants recursively
    async fn get_descendants_recursive(
        &self,
        parent_id: Uuid,
        max_depth: u32,
    ) -> Result<TreeNode<Entity>> {
        self.build_tree_recursive(parent_id, 0, max_depth).await
    }
    
    /// Build tree structure
    async fn build_tree_recursive(
        &self,
        parent_id: Uuid,
        depth: u32,
        max_depth: u32,
    ) -> Result<TreeNode<Entity>> {
        if depth >= max_depth {
            return Ok(TreeNode {
                data: None,
                children: vec![],
            });
        }
        
        let parent = self.manager.get_by_id(parent_id).await?
            .ok_or_else(|| ReedError::NotFound(format!("Snippet {}", parent_id)))?;
        
        let children = self.manager.get_children(parent_id, 100).await?;
        
        let child_nodes = futures::future::try_join_all(
            children.into_iter().map(|child| {
                Box::pin(self.build_tree_recursive(child.id, depth + 1, max_depth))
            })
        ).await?;
        
        Ok(TreeNode {
            data: Some(parent),
            children: child_nodes,
        })
    }
}

#[derive(Debug)]
struct TreeNode<T> {
    data: Option<T>,
    children: Vec<TreeNode<T>>,
}
```

## Create Command

```rust
#[derive(Args)]
pub struct CreateArgs {
    /// Snippet type
    pub r#type: String,
    
    /// Parent snippet ID
    #[arg(short, long)]
    pub parent: Option<Uuid>,
    
    /// Data fields as key=value pairs
    #[arg(value_parser = parse_data_arg)]
    pub data: Vec<DataArg>,
    
    /// Read data from file
    #[arg(short = 'f', long, conflicts_with = "stdin")]
    pub file: Option<PathBuf>,
    
    /// Read data from stdin
    #[arg(long, conflicts_with = "file")]
    pub stdin: bool,
    
    /// Output only the ID
    #[arg(long)]
    pub quiet: bool,
    
    /// Validate only, don't create
    #[arg(long)]
    pub dry_run: bool,
}

impl SnippetCommand {
    async fn handle_create(&self, args: &CreateArgs) -> Result<CommandOutput> {
        // Prepare data
        let mut data = if args.stdin {
            self.read_stdin_data().await?
        } else if let Some(ref file_path) = args.file {
            self.read_file_data(file_path).await?
        } else {
            merge_data_args(args.data.clone())
        };
        
        // Validate snippet type exists
        let definition = self.context.registry_manager
            .get_definition(&args.r#type)
            .await?;
        
        // Apply defaults from definition
        for field in &definition.fields {
            if !data.contains_key(&field.name) {
                if let Some(ref default) = field.default_value {
                    data.insert(field.name.clone(), default.clone());
                }
            }
        }
        
        // Convert to HashMap for manager
        let data_map = data.as_object()
            .ok_or_else(|| ReedError::InvalidData("Data must be an object".into()))?
            .clone();
        
        // Dry run validation
        if args.dry_run {
            let validation_result = self.context.validator
                .validate_snippet(&args.r#type, &data_map)
                .await?;
            
            if validation_result.is_valid {
                return Ok(CommandOutput::Success(OutputData::Text(
                    "Validation passed".to_string()
                )));
            } else {
                return Ok(CommandOutput::Error(
                    format!("Validation failed: {:?}", validation_result.errors)
                ));
            }
        }
        
        // Create snippet
        let user_context = self.context.get_user_context()?;
        let entity = self.manager
            .create_snippet(
                &args.r#type,
                data_map,
                args.parent,
                &user_context,
            )
            .await?;
        
        // Format output
        if args.quiet {
            Ok(CommandOutput::Success(OutputData::Text(entity.id.to_string())))
        } else {
            Ok(CommandOutput::Success(OutputData::Json(
                serde_json::to_value(&entity)?
            )))
        }
    }
    
    /// Read data from stdin
    async fn read_stdin_data(&self) -> Result<serde_json::Value> {
        use tokio::io::{self, AsyncReadExt};
        
        let mut buffer = String::new();
        io::stdin().read_to_string(&mut buffer).await?;
        
        // Try to parse as JSON
        if let Ok(value) = serde_json::from_str(&buffer) {
            Ok(value)
        } else {
            // Try to parse as key=value pairs
            let data_args: Result<Vec<DataArg>> = buffer
                .lines()
                .filter(|line| !line.trim().is_empty())
                .map(parse_data_arg)
                .collect();
            
            Ok(merge_data_args(data_args?))
        }
    }
}
```

## Update Command

```rust
#[derive(Args)]
pub struct UpdateArgs {
    /// Snippet ID
    pub id: Uuid,
    
    /// Updates as key=value pairs
    #[arg(value_parser = parse_data_arg)]
    pub updates: Vec<DataArg>,
    
    /// Replace all data (not merge)
    #[arg(long)]
    pub replace: bool,
    
    /// Unset fields (remove them)
    #[arg(long, value_delimiter = ',')]
    pub unset: Vec<String>,
    
    /// Show diff before applying
    #[arg(long)]
    pub diff: bool,
    
    /// Force update without confirmation
    #[arg(short, long)]
    pub force: bool,
}

impl SnippetCommand {
    async fn handle_update(&self, args: &UpdateArgs) -> Result<CommandOutput> {
        // Get existing snippet
        let existing = self.manager.get_by_id(args.id).await?
            .ok_or_else(|| ReedError::NotFound(format!("Snippet {}", args.id)))?;
        
        // Prepare updates
        let mut updates = if args.replace {
            merge_data_args(args.updates.clone())
        } else {
            // Merge with existing
            let mut current_data = existing.data.clone();
            let update_data = merge_data_args(args.updates.clone());
            
            if let (Some(current_obj), Some(update_obj)) = 
                (current_data.as_object_mut(), update_data.as_object()) {
                for (key, value) in update_obj {
                    current_obj.insert(key.clone(), value.clone());
                }
            }
            
            current_data
        };
        
        // Handle unset fields
        if let Some(obj) = updates.as_object_mut() {
            for field in &args.unset {
                obj.remove(field);
            }
        }
        
        // Show diff if requested
        if args.diff {
            self.show_update_diff(&existing.data, &updates)?;
            
            if !args.force {
                let proceed = Prompts::confirm("Apply these changes?", true)?;
                if !proceed {
                    return Ok(CommandOutput::Success(OutputData::Text(
                        "Update cancelled".to_string()
                    )));
                }
            }
        }
        
        // Convert to HashMap
        let updates_map = updates.as_object()
            .ok_or_else(|| ReedError::InvalidData("Updates must be an object".into()))?
            .clone();
        
        // Update snippet
        let user_context = self.context.get_user_context()?;
        let updated = self.manager
            .update_snippet(args.id, updates_map, &user_context)
            .await?;
        
        Ok(CommandOutput::Success(OutputData::Json(
            serde_json::to_value(&updated)?
        )))
    }
    
    /// Show diff between current and updated data
    fn show_update_diff(
        &self,
        current: &serde_json::Value,
        updated: &serde_json::Value,
    ) -> Result<()> {
        let current_str = serde_json::to_string_pretty(current)?;
        let updated_str = serde_json::to_string_pretty(updated)?;
        
        println!("Changes to be applied:");
        DiffDisplay::unified_diff(&current_str, &updated_str, 3);
        println!();
        
        Ok(())
    }
}
```

## Search Command

```rust
#[derive(Args)]
pub struct SearchArgs {
    /// Search query
    pub query: String,
    
    /// Filter by snippet type
    #[arg(short = 't', long)]
    pub r#type: Option<String>,
    
    /// Search in specific fields
    #[arg(short = 'f', long, value_delimiter = ',')]
    pub fields: Vec<String>,
    
    /// Use regex search
    #[arg(long)]
    pub regex: bool,
    
    /// Case sensitive search
    #[arg(long)]
    pub case_sensitive: bool,
    
    /// Limit results
    #[arg(short, long, default_value = "20")]
    pub limit: usize,
    
    /// Highlight matches
    #[arg(long)]
    pub highlight: bool,
}

impl SnippetCommand {
    async fn handle_search(&self, args: &SearchArgs) -> Result<CommandOutput> {
        let results = if args.regex {
            self.regex_search(args).await?
        } else {
            self.fulltext_search(args).await?
        };
        
        // Format results
        let output = if args.highlight {
            self.format_search_results_highlighted(&results, &args.query)
        } else {
            self.format_search_results(&results)
        };
        
        Ok(CommandOutput::Success(output))
    }
    
    /// Full-text search
    async fn fulltext_search(&self, args: &SearchArgs) -> Result<Vec<SearchResult>> {
        let snippets = self.manager
            .search(&args.query, args.r#type.as_deref(), args.limit)
            .await?;
        
        // Convert to search results
        let results = snippets.into_iter()
            .map(|entity| {
                let matches = self.find_matches(&entity, &args.query, &args.fields);
                SearchResult {
                    entity,
                    matches,
                    score: 1.0, // Default score
                }
            })
            .collect();
        
        Ok(results)
    }
    
    /// Find matching fields
    fn find_matches(
        &self,
        entity: &Entity,
        query: &str,
        fields: &[String],
    ) -> Vec<MatchContext> {
        let mut matches = Vec::new();
        let query_lower = query.to_lowercase();
        
        if let Some(obj) = entity.data.as_object() {
            for (key, value) in obj {
                // Skip if specific fields requested and not in list
                if !fields.is_empty() && !fields.contains(key) {
                    continue;
                }
                
                if let Some(text) = value.as_str() {
                    if text.to_lowercase().contains(&query_lower) {
                        // Find match position
                        if let Some(pos) = text.to_lowercase().find(&query_lower) {
                            let start = pos.saturating_sub(30);
                            let end = (pos + query.len() + 30).min(text.len());
                            
                            matches.push(MatchContext {
                                field: key.clone(),
                                context: text[start..end].to_string(),
                                position: pos - start,
                            });
                        }
                    }
                }
            }
        }
        
        matches
    }
}

#[derive(Debug)]
struct SearchResult {
    entity: Entity,
    matches: Vec<MatchContext>,
    score: f64,
}

#[derive(Debug)]
struct MatchContext {
    field: String,
    context: String,
    position: usize,
}
```

## Bulk Operations

```rust
#[derive(Args)]
pub struct BulkArgs {
    #[command(subcommand)]
    pub operation: BulkOperation,
}

#[derive(Subcommand)]
pub enum BulkOperation {
    /// Update multiple snippets
    Update {
        /// Snippet IDs (or read from stdin)
        #[arg(value_delimiter = ',')]
        ids: Vec<Uuid>,
        
        /// Updates to apply
        #[arg(value_parser = parse_data_arg)]
        updates: Vec<DataArg>,
        
        /// Read IDs from stdin
        #[arg(long)]
        stdin: bool,
    },
    
    /// Delete multiple snippets
    Delete {
        /// Snippet IDs
        #[arg(value_delimiter = ',')]
        ids: Vec<Uuid>,
        
        /// Force without confirmation
        #[arg(short, long)]
        force: bool,
    },
    
    /// Move multiple snippets
    Move {
        /// Snippet IDs
        #[arg(value_delimiter = ',')]
        ids: Vec<Uuid>,
        
        /// New parent ID
        parent: Uuid,
    },
}

impl SnippetCommand {
    async fn handle_bulk(&self, args: &BulkArgs) -> Result<CommandOutput> {
        match &args.operation {
            BulkOperation::Update { ids, updates, stdin } => {
                let ids = if *stdin {
                    self.read_ids_from_stdin().await?
                } else {
                    ids.clone()
                };
                
                self.bulk_update(ids, updates.clone()).await
            }
            BulkOperation::Delete { ids, force } => {
                self.bulk_delete(ids.clone(), *force).await
            }
            BulkOperation::Move { ids, parent } => {
                self.bulk_move(ids.clone(), *parent).await
            }
        }
    }
    
    /// Bulk update snippets
    async fn bulk_update(
        &self,
        ids: Vec<Uuid>,
        updates: Vec<DataArg>,
    ) -> Result<CommandOutput> {
        let update_data = merge_data_args(updates);
        let update_map = update_data.as_object()
            .ok_or_else(|| ReedError::InvalidData("Updates must be an object".into()))?
            .clone();
        
        let user_context = self.context.get_user_context()?;
        let mut results = Vec::new();
        let mut errors = Vec::new();
        
        // Progress bar
        let pb = ProgressBar::new(ids.len() as u64);
        pb.set_style(ProgressStyle::default_bar()
            .template("{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} {msg}")
            .unwrap());
        
        for id in ids {
            pb.set_message(format!("Updating {}", id));
            
            match self.manager.update_snippet(id, update_map.clone(), &user_context).await {
                Ok(entity) => results.push(entity),
                Err(e) => errors.push((id, e)),
            }
            
            pb.inc(1);
        }
        
        pb.finish_with_message("Done");
        
        // Format results
        let summary = BulkUpdateSummary {
            total: results.len() + errors.len(),
            updated: results.len(),
            failed: errors.len(),
            errors: errors.into_iter()
                .map(|(id, e)| format!("{}: {}", id, e))
                .collect(),
        };
        
        Ok(CommandOutput::Success(OutputData::Json(
            serde_json::to_value(&summary)?
        )))
    }
}

#[derive(Debug, Serialize)]
struct BulkUpdateSummary {
    total: usize,
    updated: usize,
    failed: usize,
    errors: Vec<String>,
}
```

## Export/Import Commands

```rust
#[derive(Args)]
pub struct ExportArgs {
    /// Export format
    #[arg(short, long, value_enum, default_value = "json")]
    pub format: ExportFormat,
    
    /// Output file (or stdout)
    #[arg(short, long)]
    pub output: Option<PathBuf>,
    
    /// Filter by type
    #[arg(short = 't', long)]
    pub r#type: Option<String>,
    
    /// Include children recursively
    #[arg(short, long)]
    pub recursive: bool,
    
    /// Pretty print output
    #[arg(long)]
    pub pretty: bool,
}

#[derive(Debug, Clone, Copy, ValueEnum)]
pub enum ExportFormat {
    Json,
    Yaml,
    Csv,
}

impl SnippetCommand {
    async fn handle_export(&self, args: &ExportArgs) -> Result<CommandOutput> {
        // Get snippets to export
        let snippets = if let Some(ref snippet_type) = args.r#type {
            self.manager.get_by_type(snippet_type, 10000, 0).await?
        } else {
            self.manager.list_all().await?
        };
        
        // Export based on format
        let output = match args.format {
            ExportFormat::Json => self.export_json(&snippets, args.pretty)?,
            ExportFormat::Yaml => self.export_yaml(&snippets)?,
            ExportFormat::Csv => self.export_csv(&snippets)?,
        };
        
        // Write to file or stdout
        if let Some(ref path) = args.output {
            tokio::fs::write(path, output).await?;
            Ok(CommandOutput::Success(OutputData::Text(
                format!("Exported {} snippets to {}", snippets.len(), path.display())
            )))
        } else {
            Ok(CommandOutput::Success(OutputData::Text(output)))
        }
    }
}
```

## Summary

Diese Snippet Commands bieten:
- **Full CRUD** - Create, Read, Update, Delete Operations
- **Advanced Search** - Full-text und Regex Search
- **Bulk Operations** - Batch Updates und Deletes
- **Tree Navigation** - Recursive Listing mit Hierarchie
- **Import/Export** - Data Migration Support

Snippet Commands machen Content Management über CLI effizient.
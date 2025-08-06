# Whois Debug Command

## Overview

Whois Debug Command für ReedCMS CLI. Deep Inspection von Entities, Associations und System State für Debugging.

## Whois Command Implementation

```rust
use clap::{Args, Subcommand};
use uuid::Uuid;
use colored::*;

/// Whois debug command implementation
pub struct WhoisCommand {
    ucg_query: Arc<UcgQuery>,
    snippet_manager: Arc<SnippetManager>,
    theme_manager: Arc<ThemeManager>,
    cache: Arc<SnippetCache>,
    context: Arc<CliContext>,
}

impl WhoisCommand {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self {
            ucg_query: context.ucg_query.clone(),
            snippet_manager: context.snippet_manager.clone(),
            theme_manager: context.theme_manager.clone(),
            cache: context.cache.clone(),
            context,
        }
    }
    
    /// Execute whois subcommand
    pub async fn execute(&self, args: &WhoisArgs) -> Result<CommandOutput> {
        match &args.target {
            WhoisTarget::Entity { id } => self.whois_entity(*id).await,
            WhoisTarget::Name { name } => self.whois_by_name(name).await,
            WhoisTarget::Path { path } => self.whois_by_path(path).await,
            WhoisTarget::File { file } => self.whois_file(file).await,
            WhoisTarget::Cache { key } => self.whois_cache(key).await,
            WhoisTarget::Memory => self.whois_memory().await,
            WhoisTarget::Stats => self.whois_stats().await,
        }
    }
}

/// Whois command arguments
#[derive(Args)]
pub struct WhoisArgs {
    /// Target to inspect
    #[command(subcommand)]
    pub target: WhoisTarget,
}

#[derive(Subcommand)]
pub enum WhoisTarget {
    /// Inspect entity by ID
    Entity {
        /// Entity UUID
        id: Uuid,
    },
    
    /// Inspect by semantic name
    Name {
        /// Semantic name
        name: String,
    },
    
    /// Inspect by UCG path
    Path {
        /// UCG path (e.g., content.1.2.3)
        path: String,
    },
    
    /// Inspect file resolution
    File {
        /// File path to trace
        file: String,
    },
    
    /// Inspect cache entry
    Cache {
        /// Cache key
        key: String,
    },
    
    /// Inspect memory usage
    Memory,
    
    /// Show system statistics
    Stats,
}
```

## Entity Inspection

```rust
impl WhoisCommand {
    /// Inspect entity by ID
    async fn whois_entity(&self, id: Uuid) -> Result<CommandOutput> {
        let mut output = String::new();
        
        // Header
        output.push_str(&format!("{}\n", "ENTITY INSPECTION".bold().cyan()));
        output.push_str(&format!("{}\n\n", "=".repeat(50)));
        
        // Get entity
        let entity = self.ucg_query.get_by_id(id).await?
            .ok_or_else(|| ReedError::NotFound(format!("Entity {}", id)))?;
        
        // Basic info
        output.push_str(&format!("{}\n", "Basic Information:".bold()));
        output.push_str(&format!("  ID:           {}\n", entity.id.to_string().yellow()));
        output.push_str(&format!("  Type:         {}\n", format!("{:?}", entity.entity_type).green()));
        
        if let Some(ref name) = entity.semantic_name {
            output.push_str(&format!("  Name:         {}\n", name.as_str().blue()));
        }
        
        output.push_str(&format!("  Created:      {}\n", entity.created_at.format("%Y-%m-%d %H:%M:%S")));
        output.push_str(&format!("  Updated:      {}\n", entity.updated_at.format("%Y-%m-%d %H:%M:%S")));
        output.push_str(&format!("  Created By:   {}\n", entity.created_by));
        output.push_str("\n");
        
        // Data inspection
        output.push_str(&format!("{}\n", "Data:".bold()));
        if let Some(snippet_type) = entity.data.get("snippet_type").and_then(|v| v.as_str()) {
            output.push_str(&format!("  Snippet Type: {}\n", snippet_type.magenta()));
        }
        
        // Format JSON data
        let formatted_data = serde_json::to_string_pretty(&entity.data)?;
        for line in formatted_data.lines() {
            output.push_str(&format!("  {}\n", line));
        }
        output.push_str("\n");
        
        // Associations
        output.push_str(&format!("{}\n", "Associations:".bold()));
        
        // Parent
        if let Some(parent) = self.ucg_query.get_parent(id).await? {
            output.push_str(&format!("  Parent:       {} ({})\n", 
                parent.parent_id.to_string().yellow(),
                parent.association_type.to_string().dim()
            ));
        } else {
            output.push_str(&format!("  Parent:       {}\n", "None".dim()));
        }
        
        // Children
        let children = self.ucg_query.get_children(id, None).await?;
        output.push_str(&format!("  Children:     {} total\n", children.len()));
        
        for (i, child) in children.iter().take(5).enumerate() {
            let child_type = child.data.get("snippet_type")
                .and_then(|v| v.as_str())
                .unwrap_or("unknown");
            
            output.push_str(&format!("    {}. {} ({})\n",
                i + 1,
                child.id.to_string().yellow(),
                child_type.dim()
            ));
        }
        
        if children.len() > 5 {
            output.push_str(&format!("    ... and {} more\n", children.len() - 5));
        }
        output.push_str("\n");
        
        // Cache status
        output.push_str(&format!("{}\n", "Cache Status:".bold()));
        let cache_key = format!("snippet:{}", id);
        let is_cached = self.cache.exists(&cache_key).await?;
        
        if is_cached {
            let ttl = self.cache.ttl(&cache_key).await?;
            output.push_str(&format!("  Status:       {}\n", "Cached".green()));
            output.push_str(&format!("  TTL:          {} seconds\n", ttl));
        } else {
            output.push_str(&format!("  Status:       {}\n", "Not cached".red()));
        }
        
        // Access stats
        if let Some(stats) = self.cache.get_access_stats_for(id).await? {
            output.push_str(&format!("  Accesses:     {}\n", stats.total));
            output.push_str(&format!("  Hit Rate:     {:.1}%\n", stats.hit_rate() * 100.0));
            
            if let Some(last) = stats.last_access {
                output.push_str(&format!("  Last Access:  {}\n", last.format("%Y-%m-%d %H:%M:%S")));
            }
        }
        output.push_str("\n");
        
        // Path information
        output.push_str(&format!("{}\n", "Path Information:".bold()));
        let paths = self.ucg_query.get_paths_to(id).await?;
        
        if paths.is_empty() {
            output.push_str(&format!("  Root entity (no paths)\n"));
        } else {
            for (i, path) in paths.iter().take(3).enumerate() {
                output.push_str(&format!("  Path {}:      {}\n", i + 1, path.cyan()));
            }
            
            if paths.len() > 3 {
                output.push_str(&format!("  ... and {} more paths\n", paths.len() - 3));
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
    
    /// Inspect by semantic name
    async fn whois_by_name(&self, name: &str) -> Result<CommandOutput> {
        // Try to parse as semantic name
        let semantic_name = SemanticName::new(name)?;
        
        // Find entity
        let entity = self.ucg_query.get_by_semantic_name(name).await?
            .ok_or_else(|| ReedError::NotFound(format!("No entity with name '{}'", name)))?;
        
        // Delegate to entity inspection
        self.whois_entity(entity.id).await
    }
}
```

## File Resolution Debugging

```rust
impl WhoisCommand {
    /// Debug file resolution
    async fn whois_file(&self, file: &str) -> Result<CommandOutput> {
        let mut output = String::new();
        
        output.push_str(&format!("{}\n", "FILE RESOLUTION TRACE".bold().cyan()));
        output.push_str(&format!("{}\n\n", "=".repeat(50)));
        
        // Get current theme
        let theme = self.theme_manager.get_current_theme().await?;
        
        output.push_str(&format!("{}\n", "Current Context:".bold()));
        output.push_str(&format!("  Theme:        {}\n", theme.name.green()));
        output.push_str(&format!("  Context:      {:?}\n", theme.context));
        output.push_str("\n");
        
        // Build theme chain
        let chain = self.theme_manager.build_theme_chain(&theme).await?;
        
        output.push_str(&format!("{}\n", "Theme Chain:".bold()));
        for (i, theme_name) in chain.iter().enumerate() {
            output.push_str(&format!("  {}. {}\n", i + 1, theme_name));
        }
        output.push_str("\n");
        
        // Try to resolve file
        output.push_str(&format!("{}\n", "Resolution Steps:".bold()));
        
        let file_type = self.detect_file_type(file);
        let mut resolved = false;
        
        for (i, theme_name) in chain.iter().enumerate() {
            let theme_path = self.context.config.themes_path.join(theme_name);
            let file_path = theme_path.join(file_type.directory()).join(file);
            
            let exists = file_path.exists();
            let status = if exists {
                resolved = true;
                "✓ FOUND".green()
            } else {
                "✗ Not found".red()
            };
            
            output.push_str(&format!("\n{}. Theme: {}\n", i + 1, theme_name.yellow()));
            output.push_str(&format!("   Path:   {}\n", file_path.display()));
            output.push_str(&format!("   Status: {}\n", status));
            
            if exists {
                // Show file info
                if let Ok(metadata) = file_path.metadata() {
                    output.push_str(&format!("   Size:   {} bytes\n", metadata.len()));
                    
                    if let Ok(modified) = metadata.modified() {
                        if let Ok(duration) = modified.duration_since(std::time::UNIX_EPOCH) {
                            let datetime = chrono::DateTime::<chrono::Utc>::from(
                                std::time::UNIX_EPOCH + duration
                            );
                            output.push_str(&format!("   Modified: {}\n", 
                                datetime.format("%Y-%m-%d %H:%M:%S")));
                        }
                    }
                }
                
                // Show first few lines if template
                if file_type == FileType::Template {
                    if let Ok(content) = tokio::fs::read_to_string(&file_path).await {
                        output.push_str(&format!("\n   {}:\n", "Preview".dim()));
                        for line in content.lines().take(5) {
                            output.push_str(&format!("   {}\n", line.dim()));
                        }
                        if content.lines().count() > 5 {
                            output.push_str(&format!("   {}\n", "...".dim()));
                        }
                    }
                }
                
                break;
            }
        }
        
        if !resolved {
            output.push_str(&format!("\n{}\n", "❌ File not found in any theme".red().bold()));
        } else {
            output.push_str(&format!("\n{}\n", "✅ File resolved successfully".green().bold()));
        }
        
        // Show overrides if any
        let overrides = self.context.epc_resolver
            .get_overrides(file, file_type, &theme)
            .await?;
        
        if overrides.len() > 1 {
            output.push_str(&format!("\n{}\n", "File Overrides:".bold()));
            for (i, override_info) in overrides.iter().enumerate() {
                output.push_str(&format!("  {}. {} (priority: {})\n",
                    i + 1,
                    override_info.theme.yellow(),
                    override_info.priority
                ));
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
    
    /// Detect file type from path
    fn detect_file_type(&self, file: &str) -> FileType {
        if file.ends_with(".tera") {
            FileType::Template
        } else if file.ends_with(".css") {
            FileType::Style
        } else if file.ends_with(".js") {
            FileType::Script
        } else if file.ends_with(".png") || file.ends_with(".jpg") || 
                  file.ends_with(".svg") || file.ends_with(".gif") {
            FileType::Image
        } else {
            FileType::Asset
        }
    }
}
```

## Cache Inspection

```rust
impl WhoisCommand {
    /// Inspect cache entry
    async fn whois_cache(&self, key: &str) -> Result<CommandOutput> {
        let mut output = String::new();
        
        output.push_str(&format!("{}\n", "CACHE INSPECTION".bold().cyan()));
        output.push_str(&format!("{}\n\n", "=".repeat(50)));
        
        output.push_str(&format!("Key: {}\n\n", key.yellow()));
        
        // Check if exists
        let exists = self.context.redis.exists(key).await?;
        
        if !exists {
            output.push_str(&format!("{}\n", "❌ Key does not exist".red()));
            
            // Try to find similar keys
            let pattern = format!("{}*", key.split(':').next().unwrap_or(""));
            let similar_keys = self.context.redis.keys(&pattern).await?;
            
            if !similar_keys.is_empty() {
                output.push_str(&format!("\n{}\n", "Similar keys:".dim()));
                for similar_key in similar_keys.iter().take(10) {
                    output.push_str(&format!("  - {}\n", similar_key.dim()));
                }
            }
            
            return Ok(CommandOutput::Success(OutputData::Text(output)));
        }
        
        // Get key info
        output.push_str(&format!("{}\n", "Key Information:".bold()));
        
        // Type
        let key_type = self.context.redis.key_type(key).await?;
        output.push_str(&format!("  Type:         {}\n", key_type.green()));
        
        // TTL
        let ttl = self.context.redis.ttl(key).await?;
        if ttl > 0 {
            output.push_str(&format!("  TTL:          {} seconds\n", ttl));
            output.push_str(&format!("  Expires:      {}\n", 
                (chrono::Utc::now() + chrono::Duration::seconds(ttl as i64))
                    .format("%Y-%m-%d %H:%M:%S")
            ));
        } else {
            output.push_str(&format!("  TTL:          {}\n", "No expiry".yellow()));
        }
        
        // Memory usage
        let memory = self.context.redis.memory_usage(key).await?;
        output.push_str(&format!("  Memory:       {} bytes\n", memory));
        output.push_str("\n");
        
        // Value preview
        output.push_str(&format!("{}\n", "Value Preview:".bold()));
        
        match key_type.as_str() {
            "string" => {
                let value: String = self.context.redis.get(key).await?;
                
                // Try to parse as JSON
                if let Ok(json) = serde_json::from_str::<serde_json::Value>(&value) {
                    let pretty = serde_json::to_string_pretty(&json)?;
                    for line in pretty.lines().take(20) {
                        output.push_str(&format!("  {}\n", line));
                    }
                    if pretty.lines().count() > 20 {
                        output.push_str(&format!("  {}\n", "...".dim()));
                    }
                } else {
                    // Show raw value
                    for line in value.lines().take(10) {
                        output.push_str(&format!("  {}\n", line));
                    }
                    if value.lines().count() > 10 {
                        output.push_str(&format!("  {}\n", "...".dim()));
                    }
                }
            }
            "list" => {
                let length = self.context.redis.llen(key).await?;
                output.push_str(&format!("  Length: {}\n", length));
                
                let items: Vec<String> = self.context.redis.lrange(key, 0, 9).await?;
                for (i, item) in items.iter().enumerate() {
                    output.push_str(&format!("  [{}] {}\n", i, item));
                }
                if length > 10 {
                    output.push_str(&format!("  ... and {} more\n", length - 10));
                }
            }
            "set" => {
                let size = self.context.redis.scard(key).await?;
                output.push_str(&format!("  Size: {}\n", size));
                
                let members: Vec<String> = self.context.redis.smembers(key).await?;
                for member in members.iter().take(10) {
                    output.push_str(&format!("  - {}\n", member));
                }
                if size > 10 {
                    output.push_str(&format!("  ... and {} more\n", size - 10));
                }
            }
            "hash" => {
                let fields: Vec<String> = self.context.redis.hkeys(key).await?;
                output.push_str(&format!("  Fields: {}\n", fields.len()));
                
                for field in fields.iter().take(10) {
                    let value: String = self.context.redis.hget(key, field).await?;
                    output.push_str(&format!("  {} = {}\n", field.blue(), value));
                }
                if fields.len() > 10 {
                    output.push_str(&format!("  ... and {} more fields\n", fields.len() - 10));
                }
            }
            _ => {
                output.push_str(&format!("  Unsupported type for preview\n"));
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
}
```

## System Statistics

```rust
impl WhoisCommand {
    /// Show system statistics
    async fn whois_stats(&self) -> Result<CommandOutput> {
        let mut output = String::new();
        
        output.push_str(&format!("{}\n", "SYSTEM STATISTICS".bold().cyan()));
        output.push_str(&format!("{}\n\n", "=".repeat(50)));
        
        // Database stats
        output.push_str(&format!("{}\n", "Database:".bold()));
        
        let entity_count: (i64,) = sqlx::query_as("SELECT COUNT(*) FROM entities")
            .fetch_one(&*self.context.db.pool)
            .await?;
        
        let snippet_count: (i64,) = sqlx::query_as(
            "SELECT COUNT(*) FROM entities WHERE entity_type = 'snippet'"
        )
            .fetch_one(&*self.context.db.pool)
            .await?;
        
        output.push_str(&format!("  Total Entities:    {}\n", entity_count.0));
        output.push_str(&format!("  Snippets:          {}\n", snippet_count.0));
        
        // Snippet types breakdown
        let type_counts: Vec<(String, i64)> = sqlx::query_as(
            r#"
            SELECT data->>'snippet_type' as type, COUNT(*) as count
            FROM entities
            WHERE entity_type = 'snippet'
            GROUP BY data->>'snippet_type'
            ORDER BY count DESC
            LIMIT 5
            "#
        )
            .fetch_all(&*self.context.db.pool)
            .await?;
        
        output.push_str(&format!("\n  Top Snippet Types:\n"));
        for (snippet_type, count) in type_counts {
            output.push_str(&format!("    {:15} {}\n", 
                snippet_type.unwrap_or("(unknown)".to_string()),
                count
            ));
        }
        output.push_str("\n");
        
        // Redis stats
        output.push_str(&format!("{}\n", "Cache:".bold()));
        
        let info = self.context.redis.info().await?;
        
        // Parse Redis INFO
        for line in info.lines() {
            if line.starts_with("used_memory_human:") {
                let memory = line.split(':').nth(1).unwrap_or("unknown");
                output.push_str(&format!("  Memory Used:       {}\n", memory));
            } else if line.starts_with("connected_clients:") {
                let clients = line.split(':').nth(1).unwrap_or("0");
                output.push_str(&format!("  Connected Clients: {}\n", clients));
            } else if line.starts_with("total_commands_processed:") {
                let commands = line.split(':').nth(1).unwrap_or("0");
                output.push_str(&format!("  Total Commands:    {}\n", commands));
            }
        }
        
        // Key statistics
        let key_count = self.context.redis.dbsize().await?;
        output.push_str(&format!("  Total Keys:        {}\n", key_count));
        
        // Key type breakdown
        output.push_str(&format!("\n  Key Types:\n"));
        
        let patterns = vec![
            ("snippet:*", "Snippets"),
            ("ucg:entity:*", "UCG Entities"),
            ("ucg:assoc:*", "UCG Associations"),
            ("idx:*", "Indices"),
            ("stats:*", "Statistics"),
        ];
        
        for (pattern, label) in patterns {
            let keys: Vec<String> = self.context.redis.keys(pattern).await?;
            if !keys.is_empty() {
                output.push_str(&format!("    {:15} {}\n", label, keys.len()));
            }
        }
        output.push_str("\n");
        
        // Theme stats
        output.push_str(&format!("{}\n", "Themes:".bold()));
        
        let themes = self.theme_manager.list_themes().await?;
        let active_themes = themes.iter().filter(|t| t.active).count();
        
        output.push_str(&format!("  Total Themes:      {}\n", themes.len()));
        output.push_str(&format!("  Active Themes:     {}\n", active_themes));
        
        // Performance stats
        output.push_str(&format!("\n{}\n", "Performance:".bold()));
        
        // Get cache hit rate
        let cache_stats = self.cache.get_global_stats().await?;
        
        output.push_str(&format!("  Cache Hit Rate:    {:.1}%\n", 
            cache_stats.hit_rate * 100.0));
        output.push_str(&format!("  Avg Response Time: {:.2}ms\n",
            cache_stats.avg_response_time));
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
}
```

## Summary

Der Whois Command bietet:
- **Entity Inspection** - Deep Dive in Entity Details
- **Path Tracing** - UCG Path Resolution
- **File Debugging** - EPC File Resolution Trace
- **Cache Analysis** - Redis Key Inspection
- **System Stats** - Performance und Usage Metrics

Whois ist das ultimative Debug-Tool für ReedCMS.
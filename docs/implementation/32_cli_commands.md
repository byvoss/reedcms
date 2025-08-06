# CLI Command Registry

## Overview

Command Registry für ReedCMS CLI. Zentrale Verwaltung aller Commands mit Plugin Support und Dynamic Loading.

## Command Registry Core

```rust
use std::collections::HashMap;
use async_trait::async_trait;

/// Central registry for CLI commands
pub struct CommandRegistry {
    commands: HashMap<String, Box<dyn Command>>,
    aliases: HashMap<String, String>,
    context: Arc<CliContext>,
}

impl CommandRegistry {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self {
            commands: HashMap::new(),
            aliases: HashMap::new(),
            context,
        }
    }
    
    /// Initialize with built-in commands
    pub async fn initialize(&mut self) -> Result<()> {
        // Register core commands
        self.register_command("snippet", Box::new(SnippetCommand::new(self.context.clone()))).await?;
        self.register_command("theme", Box::new(ThemeCommand::new(self.context.clone()))).await?;
        self.register_command("schema", Box::new(SchemaCommand::new(self.context.clone()))).await?;
        self.register_command("whois", Box::new(WhoisCommand::new(self.context.clone()))).await?;
        self.register_command("server", Box::new(ServerCommand::new(self.context.clone()))).await?;
        self.register_command("cache", Box::new(CacheCommand::new(self.context.clone()))).await?;
        self.register_command("plugin", Box::new(PluginCommand::new(self.context.clone()))).await?;
        
        // Register aliases
        self.register_alias("s", "snippet");
        self.register_alias("t", "theme");
        self.register_alias("w", "whois");
        
        // Load plugin commands
        self.load_plugin_commands().await?;
        
        Ok(())
    }
    
    /// Register a command
    pub async fn register_command(&mut self, name: &str, command: Box<dyn Command>) -> Result<()> {
        if self.commands.contains_key(name) {
            return Err(ReedError::CommandExists(name.to_string()));
        }
        
        self.commands.insert(name.to_string(), command);
        Ok(())
    }
    
    /// Register command alias
    pub fn register_alias(&mut self, alias: &str, command: &str) {
        self.aliases.insert(alias.to_string(), command.to_string());
    }
    
    /// Execute command by name
    pub async fn execute(&self, name: &str, args: Vec<String>) -> Result<CommandOutput> {
        // Resolve alias
        let command_name = self.aliases.get(name).unwrap_or(&name.to_string());
        
        // Find command
        let command = self.commands.get(command_name)
            .ok_or_else(|| ReedError::CommandNotFound(name.to_string()))?;
        
        // Execute with context
        command.execute(&args, &self.context).await
    }
}
```

## Command Trait

```rust
/// Base trait for all commands
#[async_trait]
pub trait Command: Send + Sync {
    /// Get command name
    fn name(&self) -> &str;
    
    /// Get command description
    fn description(&self) -> &str;
    
    /// Get command help
    fn help(&self) -> CommandHelp;
    
    /// Execute command
    async fn execute(&self, args: &[String], context: &CliContext) -> Result<CommandOutput>;
    
    /// Validate arguments
    fn validate_args(&self, args: &[String]) -> Result<()> {
        Ok(())
    }
    
    /// Get subcommands
    fn subcommands(&self) -> Vec<&str> {
        vec![]
    }
}

/// Command help information
#[derive(Debug, Clone)]
pub struct CommandHelp {
    pub usage: String,
    pub description: String,
    pub examples: Vec<Example>,
    pub flags: Vec<Flag>,
    pub subcommands: Vec<SubcommandHelp>,
}

#[derive(Debug, Clone)]
pub struct Example {
    pub command: String,
    pub description: String,
}

#[derive(Debug, Clone)]
pub struct Flag {
    pub short: Option<char>,
    pub long: String,
    pub description: String,
    pub value_name: Option<String>,
}

#[derive(Debug, Clone)]
pub struct SubcommandHelp {
    pub name: String,
    pub description: String,
    pub usage: String,
}

/// Command output
#[derive(Debug)]
pub enum CommandOutput {
    Success(OutputData),
    Error(String),
    Empty,
}

#[derive(Debug)]
pub enum OutputData {
    Text(String),
    Json(serde_json::Value),
    Table(TableData),
    List(Vec<String>),
}

#[derive(Debug)]
pub struct TableData {
    pub headers: Vec<String>,
    pub rows: Vec<Vec<String>>,
}
```

## CLI Context

```rust
/// Shared context for all commands
pub struct CliContext {
    pub config: Arc<Config>,
    pub db: Arc<DatabaseConnection>,
    pub redis: Arc<RedisManager>,
    pub snippet_manager: Arc<SnippetManager>,
    pub theme_manager: Arc<ThemeManager>,
    pub registry_manager: Arc<RegistryManager>,
    pub output_format: OutputFormat,
    pub is_interactive: bool,
    pub is_piped: bool,
}

impl CliContext {
    pub async fn new(config: Config, output_format: OutputFormat) -> Result<Self> {
        // Initialize connections
        let db = Arc::new(DatabaseConnection::new(&config.database).await?);
        let redis = Arc::new(RedisManager::new(&config.redis).await?);
        
        // Initialize managers
        let registry_manager = Arc::new(RegistryManager::new(config.registry_path.clone()));
        registry_manager.initialize().await?;
        
        let snippet_manager = Arc::new(SnippetManager::new(
            // ... dependencies
        ));
        
        let theme_manager = Arc::new(ThemeManager::new(config.clone()));
        theme_manager.initialize().await?;
        
        Ok(Self {
            config: Arc::new(config),
            db,
            redis,
            snippet_manager,
            theme_manager,
            registry_manager,
            output_format,
            is_interactive: atty::is(atty::Stream::Stdout),
            is_piped: !atty::is(atty::Stream::Stdin),
        })
    }
}
```

## Built-in Commands

```rust
/// Snippet command implementation
pub struct SnippetCommand {
    context: Arc<CliContext>,
}

impl SnippetCommand {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self { context }
    }
    
    async fn handle_list(&self, args: &SnippetListArgs) -> Result<CommandOutput> {
        let snippets = if let Some(ref snippet_type) = args.r#type {
            self.context.snippet_manager
                .get_by_type(snippet_type, args.limit, 0)
                .await?
        } else if let Some(parent_id) = args.parent {
            self.context.snippet_manager
                .get_children(parent_id, args.limit)
                .await?
        } else {
            self.context.snippet_manager
                .list_recent(args.limit)
                .await?
        };
        
        // Format output
        match self.context.output_format {
            OutputFormat::Json => {
                Ok(CommandOutput::Success(OutputData::Json(
                    serde_json::to_value(&snippets)?
                )))
            }
            OutputFormat::Table => {
                let table = self.format_snippet_table(&snippets, args.all);
                Ok(CommandOutput::Success(OutputData::Table(table)))
            }
            _ => {
                let lines: Vec<String> = snippets.iter()
                    .map(|s| format!("{}\t{}", s.id, s.semantic_name.as_ref()
                        .map(|n| n.as_str())
                        .unwrap_or("(unnamed)")))
                    .collect();
                Ok(CommandOutput::Success(OutputData::List(lines)))
            }
        }
    }
    
    fn format_snippet_table(&self, snippets: &[Entity], show_all: bool) -> TableData {
        let mut headers = vec![
            "ID".to_string(),
            "Type".to_string(),
            "Name".to_string(),
            "Created".to_string(),
        ];
        
        if show_all {
            headers.extend(vec![
                "Updated".to_string(),
                "Created By".to_string(),
            ]);
        }
        
        let rows: Vec<Vec<String>> = snippets.iter()
            .map(|s| {
                let mut row = vec![
                    s.id.to_string(),
                    s.data.get("snippet_type")
                        .and_then(|v| v.as_str())
                        .unwrap_or("unknown")
                        .to_string(),
                    s.semantic_name.as_ref()
                        .map(|n| n.as_str())
                        .unwrap_or("(unnamed)")
                        .to_string(),
                    s.created_at.format("%Y-%m-%d %H:%M").to_string(),
                ];
                
                if show_all {
                    row.extend(vec![
                        s.updated_at.format("%Y-%m-%d %H:%M").to_string(),
                        s.created_by.clone(),
                    ]);
                }
                
                row
            })
            .collect();
        
        TableData { headers, rows }
    }
}

#[async_trait]
impl Command for SnippetCommand {
    fn name(&self) -> &str {
        "snippet"
    }
    
    fn description(&self) -> &str {
        "Manage content snippets"
    }
    
    fn help(&self) -> CommandHelp {
        CommandHelp {
            usage: "reed snippet <subcommand> [options]".to_string(),
            description: "Create, read, update, and delete content snippets".to_string(),
            examples: vec![
                Example {
                    command: "reed snippet list --type hero-banner".to_string(),
                    description: "List all hero banner snippets".to_string(),
                },
                Example {
                    command: "reed snippet create article title=\"Hello World\"".to_string(),
                    description: "Create new article snippet".to_string(),
                },
            ],
            flags: vec![
                Flag {
                    short: Some('f'),
                    long: "format".to_string(),
                    description: "Output format".to_string(),
                    value_name: Some("FORMAT".to_string()),
                },
            ],
            subcommands: vec![
                SubcommandHelp {
                    name: "list".to_string(),
                    description: "List snippets".to_string(),
                    usage: "reed snippet list [--type TYPE] [--limit N]".to_string(),
                },
                SubcommandHelp {
                    name: "create".to_string(),
                    description: "Create new snippet".to_string(),
                    usage: "reed snippet create TYPE [KEY=VALUE...]".to_string(),
                },
            ],
        }
    }
    
    async fn execute(&self, args: &[String], context: &CliContext) -> Result<CommandOutput> {
        if args.is_empty() {
            return Ok(CommandOutput::Success(OutputData::Text(self.help().to_string())));
        }
        
        match args[0].as_str() {
            "list" => {
                let list_args = parse_list_args(&args[1..])?;
                self.handle_list(&list_args).await
            }
            "get" => {
                let get_args = parse_get_args(&args[1..])?;
                self.handle_get(&get_args).await
            }
            "create" => {
                let create_args = parse_create_args(&args[1..])?;
                self.handle_create(&create_args).await
            }
            "update" => {
                let update_args = parse_update_args(&args[1..])?;
                self.handle_update(&update_args).await
            }
            "delete" => {
                let delete_args = parse_delete_args(&args[1..])?;
                self.handle_delete(&delete_args).await
            }
            _ => Err(ReedError::InvalidSubcommand(args[0].clone())),
        }
    }
}
```

## Plugin Commands

```rust
/// Load commands from plugins
impl CommandRegistry {
    async fn load_plugin_commands(&mut self) -> Result<()> {
        let plugin_manager = &self.context.plugin_manager;
        let plugins = plugin_manager.list_active_plugins().await?;
        
        for plugin in plugins {
            if let Some(commands) = plugin.get_commands() {
                for cmd_config in commands {
                    let plugin_cmd = PluginCommand::new(
                        cmd_config,
                        plugin.id.clone(),
                        self.context.clone(),
                    );
                    
                    self.register_command(
                        &cmd_config.name,
                        Box::new(plugin_cmd),
                    ).await?;
                }
            }
        }
        
        Ok(())
    }
}

/// Wrapper for plugin-provided commands
pub struct PluginCommand {
    config: PluginCommandConfig,
    plugin_id: String,
    context: Arc<CliContext>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginCommandConfig {
    pub name: String,
    pub description: String,
    pub usage: String,
    pub handler: String, // Function name in plugin
}

#[async_trait]
impl Command for PluginCommand {
    fn name(&self) -> &str {
        &self.config.name
    }
    
    fn description(&self) -> &str {
        &self.config.description
    }
    
    fn help(&self) -> CommandHelp {
        CommandHelp {
            usage: self.config.usage.clone(),
            description: self.config.description.clone(),
            examples: vec![],
            flags: vec![],
            subcommands: vec![],
        }
    }
    
    async fn execute(&self, args: &[String], context: &CliContext) -> Result<CommandOutput> {
        // Call plugin handler
        let result = context.plugin_manager
            .call_plugin_function(
                &self.plugin_id,
                &self.config.handler,
                serde_json::to_value(args)?,
            )
            .await?;
        
        // Convert plugin result to command output
        Ok(CommandOutput::Success(OutputData::Json(result)))
    }
}
```

## Command Helpers

```rust
/// Parse common command patterns
pub mod parse_helpers {
    use super::*;
    
    /// Parse UUID or semantic name
    pub fn parse_identifier(s: &str) -> Result<EntityIdentifier> {
        if let Ok(uuid) = Uuid::parse_str(s) {
            Ok(EntityIdentifier::Id(uuid))
        } else {
            Ok(EntityIdentifier::Name(s.to_string()))
        }
    }
    
    /// Parse key=value arguments
    pub fn parse_data_map(args: &[String]) -> Result<HashMap<String, serde_json::Value>> {
        let mut map = HashMap::new();
        
        for arg in args {
            if let Some(pos) = arg.find('=') {
                let (key, value_str) = arg.split_at(pos);
                let value = parse_value(&value_str[1..])?;
                map.insert(key.to_string(), value);
            }
        }
        
        Ok(map)
    }
    
    /// Parse value string to JSON
    fn parse_value(s: &str) -> Result<serde_json::Value> {
        // Try JSON first
        if s.starts_with('{') || s.starts_with('[') {
            return Ok(serde_json::from_str(s)?);
        }
        
        // Try number
        if let Ok(n) = s.parse::<i64>() {
            return Ok(serde_json::Value::Number(n.into()));
        }
        
        if let Ok(f) = s.parse::<f64>() {
            if let Some(n) = serde_json::Number::from_f64(f) {
                return Ok(serde_json::Value::Number(n));
            }
        }
        
        // Try boolean
        if s == "true" || s == "false" {
            return Ok(serde_json::Value::Bool(s == "true"));
        }
        
        // Default to string
        Ok(serde_json::Value::String(s.to_string()))
    }
}

#[derive(Debug, Clone)]
pub enum EntityIdentifier {
    Id(Uuid),
    Name(String),
}
```

## Command Execution Pipeline

```rust
/// Execute commands with middleware
pub struct CommandExecutor {
    registry: Arc<CommandRegistry>,
    middleware: Vec<Box<dyn CommandMiddleware>>,
}

#[async_trait]
pub trait CommandMiddleware: Send + Sync {
    async fn before_execute(
        &self,
        command: &str,
        args: &[String],
        context: &CliContext,
    ) -> Result<()>;
    
    async fn after_execute(
        &self,
        command: &str,
        args: &[String],
        output: &CommandOutput,
        context: &CliContext,
    ) -> Result<()>;
}

impl CommandExecutor {
    pub fn new(registry: Arc<CommandRegistry>) -> Self {
        Self {
            registry,
            middleware: vec![
                Box::new(LoggingMiddleware::new()),
                Box::new(TimingMiddleware::new()),
                Box::new(ValidationMiddleware::new()),
            ],
        }
    }
    
    pub async fn execute(&self, command: &str, args: Vec<String>) -> Result<CommandOutput> {
        // Run before middleware
        for mw in &self.middleware {
            mw.before_execute(command, &args, &self.registry.context).await?;
        }
        
        // Execute command
        let output = self.registry.execute(command, args.clone()).await?;
        
        // Run after middleware
        for mw in &self.middleware {
            mw.after_execute(command, &args, &output, &self.registry.context).await?;
        }
        
        Ok(output)
    }
}
```

## Summary

Dieses Command Registry System bietet:
- **Central Registry** - Alle Commands an einem Ort
- **Plugin Support** - Dynamisches Laden von Plugin Commands
- **Consistent Interface** - Einheitliche Command Struktur
- **Middleware Pipeline** - Logging, Timing, Validation
- **Flexible Output** - Multiple Output Formats

Die Command Registry macht CLI Commands erweiterbar und wartbar.
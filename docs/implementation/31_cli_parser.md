# CLI Parser System

## Overview

FreeBSD-style Command Parser für ReedCMS CLI. Chainable Commands mit konsistenter Syntax und Pipe Support.

## CLI Parser Core

```rust
use clap::{Parser, Subcommand, Args};
use std::io::{self, BufRead};

/// ReedCMS command line interface
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
#[command(propagate_version = true)]
pub struct Cli {
    /// Global flags
    #[command(flatten)]
    pub global: GlobalArgs,
    
    /// Subcommands
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Args)]
pub struct GlobalArgs {
    /// Config file path
    #[arg(short, long, value_name = "FILE", global = true)]
    pub config: Option<PathBuf>,
    
    /// Output format
    #[arg(short, long, value_enum, default_value = "table", global = true)]
    pub format: OutputFormat,
    
    /// Quiet mode (no output except errors)
    #[arg(short, long, global = true)]
    pub quiet: bool,
    
    /// Verbose output
    #[arg(short, long, action = clap::ArgAction::Count, global = true)]
    pub verbose: u8,
    
    /// Accept piped input
    #[arg(long, global = true)]
    pub pipe: bool,
    
    /// Non-interactive mode
    #[arg(long, global = true)]
    pub non_interactive: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, clap::ValueEnum)]
pub enum OutputFormat {
    Table,
    Json,
    Yaml,
    Csv,
    Plain,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Snippet operations
    Snippet(SnippetArgs),
    
    /// Theme operations
    Theme(ThemeArgs),
    
    /// Schema operations
    Schema(SchemaArgs),
    
    /// Debug with whois
    Whois(WhoisArgs),
    
    /// Server operations
    Server(ServerArgs),
    
    /// Cache operations
    Cache(CacheArgs),
    
    /// Plugin operations
    Plugin(PluginArgs),
}
```

## Command Structure

```rust
/// Snippet command arguments
#[derive(Args)]
pub struct SnippetArgs {
    #[command(subcommand)]
    pub command: SnippetCommands,
}

#[derive(Subcommand)]
pub enum SnippetCommands {
    /// List snippets
    List {
        /// Filter by type
        #[arg(short, long)]
        r#type: Option<String>,
        
        /// Filter by parent
        #[arg(short, long)]
        parent: Option<Uuid>,
        
        /// Limit results
        #[arg(short, long, default_value = "50")]
        limit: usize,
        
        /// Show all fields
        #[arg(short, long)]
        all: bool,
    },
    
    /// Get snippet by ID or name
    Get {
        /// Snippet ID or semantic name
        identifier: String,
        
        /// Output specific field
        #[arg(short, long)]
        field: Option<String>,
    },
    
    /// Create new snippet
    Create {
        /// Snippet type
        r#type: String,
        
        /// Parent ID
        #[arg(short, long)]
        parent: Option<Uuid>,
        
        /// Data as JSON or key=value pairs
        #[arg(value_parser = parse_data_args)]
        data: Vec<DataArg>,
        
        /// Read data from stdin
        #[arg(long, conflicts_with = "data")]
        stdin: bool,
    },
    
    /// Update existing snippet
    Update {
        /// Snippet ID
        id: Uuid,
        
        /// Updates as key=value pairs
        #[arg(value_parser = parse_data_args)]
        updates: Vec<DataArg>,
        
        /// Replace all data
        #[arg(long)]
        replace: bool,
    },
    
    /// Delete snippet
    Delete {
        /// Snippet ID
        id: Uuid,
        
        /// Force delete without confirmation
        #[arg(short, long)]
        force: bool,
        
        /// Delete recursively
        #[arg(short, long)]
        recursive: bool,
    },
    
    /// Search snippets
    Search {
        /// Search query
        query: String,
        
        /// Filter by type
        #[arg(short, long)]
        r#type: Option<String>,
        
        /// Limit results
        #[arg(short, long, default_value = "20")]
        limit: usize,
    },
}
```

## Data Argument Parsing

```rust
/// Represents key=value data argument
#[derive(Debug, Clone)]
pub struct DataArg {
    pub key: String,
    pub value: serde_json::Value,
}

/// Parse data arguments in various formats
fn parse_data_args(s: &str) -> Result<DataArg, String> {
    // Try key=value format
    if let Some(pos) = s.find('=') {
        let (key, value_str) = s.split_at(pos);
        let value_str = &value_str[1..]; // Skip '='
        
        // Try to parse as JSON
        let value = if value_str.starts_with('{') || value_str.starts_with('[') {
            serde_json::from_str(value_str)
                .map_err(|e| format!("Invalid JSON value: {}", e))?
        } else if let Ok(n) = value_str.parse::<i64>() {
            serde_json::Value::Number(n.into())
        } else if let Ok(f) = value_str.parse::<f64>() {
            serde_json::Number::from_f64(f)
                .map(serde_json::Value::Number)
                .ok_or_else(|| "Invalid float value".to_string())?
        } else if value_str == "true" || value_str == "false" {
            serde_json::Value::Bool(value_str == "true")
        } else {
            serde_json::Value::String(value_str.to_string())
        };
        
        Ok(DataArg {
            key: key.to_string(),
            value,
        })
    } else {
        Err(format!("Invalid data format: '{}'. Expected key=value", s))
    }
}

/// Merge data arguments into JSON object
pub fn merge_data_args(args: Vec<DataArg>) -> serde_json::Value {
    let mut obj = serde_json::Map::new();
    
    for arg in args {
        // Handle nested keys (e.g., meta.title=Hello)
        if arg.key.contains('.') {
            insert_nested(&mut obj, &arg.key, arg.value);
        } else {
            obj.insert(arg.key, arg.value);
        }
    }
    
    serde_json::Value::Object(obj)
}

/// Insert value at nested path
fn insert_nested(obj: &mut serde_json::Map<String, serde_json::Value>, path: &str, value: serde_json::Value) {
    let parts: Vec<&str> = path.split('.').collect();
    
    if parts.is_empty() {
        return;
    }
    
    if parts.len() == 1 {
        obj.insert(parts[0].to_string(), value);
        return;
    }
    
    // Create nested structure
    let key = parts[0].to_string();
    let remaining_path = parts[1..].join(".");
    
    let nested = obj.entry(key)
        .or_insert_with(|| serde_json::Value::Object(serde_json::Map::new()));
    
    if let serde_json::Value::Object(nested_obj) = nested {
        insert_nested(nested_obj, &remaining_path, value);
    }
}
```

## Pipe Support

```rust
/// Handle piped input/output
pub struct PipeHandler {
    input_buffer: Option<String>,
}

impl PipeHandler {
    pub fn new() -> Self {
        Self {
            input_buffer: None,
        }
    }
    
    /// Read piped input if available
    pub fn read_pipe_input(&mut self) -> Result<Option<String>> {
        if atty::is(atty::Stream::Stdin) {
            return Ok(None);
        }
        
        let stdin = io::stdin();
        let mut buffer = String::new();
        
        for line in stdin.lock().lines() {
            buffer.push_str(&line?);
            buffer.push('\n');
        }
        
        if !buffer.is_empty() {
            self.input_buffer = Some(buffer.clone());
            Ok(Some(buffer))
        } else {
            Ok(None)
        }
    }
    
    /// Parse piped JSON input
    pub fn parse_json_input(&self) -> Result<Option<serde_json::Value>> {
        if let Some(ref input) = self.input_buffer {
            let value = serde_json::from_str(input)?;
            Ok(Some(value))
        } else {
            Ok(None)
        }
    }
    
    /// Parse piped ID list
    pub fn parse_id_list(&self) -> Result<Vec<String>> {
        if let Some(ref input) = self.input_buffer {
            Ok(input.lines()
                .map(|s| s.trim().to_string())
                .filter(|s| !s.is_empty())
                .collect())
        } else {
            Ok(vec![])
        }
    }
}

/// Check if output should be piped
pub fn should_pipe_output(format: OutputFormat) -> bool {
    !atty::is(atty::Stream::Stdout) || matches!(format, OutputFormat::Json | OutputFormat::Plain)
}
```

## Command Chaining

```rust
/// Support for command chaining with &&, ||, ;
pub struct CommandChain {
    commands: Vec<ChainedCommand>,
}

#[derive(Debug)]
struct ChainedCommand {
    command: String,
    args: Vec<String>,
    operator: ChainOperator,
}

#[derive(Debug, Clone, Copy)]
enum ChainOperator {
    And,        // &&
    Or,         // ||
    Semicolon,  // ;
    Pipe,       // |
}

impl CommandChain {
    /// Parse command chain from shell-like syntax
    pub fn parse(input: &str) -> Result<Self> {
        let mut commands = Vec::new();
        let mut current_cmd = String::new();
        let mut current_args = Vec::new();
        let mut in_quotes = false;
        let mut escape_next = false;
        
        let chars: Vec<char> = input.chars().collect();
        let mut i = 0;
        
        while i < chars.len() {
            let ch = chars[i];
            
            if escape_next {
                current_args.last_mut()
                    .unwrap_or(&mut current_cmd)
                    .push(ch);
                escape_next = false;
                i += 1;
                continue;
            }
            
            match ch {
                '\\' => escape_next = true,
                '"' => in_quotes = !in_quotes,
                ' ' if !in_quotes => {
                    if !current_cmd.is_empty() && current_args.is_empty() {
                        // First space after command
                        current_args.push(String::new());
                    } else if let Some(last) = current_args.last() {
                        if !last.is_empty() {
                            current_args.push(String::new());
                        }
                    }
                }
                '&' if !in_quotes && i + 1 < chars.len() && chars[i + 1] == '&' => {
                    commands.push(ChainedCommand {
                        command: current_cmd.clone(),
                        args: current_args.clone(),
                        operator: ChainOperator::And,
                    });
                    current_cmd.clear();
                    current_args.clear();
                    i += 1; // Skip second &
                }
                '|' if !in_quotes => {
                    if i + 1 < chars.len() && chars[i + 1] == '|' {
                        commands.push(ChainedCommand {
                            command: current_cmd.clone(),
                            args: current_args.clone(),
                            operator: ChainOperator::Or,
                        });
                        i += 1; // Skip second |
                    } else {
                        commands.push(ChainedCommand {
                            command: current_cmd.clone(),
                            args: current_args.clone(),
                            operator: ChainOperator::Pipe,
                        });
                    }
                    current_cmd.clear();
                    current_args.clear();
                }
                ';' if !in_quotes => {
                    commands.push(ChainedCommand {
                        command: current_cmd.clone(),
                        args: current_args.clone(),
                        operator: ChainOperator::Semicolon,
                    });
                    current_cmd.clear();
                    current_args.clear();
                }
                _ => {
                    if current_args.is_empty() {
                        current_cmd.push(ch);
                    } else {
                        current_args.last_mut().unwrap().push(ch);
                    }
                }
            }
            
            i += 1;
        }
        
        // Add final command
        if !current_cmd.is_empty() {
            commands.push(ChainedCommand {
                command: current_cmd,
                args: current_args,
                operator: ChainOperator::Semicolon,
            });
        }
        
        Ok(Self { commands })
    }
    
    /// Execute command chain
    pub async fn execute(&self, ctx: &CliContext) -> Result<()> {
        let mut last_result = Ok(());
        let mut pipe_data: Option<String> = None;
        
        for cmd in &self.commands {
            // Check if we should execute based on operator
            match cmd.operator {
                ChainOperator::And => {
                    if last_result.is_err() {
                        continue;
                    }
                }
                ChainOperator::Or => {
                    if last_result.is_ok() {
                        continue;
                    }
                }
                _ => {}
            }
            
            // Execute command
            last_result = self.execute_single(cmd, ctx, pipe_data.as_deref()).await;
            
            // Handle pipe
            if matches!(cmd.operator, ChainOperator::Pipe) {
                if let Ok(output) = &last_result {
                    pipe_data = Some(output.clone());
                }
            }
        }
        
        last_result.map(|_| ())
    }
}
```

## Interactive Mode

```rust
/// Interactive CLI mode
pub struct InteractiveMode {
    context: CliContext,
    history: Vec<String>,
    completer: CommandCompleter,
}

impl InteractiveMode {
    pub async fn run(&mut self) -> Result<()> {
        println!("ReedCMS Interactive Mode");
        println!("Type 'help' for commands, 'exit' to quit");
        
        loop {
            // Print prompt
            print!("reed> ");
            io::stdout().flush()?;
            
            // Read input
            let mut input = String::new();
            io::stdin().read_line(&mut input)?;
            let input = input.trim();
            
            if input.is_empty() {
                continue;
            }
            
            // Handle special commands
            match input {
                "exit" | "quit" => break,
                "help" => {
                    self.show_help();
                    continue;
                }
                "history" => {
                    self.show_history();
                    continue;
                }
                _ => {}
            }
            
            // Add to history
            self.history.push(input.to_string());
            
            // Parse and execute
            match CommandChain::parse(input) {
                Ok(chain) => {
                    if let Err(e) = chain.execute(&self.context).await {
                        eprintln!("Error: {}", e);
                    }
                }
                Err(e) => {
                    eprintln!("Parse error: {}", e);
                }
            }
        }
        
        Ok(())
    }
    
    fn show_help(&self) {
        println!("Available commands:");
        println!("  snippet  - Manage snippets");
        println!("  theme    - Manage themes");
        println!("  schema   - Manage registry schemas");
        println!("  whois    - Debug entities");
        println!("  cache    - Cache operations");
        println!("  help     - Show this help");
        println!("  history  - Show command history");
        println!("  exit     - Exit interactive mode");
    }
    
    fn show_history(&self) {
        for (i, cmd) in self.history.iter().enumerate() {
            println!("{:4}: {}", i + 1, cmd);
        }
    }
}
```

## Command Completion

```rust
/// Command completion for interactive mode
pub struct CommandCompleter {
    commands: Vec<CommandInfo>,
}

#[derive(Debug, Clone)]
struct CommandInfo {
    name: String,
    subcommands: Vec<String>,
    flags: Vec<String>,
}

impl CommandCompleter {
    pub fn new() -> Self {
        let commands = vec![
            CommandInfo {
                name: "snippet".to_string(),
                subcommands: vec!["list", "get", "create", "update", "delete", "search"]
                    .into_iter().map(String::from).collect(),
                flags: vec!["--type", "--parent", "--limit", "--format"]
                    .into_iter().map(String::from).collect(),
            },
            CommandInfo {
                name: "theme".to_string(),
                subcommands: vec!["list", "activate", "info", "validate"]
                    .into_iter().map(String::from).collect(),
                flags: vec!["--context", "--validate"]
                    .into_iter().map(String::from).collect(),
            },
            // ... more commands
        ];
        
        Self { commands }
    }
    
    /// Get completions for partial input
    pub fn complete(&self, partial: &str) -> Vec<String> {
        let parts: Vec<&str> = partial.split_whitespace().collect();
        
        if parts.is_empty() {
            return self.commands.iter()
                .map(|c| c.name.clone())
                .collect();
        }
        
        if parts.len() == 1 {
            // Complete command name
            self.commands.iter()
                .map(|c| &c.name)
                .filter(|name| name.starts_with(parts[0]))
                .cloned()
                .collect()
        } else if parts.len() == 2 {
            // Complete subcommand
            if let Some(cmd) = self.commands.iter().find(|c| c.name == parts[0]) {
                cmd.subcommands.iter()
                    .filter(|sub| sub.starts_with(parts[1]))
                    .cloned()
                    .collect()
            } else {
                vec![]
            }
        } else {
            // Complete flags
            if let Some(cmd) = self.commands.iter().find(|c| c.name == parts[0]) {
                cmd.flags.iter()
                    .filter(|flag| flag.starts_with(parts.last().unwrap()))
                    .cloned()
                    .collect()
            } else {
                vec![]
            }
        }
    }
}
```

## Summary

Dieses CLI Parser System bietet:
- **FreeBSD-style Commands** - Konsistente, chainable Syntax
- **Pipe Support** - Unix-style Input/Output Piping
- **Data Parsing** - Flexible key=value und JSON Support
- **Interactive Mode** - REPL mit Command Completion
- **Command Chaining** - &&, ||, ; und | Operators

Der CLI Parser macht ReedCMS zu einem mächtigen Command-Line Tool.
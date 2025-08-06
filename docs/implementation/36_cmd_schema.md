# Schema CLI Commands

## Overview

Schema Management Commands für ReedCMS CLI. Registry Schema Operations, Field Management und Type Validation.

## Schema Command Implementation

```rust
use clap::{Args, Subcommand};
use std::path::PathBuf;

/// Schema command implementation
pub struct SchemaCommand {
    registry: Arc<RegistryManager>,
    validator: Arc<FieldValidators>,
    context: Arc<CliContext>,
}

impl SchemaCommand {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self {
            registry: context.registry_manager.clone(),
            validator: context.field_validators.clone(),
            context,
        }
    }
    
    /// Execute schema subcommand
    pub async fn execute(&self, args: &SchemaArgs) -> Result<CommandOutput> {
        match &args.command {
            SchemaCommands::List(list_args) => self.handle_list(list_args).await,
            SchemaCommands::Show(show_args) => self.handle_show(show_args).await,
            SchemaCommands::Create(create_args) => self.handle_create(create_args).await,
            SchemaCommands::Update(update_args) => self.handle_update(update_args).await,
            SchemaCommands::Delete(delete_args) => self.handle_delete(delete_args).await,
            SchemaCommands::Validate(validate_args) => self.handle_validate(validate_args).await,
            SchemaCommands::Export(export_args) => self.handle_export(export_args).await,
            SchemaCommands::Import(import_args) => self.handle_import(import_args).await,
            SchemaCommands::Field(field_args) => self.handle_field(field_args).await,
            SchemaCommands::Meta(meta_args) => self.handle_meta(meta_args).await,
        }
    }
}

/// Schema command arguments
#[derive(Args)]
pub struct SchemaArgs {
    #[command(subcommand)]
    pub command: SchemaCommands,
}

#[derive(Subcommand)]
pub enum SchemaCommands {
    /// List registered schemas
    List(ListArgs),
    
    /// Show schema details
    Show(ShowArgs),
    
    /// Create new schema
    Create(CreateArgs),
    
    /// Update existing schema
    Update(UpdateArgs),
    
    /// Delete schema
    Delete(DeleteArgs),
    
    /// Validate data against schema
    Validate(ValidateArgs),
    
    /// Export schemas
    Export(ExportArgs),
    
    /// Import schemas
    Import(ImportArgs),
    
    /// Manage schema fields
    Field(FieldArgs),
    
    /// Manage meta properties
    Meta(MetaArgs),
}
```

## List Command

```rust
#[derive(Args)]
pub struct ListArgs {
    /// Filter by capability
    #[arg(long)]
    pub capability: Option<String>,
    
    /// Show field counts
    #[arg(long)]
    pub fields: bool,
    
    /// Show meta properties
    #[arg(long)]
    pub meta: bool,
    
    /// Output as tree
    #[arg(long)]
    pub tree: bool,
}

impl SchemaCommand {
    async fn handle_list(&self, args: &ListArgs) -> Result<CommandOutput> {
        let schemas = self.registry.list_snippet_types().await;
        
        // Filter by capability
        let filtered = if let Some(ref cap) = args.capability {
            let capability = self.parse_capability(cap)?;
            self.registry.find_by_capability(capability).await
        } else {
            schemas
        };
        
        // Load full definitions if needed
        let definitions = if args.fields || args.meta {
            let mut defs = Vec::new();
            for schema_name in &filtered {
                let def = self.registry.get_definition(schema_name).await?;
                defs.push(def);
            }
            Some(defs)
        } else {
            None
        };
        
        // Format output
        if args.tree {
            self.format_schema_tree(&filtered, definitions.as_deref())
        } else {
            self.format_schema_list(&filtered, definitions.as_deref(), args)
        }
    }
    
    /// Format schema list as table
    fn format_schema_list(
        &self,
        schemas: &[String],
        definitions: Option<&[SnippetDefinition]>,
        args: &ListArgs,
    ) -> Result<CommandOutput> {
        let mut headers = vec!["Name".to_string(), "Display Name".to_string()];
        
        if args.fields {
            headers.push("Fields".to_string());
        }
        
        if args.meta {
            headers.extend(vec![
                "Routable".to_string(),
                "Navigable".to_string(),
                "Searchable".to_string(),
            ]);
        }
        
        let mut rows = Vec::new();
        
        for (i, schema) in schemas.iter().enumerate() {
            let mut row = vec![schema.clone()];
            
            if let Some(defs) = definitions {
                if let Some(def) = defs.get(i) {
                    row.push(def.display_name.clone());
                    
                    if args.fields {
                        row.push(def.fields.len().to_string());
                    }
                    
                    if args.meta {
                        row.push(if def.meta.is_routable { "✓" } else { "✗" }.to_string());
                        row.push(if def.meta.is_navigable { "✓" } else { "✗" }.to_string());
                        row.push(if def.meta.is_searchable { "✓" } else { "✗" }.to_string());
                    }
                } else {
                    row.push("Unknown".to_string());
                }
            } else {
                row.push(self.humanize_name(schema));
            }
            
            rows.push(row);
        }
        
        Ok(CommandOutput::Success(OutputData::Table(TableData {
            headers,
            rows,
        })))
    }
    
    fn humanize_name(&self, name: &str) -> String {
        name.split('-')
            .map(|part| {
                let mut chars = part.chars();
                match chars.next() {
                    None => String::new(),
                    Some(first) => first.to_uppercase().chain(chars).collect(),
                }
            })
            .collect::<Vec<_>>()
            .join(" ")
    }
}
```

## Show Command

```rust
#[derive(Args)]
pub struct ShowArgs {
    /// Schema name
    pub name: String,
    
    /// Show as JSON
    #[arg(long)]
    pub json: bool,
    
    /// Show validation rules
    #[arg(long)]
    pub rules: bool,
    
    /// Show example data
    #[arg(long)]
    pub example: bool,
}

impl SchemaCommand {
    async fn handle_show(&self, args: &ShowArgs) -> Result<CommandOutput> {
        let definition = self.registry.get_definition(&args.name).await?;
        
        if args.json {
            // Export as JSON schema
            let json_schema = self.definition_to_json_schema(&definition)?;
            Ok(CommandOutput::Success(OutputData::Json(json_schema)))
        } else if args.example {
            // Generate example data
            let example = self.generate_example_data(&definition)?;
            Ok(CommandOutput::Success(OutputData::Json(example)))
        } else {
            // Format human-readable output
            self.format_schema_details(&definition, args.rules)
        }
    }
    
    /// Format schema details
    fn format_schema_details(
        &self,
        definition: &SnippetDefinition,
        show_rules: bool,
    ) -> Result<CommandOutput> {
        let mut output = String::new();
        
        // Header
        output.push_str(&format!("Schema: {}\n", definition.name.bold()));
        output.push_str(&format!("Display Name: {}\n", definition.display_name));
        output.push_str(&format!("Description: {}\n", definition.description));
        output.push_str("\n");
        
        // Meta properties
        output.push_str("Meta Properties:\n");
        output.push_str(&format!("  Routable:    {}\n", 
            if definition.meta.is_routable { "Yes".green() } else { "No".red() }));
        output.push_str(&format!("  Navigable:   {}\n",
            if definition.meta.is_navigable { "Yes".green() } else { "No".red() }));
        output.push_str(&format!("  Searchable:  {}\n",
            if definition.meta.is_searchable { "Yes".green() } else { "No".red() }));
        
        if let Some(ttl) = definition.meta.cache_ttl {
            output.push_str(&format!("  Cache TTL:   {} seconds\n", ttl));
        }
        
        if let Some(max) = definition.meta.max_instances {
            output.push_str(&format!("  Max Instances: {}\n", max));
        }
        
        if !definition.meta.allowed_child_types.is_empty() {
            output.push_str(&format!("  Allowed Children: {}\n", 
                definition.meta.allowed_child_types.join(", ")));
        }
        
        output.push_str("\n");
        
        // Fields
        output.push_str("Fields:\n");
        for field in &definition.fields {
            let required = if field.required { "*".red() } else { " ".normal() };
            let localized = if field.localized { " [L]".cyan() } else { "".normal() };
            
            output.push_str(&format!(
                "  {}{} {} ({}){}\n",
                required,
                field.name.blue(),
                field.display_name,
                self.format_field_type(&field.field_type),
                localized
            ));
            
            if let Some(ref help) = field.help_text {
                output.push_str(&format!("      {}\n", help.dim()));
            }
            
            if let Some(ref default) = field.default_value {
                output.push_str(&format!("      Default: {}\n", 
                    serde_json::to_string(default)?.yellow()));
            }
        }
        
        // Validation rules
        if show_rules && !definition.validation_rules.is_empty() {
            output.push_str("\nValidation Rules:\n");
            for rule in &definition.validation_rules {
                output.push_str(&format!(
                    "  {} → {} ({})\n",
                    rule.field.blue(),
                    self.format_validation_type(&rule.rule_type),
                    rule.error_message.dim()
                ));
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
    
    fn format_field_type(&self, field_type: &FieldType) -> String {
        match field_type {
            FieldType::Text => "text".to_string(),
            FieldType::Textarea => "textarea".to_string(),
            FieldType::Number => "number".to_string(),
            FieldType::Boolean => "boolean".to_string(),
            FieldType::Date => "date".to_string(),
            FieldType::Email => "email".to_string(),
            FieldType::Url => "url".to_string(),
            FieldType::Select(options) => format!("select[{}]", options.len()),
            FieldType::Reference(entity) => format!("ref:{}", entity),
            FieldType::Json => "json".to_string(),
        }
    }
}
```

## Field Management

```rust
#[derive(Args)]
pub struct FieldArgs {
    #[command(subcommand)]
    pub command: FieldCommands,
}

#[derive(Subcommand)]
pub enum FieldCommands {
    /// Add field to schema
    Add {
        /// Schema name
        schema: String,
        
        /// Field name
        name: String,
        
        /// Field type
        #[arg(short = 't', long)]
        field_type: String,
        
        /// Required field
        #[arg(short, long)]
        required: bool,
        
        /// Default value
        #[arg(short, long)]
        default: Option<String>,
        
        /// Help text
        #[arg(long)]
        help: Option<String>,
        
        /// Localized field
        #[arg(short, long)]
        localized: bool,
    },
    
    /// Update field in schema
    Update {
        /// Schema name
        schema: String,
        
        /// Field name
        name: String,
        
        /// New required status
        #[arg(long)]
        required: Option<bool>,
        
        /// New default value
        #[arg(long)]
        default: Option<String>,
        
        /// New help text
        #[arg(long)]
        help: Option<String>,
    },
    
    /// Remove field from schema
    Remove {
        /// Schema name
        schema: String,
        
        /// Field name
        name: String,
        
        /// Force removal
        #[arg(short, long)]
        force: bool,
    },
}

impl SchemaCommand {
    async fn handle_field(&self, args: &FieldArgs) -> Result<CommandOutput> {
        match &args.command {
            FieldCommands::Add { schema, name, field_type, required, default, help, localized } => {
                self.add_field(
                    schema,
                    name,
                    field_type,
                    *required,
                    default.as_deref(),
                    help.as_deref(),
                    *localized,
                ).await
            }
            FieldCommands::Update { schema, name, required, default, help } => {
                self.update_field(schema, name, *required, default.as_deref(), help.as_deref()).await
            }
            FieldCommands::Remove { schema, name, force } => {
                self.remove_field(schema, name, *force).await
            }
        }
    }
    
    /// Add field to schema
    async fn add_field(
        &self,
        schema_name: &str,
        field_name: &str,
        field_type_str: &str,
        required: bool,
        default: Option<&str>,
        help: Option<&str>,
        localized: bool,
    ) -> Result<CommandOutput> {
        // Get current definition
        let mut definition = self.registry.get_definition(schema_name).await?;
        
        // Check if field already exists
        if definition.fields.iter().any(|f| f.name == field_name) {
            return Err(ReedError::AlreadyExists(
                format!("Field '{}' already exists in schema '{}'", field_name, schema_name)
            ));
        }
        
        // Parse field type
        let field_type = self.parse_field_type(field_type_str)?;
        
        // Parse default value
        let default_value = if let Some(def) = default {
            Some(serde_json::from_str(def)?)
        } else {
            None
        };
        
        // Create new field
        let field = FieldDefinition {
            name: field_name.to_string(),
            display_name: self.humanize_name(field_name),
            field_type,
            required,
            default_value,
            validation: None,
            help_text: help.map(String::from),
            localized,
        };
        
        // Add to definition
        definition.fields.push(field);
        
        // Update in registry
        self.update_schema_definition(&definition).await?;
        
        Ok(CommandOutput::Success(OutputData::Text(format!(
            "Field '{}' added to schema '{}'",
            field_name,
            schema_name
        ))))
    }
    
    /// Parse field type string
    fn parse_field_type(&self, type_str: &str) -> Result<FieldType> {
        match type_str {
            "text" => Ok(FieldType::Text),
            "textarea" => Ok(FieldType::Textarea),
            "number" => Ok(FieldType::Number),
            "boolean" | "bool" => Ok(FieldType::Boolean),
            "date" => Ok(FieldType::Date),
            "email" => Ok(FieldType::Email),
            "url" => Ok(FieldType::Url),
            "json" => Ok(FieldType::Json),
            s if s.starts_with("ref:") => {
                let entity = s.strip_prefix("ref:").unwrap();
                Ok(FieldType::Reference(entity.to_string()))
            }
            s if s.starts_with("select:") => {
                let options_str = s.strip_prefix("select:").unwrap();
                let options: Vec<SelectOption> = options_str
                    .split('|')
                    .map(|opt| {
                        let parts: Vec<&str> = opt.split('=').collect();
                        SelectOption {
                            value: parts.get(0).unwrap_or(&"").to_string(),
                            label: parts.get(1).unwrap_or(parts.get(0).unwrap_or(&"")).to_string(),
                        }
                    })
                    .collect();
                Ok(FieldType::Select(options))
            }
            _ => Err(ReedError::InvalidFieldType(type_str.to_string())),
        }
    }
}
```

## Validation Command

```rust
#[derive(Args)]
pub struct ValidateArgs {
    /// Schema name
    pub schema: String,
    
    /// Data to validate (JSON)
    #[arg(value_parser = parse_json_value)]
    pub data: Option<serde_json::Value>,
    
    /// Read data from file
    #[arg(short, long, conflicts_with = "data")]
    pub file: Option<PathBuf>,
    
    /// Read data from stdin
    #[arg(long, conflicts_with_all = &["data", "file"])]
    pub stdin: bool,
    
    /// Show detailed validation errors
    #[arg(short, long)]
    pub verbose: bool,
}

impl SchemaCommand {
    async fn handle_validate(&self, args: &ValidateArgs) -> Result<CommandOutput> {
        // Get schema definition
        let definition = self.registry.get_definition(&args.schema).await?;
        
        // Get data to validate
        let data = if args.stdin {
            self.read_stdin_json().await?
        } else if let Some(ref file) = args.file {
            self.read_file_json(file).await?
        } else if let Some(ref data) = args.data {
            data.clone()
        } else {
            return Err(ReedError::MissingArgument("No data provided".into()));
        };
        
        // Convert to HashMap
        let data_map = data.as_object()
            .ok_or_else(|| ReedError::InvalidData("Data must be an object".into()))?
            .clone();
        
        // Validate
        let result = self.validator
            .validate_snippet(&args.schema, &data_map)
            .await?;
        
        // Format result
        if result.is_valid {
            Ok(CommandOutput::Success(OutputData::Text(
                "✓ Validation passed".green().to_string()
            )))
        } else {
            if args.verbose {
                self.format_validation_errors(&result)
            } else {
                Ok(CommandOutput::Error(format!(
                    "Validation failed: {} errors",
                    result.errors.len()
                )))
            }
        }
    }
    
    /// Format validation errors
    fn format_validation_errors(&self, result: &ValidationResult) -> Result<CommandOutput> {
        let mut output = String::new();
        
        output.push_str(&"✗ Validation failed\n\n".red().to_string());
        
        output.push_str("Errors:\n");
        for error in &result.errors {
            output.push_str(&format!(
                "  • {} {}\n    {}\n",
                error.field.blue(),
                format!("[{}]", error.code).yellow(),
                error.message
            ));
        }
        
        if !result.warnings.is_empty() {
            output.push_str("\nWarnings:\n");
            for warning in &result.warnings {
                output.push_str(&format!("  • {}\n", warning.yellow()));
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
}
```

## Export/Import Commands

```rust
#[derive(Args)]
pub struct ExportArgs {
    /// Output format
    #[arg(short, long, value_enum, default_value = "csv")]
    pub format: ExportFormat,
    
    /// Output directory
    #[arg(short, long)]
    pub output: PathBuf,
    
    /// Include meta configuration
    #[arg(long)]
    pub meta: bool,
    
    /// Include validation rules
    #[arg(long)]
    pub rules: bool,
}

#[derive(Debug, Clone, Copy, ValueEnum)]
pub enum ExportFormat {
    Csv,
    Json,
    Yaml,
}

impl SchemaCommand {
    async fn handle_export(&self, args: &ExportArgs) -> Result<CommandOutput> {
        // Ensure output directory exists
        tokio::fs::create_dir_all(&args.output).await?;
        
        match args.format {
            ExportFormat::Csv => self.export_csv(&args.output, args.meta, args.rules).await,
            ExportFormat::Json => self.export_json(&args.output).await,
            ExportFormat::Yaml => self.export_yaml(&args.output).await,
        }
    }
    
    /// Export schemas as CSV
    async fn export_csv(
        &self,
        output_dir: &Path,
        include_meta: bool,
        include_rules: bool,
    ) -> Result<CommandOutput> {
        let schemas = self.registry.list_snippet_types().await;
        let mut total_exported = 0;
        
        // Export snippets.csv
        let snippets_path = output_dir.join("snippets.csv");
        let mut writer = csv::Writer::from_path(&snippets_path)?;
        
        // Write headers
        writer.write_record(&[
            "snippet_name",
            "field_name", 
            "field_type",
            "required",
            "default_value",
            "localized",
            "help_text"
        ])?;
        
        for schema_name in &schemas {
            let definition = self.registry.get_definition(schema_name).await?;
            
            for field in &definition.fields {
                writer.write_record(&[
                    schema_name,
                    &field.name,
                    &self.field_type_to_string(&field.field_type),
                    &field.required.to_string(),
                    &field.default_value.as_ref()
                        .map(|v| serde_json::to_string(v).unwrap_or_default())
                        .unwrap_or_default(),
                    &field.localized.to_string(),
                    &field.help_text.as_ref().unwrap_or(&String::new()),
                ])?;
                
                total_exported += 1;
            }
        }
        
        writer.flush()?;
        
        // Export meta_snippets.csv if requested
        if include_meta {
            let meta_path = output_dir.join("meta_snippets.csv");
            self.export_meta_csv(&meta_path, &schemas).await?;
        }
        
        // Export validation_rules.csv if requested
        if include_rules {
            let rules_path = output_dir.join("validation_rules.csv");
            self.export_rules_csv(&rules_path, &schemas).await?;
        }
        
        Ok(CommandOutput::Success(OutputData::Text(format!(
            "Exported {} schemas with {} fields to {}",
            schemas.len(),
            total_exported,
            output_dir.display()
        ))))
    }
}
```

## Summary

Diese Schema Commands bieten:
- **Schema Management** - List, Show, Create, Update, Delete
- **Field Operations** - Add, Update, Remove Fields
- **Validation** - Test Data gegen Schemas
- **Export/Import** - CSV, JSON, YAML Support
- **Meta Properties** - Routable, Navigable, Searchable Config

Schema Commands machen Registry Management transparent und einfach.
# Theme CLI Commands

## Overview

Theme Management Commands für ReedCMS CLI. Theme Activation, Validation und Context-aware Theme Operations.

## Theme Command Implementation

```rust
use clap::{Args, Subcommand};
use std::path::PathBuf;

/// Theme command implementation
pub struct ThemeCommand {
    manager: Arc<ThemeManager>,
    epc_resolver: Arc<EpcResolver>,
    context: Arc<CliContext>,
}

impl ThemeCommand {
    pub fn new(context: Arc<CliContext>) -> Self {
        Self {
            manager: context.theme_manager.clone(),
            epc_resolver: context.epc_resolver.clone(),
            context,
        }
    }
    
    /// Execute theme subcommand
    pub async fn execute(&self, args: &ThemeArgs) -> Result<CommandOutput> {
        match &args.command {
            ThemeCommands::List(list_args) => self.handle_list(list_args).await,
            ThemeCommands::Info(info_args) => self.handle_info(info_args).await,
            ThemeCommands::Activate(activate_args) => self.handle_activate(activate_args).await,
            ThemeCommands::Deactivate(deactivate_args) => self.handle_deactivate(deactivate_args).await,
            ThemeCommands::Validate(validate_args) => self.handle_validate(validate_args).await,
            ThemeCommands::Create(create_args) => self.handle_create(create_args).await,
            ThemeCommands::Chain(chain_args) => self.handle_chain(chain_args).await,
            ThemeCommands::Files(files_args) => self.handle_files(files_args).await,
            ThemeCommands::Override(override_args) => self.handle_override(override_args).await,
            ThemeCommands::Context(context_args) => self.handle_context(context_args).await,
        }
    }
}

/// Theme command arguments
#[derive(Args)]
pub struct ThemeArgs {
    #[command(subcommand)]
    pub command: ThemeCommands,
}

#[derive(Subcommand)]
pub enum ThemeCommands {
    /// List available themes
    List(ListArgs),
    
    /// Show theme information
    Info(InfoArgs),
    
    /// Activate theme
    Activate(ActivateArgs),
    
    /// Deactivate theme
    Deactivate(DeactivateArgs),
    
    /// Validate theme
    Validate(ValidateArgs),
    
    /// Create new theme
    Create(CreateArgs),
    
    /// Show theme chain
    Chain(ChainArgs),
    
    /// List theme files
    Files(FilesArgs),
    
    /// Show file overrides
    Override(OverrideArgs),
    
    /// Manage theme contexts
    Context(ContextArgs),
}
```

## List Command

```rust
#[derive(Args)]
pub struct ListArgs {
    /// Show only active themes
    #[arg(short, long)]
    pub active: bool,
    
    /// Show theme hierarchy
    #[arg(long)]
    pub tree: bool,
    
    /// Include theme metadata
    #[arg(short, long)]
    pub metadata: bool,
    
    /// Filter by context type
    #[arg(long)]
    pub context: Option<String>,
}

impl ThemeCommand {
    async fn handle_list(&self, args: &ListArgs) -> Result<CommandOutput> {
        let themes = self.manager.list_themes().await?;
        
        // Filter themes
        let filtered: Vec<_> = themes.into_iter()
            .filter(|theme| {
                if args.active && !theme.active {
                    return false;
                }
                
                if let Some(ref ctx) = args.context {
                    match ctx.as_str() {
                        "location" => matches!(theme.context, ThemeContext::Location(_)),
                        "season" => matches!(theme.context, ThemeContext::Season(_)),
                        "event" => matches!(theme.context, ThemeContext::Event(_)),
                        _ => false,
                    }
                } else {
                    true
                }
            })
            .collect();
        
        // Format output
        if args.tree {
            self.format_theme_tree(&filtered)
        } else {
            self.format_theme_list(&filtered, args.metadata)
        }
    }
    
    /// Format themes as tree
    fn format_theme_tree(&self, themes: &[Theme]) -> Result<CommandOutput> {
        let mut tree = TreeBuilder::new();
        
        // Build theme hierarchy
        let mut theme_map: HashMap<String, Vec<&Theme>> = HashMap::new();
        
        for theme in themes {
            let parent = theme.parent.as_deref().unwrap_or("(root)");
            theme_map.entry(parent.to_string())
                .or_insert_with(Vec::new)
                .push(theme);
        }
        
        // Recursive tree building
        fn add_theme_to_tree(
            tree: &mut TreeBuilder,
            theme: &Theme,
            theme_map: &HashMap<String, Vec<&Theme>>,
            indent: usize,
        ) {
            let prefix = "  ".repeat(indent);
            let status = if theme.active { "✓".green() } else { "✗".red() };
            let context = match &theme.context {
                ThemeContext::Default => "".to_string(),
                ThemeContext::Location(loc) => format!(" [location: {}]", loc).yellow().to_string(),
                ThemeContext::Season(s) => format!(" [season: {}]", s).cyan().to_string(),
                ThemeContext::Event(e) => format!(" [event: {}]", e).magenta().to_string(),
                ThemeContext::Custom(c) => format!(" [{}]", c).blue().to_string(),
            };
            
            tree.add_line(format!("{}{} {}{}", prefix, status, theme.name, context));
            
            // Add children
            if let Some(children) = theme_map.get(&theme.name) {
                for child in children {
                    add_theme_to_tree(tree, child, theme_map, indent + 1);
                }
            }
        }
        
        // Start with root themes
        let root_themes: Vec<_> = themes.iter()
            .filter(|t| t.parent.is_none())
            .collect();
        
        for theme in root_themes {
            add_theme_to_tree(&mut tree, theme, &theme_map, 0);
        }
        
        Ok(CommandOutput::Success(OutputData::Text(tree.build())))
    }
}
```

## Activate/Deactivate Commands

```rust
#[derive(Args)]
pub struct ActivateArgs {
    /// Theme name
    pub name: String,
    
    /// Context type (location, season, event)
    #[arg(long)]
    pub context_type: Option<String>,
    
    /// Context value
    #[arg(long)]
    pub context_value: Option<String>,
    
    /// Force activation even if dependencies missing
    #[arg(short, long)]
    pub force: bool,
}

impl ThemeCommand {
    async fn handle_activate(&self, args: &ActivateArgs) -> Result<CommandOutput> {
        // Build theme context
        let context = if let (Some(ctx_type), Some(ctx_value)) = 
            (&args.context_type, &args.context_value) {
            match ctx_type.as_str() {
                "location" => ThemeContext::Location(ctx_value.clone()),
                "season" => ThemeContext::Season(ctx_value.clone()),
                "event" => ThemeContext::Event(ctx_value.clone()),
                _ => ThemeContext::Custom(format!("{}.{}", ctx_type, ctx_value)),
            }
        } else {
            ThemeContext::Default
        };
        
        // Check if theme exists
        let theme = self.manager.get_theme(&args.name).await?
            .ok_or_else(|| ReedError::ThemeNotFound(args.name.clone()))?;
        
        // Validate dependencies
        if !args.force {
            self.validate_theme_dependencies(&theme).await?;
        }
        
        // Activate theme
        self.manager.activate_theme(&args.name, context).await?;
        
        // Show activation result
        let chain = self.manager.build_theme_chain(&theme).await?;
        
        Ok(CommandOutput::Success(OutputData::Text(format!(
            "Theme '{}' activated\nTheme chain: {}",
            args.name,
            chain.join(" → ")
        ))))
    }
    
    /// Validate theme dependencies
    async fn validate_theme_dependencies(&self, theme: &Theme) -> Result<()> {
        // Check parent exists
        if let Some(ref parent) = theme.parent {
            self.manager.get_theme(parent).await?
                .ok_or_else(|| ReedError::ThemeNotFound(parent.clone()))?;
        }
        
        // Check required plugins
        for plugin in &theme.metadata.requirements.required_plugins {
            if !self.context.plugin_manager.is_plugin_active(plugin).await? {
                return Err(ReedError::MissingDependency(
                    format!("Required plugin '{}' is not active", plugin)
                ));
            }
        }
        
        // Check required features
        for feature in &theme.metadata.requirements.required_features {
            if !self.context.feature_manager.is_feature_enabled(feature).await? {
                return Err(ReedError::MissingDependency(
                    format!("Required feature '{}' is not enabled", feature)
                ));
            }
        }
        
        Ok(())
    }
}
```

## Validate Command

```rust
#[derive(Args)]
pub struct ValidateArgs {
    /// Theme name (or current if not specified)
    pub name: Option<String>,
    
    /// Validate all themes
    #[arg(long)]
    pub all: bool,
    
    /// Check file integrity
    #[arg(long)]
    pub files: bool,
    
    /// Check template syntax
    #[arg(long)]
    pub templates: bool,
    
    /// Check asset references
    #[arg(long)]
    pub assets: bool,
    
    /// Verbose output
    #[arg(short, long)]
    pub verbose: bool,
}

impl ThemeCommand {
    async fn handle_validate(&self, args: &ValidateArgs) -> Result<CommandOutput> {
        let themes = if args.all {
            self.manager.list_themes().await?
        } else if let Some(ref name) = args.name {
            vec![self.manager.get_theme(name).await?
                .ok_or_else(|| ReedError::ThemeNotFound(name.clone()))?]
        } else {
            vec![self.manager.get_current_theme().await?]
        };
        
        let mut results = Vec::new();
        
        for theme in themes {
            let result = self.validate_theme(&theme, args).await?;
            results.push(result);
        }
        
        // Format validation results
        self.format_validation_results(results, args.verbose)
    }
    
    /// Validate single theme
    async fn validate_theme(
        &self,
        theme: &Theme,
        args: &ValidateArgs,
    ) -> Result<ValidationResult> {
        let mut result = ValidationResult {
            theme_name: theme.name.clone(),
            errors: Vec::new(),
            warnings: Vec::new(),
            info: Vec::new(),
        };
        
        // Check theme.toml
        let theme_path = self.context.config.themes_path.join(&theme.name);
        let config_path = theme_path.join("theme.toml");
        
        if !config_path.exists() {
            result.errors.push(ValidationIssue {
                category: "config".to_string(),
                message: "Missing theme.toml".to_string(),
                file: None,
            });
        }
        
        // Validate files
        if args.files || args.all {
            self.validate_theme_files(&theme_path, &mut result).await?;
        }
        
        // Validate templates
        if args.templates || args.all {
            self.validate_theme_templates(&theme, &mut result).await?;
        }
        
        // Validate assets
        if args.assets || args.all {
            self.validate_theme_assets(&theme_path, &mut result).await?;
        }
        
        Ok(result)
    }
    
    /// Validate theme templates
    async fn validate_theme_templates(
        &self,
        theme: &Theme,
        result: &mut ValidationResult,
    ) -> Result<()> {
        let template_dir = self.context.config.themes_path
            .join(&theme.name)
            .join("templates");
        
        if !template_dir.exists() {
            result.warnings.push(ValidationIssue {
                category: "templates".to_string(),
                message: "No templates directory".to_string(),
                file: None,
            });
            return Ok(());
        }
        
        // Find all template files
        for entry in glob::glob(&format!("{}/**/*.tera", template_dir.display()))? {
            if let Ok(path) = entry {
                // Try to parse template
                let content = tokio::fs::read_to_string(&path).await?;
                
                match tera::Tera::one_off(&content, &tera::Context::new(), false) {
                    Ok(_) => {
                        result.info.push(ValidationIssue {
                            category: "templates".to_string(),
                            message: "Valid syntax".to_string(),
                            file: Some(path.display().to_string()),
                        });
                    }
                    Err(e) => {
                        result.errors.push(ValidationIssue {
                            category: "templates".to_string(),
                            message: format!("Syntax error: {}", e),
                            file: Some(path.display().to_string()),
                        });
                    }
                }
            }
        }
        
        Ok(())
    }
}

#[derive(Debug)]
struct ValidationResult {
    theme_name: String,
    errors: Vec<ValidationIssue>,
    warnings: Vec<ValidationIssue>,
    info: Vec<ValidationIssue>,
}

#[derive(Debug)]
struct ValidationIssue {
    category: String,
    message: String,
    file: Option<String>,
}
```

## Theme Chain Command

```rust
#[derive(Args)]
pub struct ChainArgs {
    /// Theme name (or current)
    pub name: Option<String>,
    
    /// Show for specific context
    #[arg(long)]
    pub context: Option<String>,
    
    /// Show file resolution
    #[arg(long)]
    pub files: bool,
    
    /// Specific file to trace
    #[arg(long)]
    pub trace: Option<String>,
}

impl ThemeCommand {
    async fn handle_chain(&self, args: &ChainArgs) -> Result<CommandOutput> {
        // Get theme
        let theme = if let Some(ref name) = args.name {
            self.manager.get_theme(name).await?
                .ok_or_else(|| ReedError::ThemeNotFound(name.clone()))?
        } else {
            self.manager.get_current_theme().await?
        };
        
        // Build chain
        let chain = self.manager.build_theme_chain(&theme).await?;
        
        // Format output
        if let Some(ref file) = args.trace {
            self.trace_file_resolution(file, &theme).await
        } else if args.files {
            self.show_chain_files(&chain).await
        } else {
            self.format_chain_simple(&chain)
        }
    }
    
    /// Trace file resolution through chain
    async fn trace_file_resolution(
        &self,
        file: &str,
        theme: &Theme,
    ) -> Result<CommandOutput> {
        let chain = self.manager.build_theme_chain(theme).await?;
        let mut resolution_steps = Vec::new();
        
        for theme_name in &chain {
            let theme_path = self.context.config.themes_path.join(theme_name);
            let file_path = theme_path.join(file);
            
            let step = FileResolutionStep {
                theme: theme_name.clone(),
                path: file_path.display().to_string(),
                exists: file_path.exists(),
                size: if file_path.exists() {
                    file_path.metadata().ok().map(|m| m.len())
                } else {
                    None
                },
            };
            
            resolution_steps.push(step);
            
            if file_path.exists() {
                break;
            }
        }
        
        // Format resolution trace
        let mut output = String::new();
        output.push_str(&format!("File resolution for: {}\n", file));
        output.push_str(&format!("Theme chain: {}\n\n", chain.join(" → ")));
        
        for (i, step) in resolution_steps.iter().enumerate() {
            let status = if step.exists {
                "✓ FOUND".green()
            } else {
                "✗ Not found".red()
            };
            
            output.push_str(&format!(
                "{:2}. {} {}\n    Path: {}\n",
                i + 1,
                status,
                step.theme,
                step.path
            ));
            
            if let Some(size) = step.size {
                output.push_str(&format!("    Size: {} bytes\n", size));
            }
            
            if step.exists {
                output.push_str(&format!("\n    → File resolved in theme '{}'\n", step.theme));
                break;
            }
        }
        
        Ok(CommandOutput::Success(OutputData::Text(output)))
    }
}

#[derive(Debug)]
struct FileResolutionStep {
    theme: String,
    path: String,
    exists: bool,
    size: Option<u64>,
}
```

## Create Theme Command

```rust
#[derive(Args)]
pub struct CreateArgs {
    /// Theme name
    pub name: String,
    
    /// Parent theme
    #[arg(short, long)]
    pub extends: Option<String>,
    
    /// Theme author
    #[arg(long, default_value = "Unknown")]
    pub author: String,
    
    /// Theme description
    #[arg(long)]
    pub description: Option<String>,
    
    /// Create from template
    #[arg(long)]
    pub template: Option<String>,
    
    /// Initialize with basic files
    #[arg(long)]
    pub init: bool,
}

impl ThemeCommand {
    async fn handle_create(&self, args: &CreateArgs) -> Result<CommandOutput> {
        let theme_path = self.context.config.themes_path.join(&args.name);
        
        // Check if already exists
        if theme_path.exists() {
            return Err(ReedError::AlreadyExists(
                format!("Theme '{}' already exists", args.name)
            ));
        }
        
        // Create theme directory
        tokio::fs::create_dir_all(&theme_path).await?;
        
        // Create theme.toml
        let theme_config = ThemeConfig {
            version: "1.0.0".to_string(),
            author: args.author.clone(),
            description: args.description.clone()
                .unwrap_or_else(|| format!("{} theme", args.name)),
            extends: args.extends.clone(),
            active: Some(false),
            preview: None,
            min_version: Some(env!("CARGO_PKG_VERSION").to_string()),
            plugins: None,
            features: None,
            context: None,
        };
        
        let config_content = toml::to_string_pretty(&theme_config)?;
        let config_path = theme_path.join("theme.toml");
        tokio::fs::write(&config_path, config_content).await?;
        
        // Initialize basic structure if requested
        if args.init {
            self.init_theme_structure(&theme_path).await?;
        }
        
        // Copy from template if specified
        if let Some(ref template) = args.template {
            self.copy_from_template(&theme_path, template).await?;
        }
        
        Ok(CommandOutput::Success(OutputData::Text(format!(
            "Theme '{}' created at {}",
            args.name,
            theme_path.display()
        ))))
    }
    
    /// Initialize basic theme structure
    async fn init_theme_structure(&self, theme_path: &Path) -> Result<()> {
        // Create directories
        for dir in &["templates", "partials", "layouts", "assets/css", "assets/js", "assets/images"] {
            tokio::fs::create_dir_all(theme_path.join(dir)).await?;
        }
        
        // Create base layout
        let base_layout = r#"<!DOCTYPE html>
<html lang="{{ locale | default(value="en") }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}{{ page.title | default(value=site.title) }}{% endblock %}</title>
    {% block head %}{% endblock %}
</head>
<body>
    {% block content %}{% endblock %}
    {% block scripts %}{% endblock %}
</body>
</html>"#;
        
        tokio::fs::write(
            theme_path.join("layouts/base.tera"),
            base_layout
        ).await?;
        
        // Create index template
        let index_template = r#"{% extends "layouts/base.tera" %}

{% block content %}
<h1>{{ page.title }}</h1>
<div class="content">
    {{ page.content | safe }}
</div>
{% endblock %}"#;
        
        tokio::fs::write(
            theme_path.join("templates/index.tera"),
            index_template
        ).await?;
        
        // Create basic CSS
        let basic_css = r#"/* Basic theme styles */
body {
    font-family: system-ui, -apple-system, sans-serif;
    line-height: 1.6;
    color: #333;
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem;
}

h1 {
    color: #2c3e50;
    border-bottom: 2px solid #ecf0f1;
    padding-bottom: 0.5rem;
}"#;
        
        tokio::fs::write(
            theme_path.join("assets/css/style.css"),
            basic_css
        ).await?;
        
        Ok(())
    }
}
```

## Summary

Diese Theme Commands bieten:
- **Theme Management** - List, Activate, Deactivate Themes
- **Context Support** - Location, Season, Event basierte Themes
- **Validation** - Template Syntax und Asset Checking
- **Chain Visualization** - EPC Resolution Tracing
- **Theme Creation** - Scaffolding neuer Themes

Theme Commands machen Theme Development und Management einfach.
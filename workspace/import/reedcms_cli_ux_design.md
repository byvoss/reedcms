# ReedCMS CLI UX Design - KISS Developer Experience

## CLI Philosophy

**"Intuitive Commands, Predictable Responses, Zero Friction"**

KISS-Brain CLI design where every command does exactly what developers expect, with intelligent autocompletion, contextual help, and graceful error handling.

## Command Completion System

### **Intelligent Autocompletion**
```rust
pub struct CLIAutocompletion {
    snippet_registry: SnippetRegistry,
    semantic_name_cache: HashMap<String, String>,  // $name -> UUID
    command_tree: CommandTree,
}

impl CLIAutocompletion {
    pub async fn complete_command(&self, partial: &str) -> Vec<String> {
        let parts: Vec<&str> = partial.trim().split_whitespace().collect();
        
        match parts.as_slice() {
            ["reed"] => {
                // Top-level commands
                vec!["snippet", "theme", "schema", "registry", "plugin", "memory", "serve", "build"]
            },
            
            ["reed", "snippet"] => {
                // Snippet subcommands
                vec!["create", "update", "delete", "get", "list"]
            },
            
            ["reed", "snippet", "create", snippet_type] if snippet_type.starts_with('"') => {
                // After snippet type - suggest semantic names
                vec!["name", "/", "with"]
            },
            
            ["reed", "snippet", "create", _, "name", partial_name] if partial_name.starts_with('$') => {
                // Semantic name completion
                self.complete_semantic_names(partial_name)
            },
            
            ["reed", "snippet", "update", partial_ref] if partial_ref.starts_with('$') => {
                // Update existing snippets - show available semantic names
                self.complete_existing_semantic_names(partial_ref)
            },
            
            _ => vec![]
        }
    }
    
    fn complete_semantic_names(&self, partial: &str) -> Vec<String> {
        // Complete $welcome_hero, $main_nav, etc.
        self.semantic_name_cache.keys()
            .filter(|name| name.starts_with(partial))
            .take(10)
            .cloned()
            .collect()
    }
    
    fn complete_existing_semantic_names(&self, partial: &str) -> Vec<String> {
        // Only show existing semantic names for update/delete
        self.get_existing_snippets()
            .iter()
            .filter(|name| name.starts_with(partial))
            .take(10)
            .cloned()
            .collect()
    }
}

// Shell integration
pub fn generate_bash_completion() -> String {
    r#"
_reed_completions() {
    local cur prev opts
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    
    # Call reed CLI for intelligent completion
    opts=$(reed __complete "${COMP_WORDS[@]:1}")
    COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
    return 0
}

complete -F _reed_completions reed
"#.to_string()
}
```

### **Context-Aware Help System**
```rust
pub struct ContextualHelp {
    current_command: Vec<String>,
    registry: SnippetRegistry,
}

impl ContextualHelp {
    pub fn get_contextual_help(&self, partial_command: &[String]) -> HelpResponse {
        match partial_command {
            ["snippet", "create"] => HelpResponse {
                usage: "reed snippet create \"<snippet-type>\" / name $<semantic-name> / with <field>=<value>",
                description: "Create a new snippet instance",
                examples: vec![
                    "reed snippet create \"hero-banner\" / name $welcome_hero / with title=\"Welcome\"",
                    "reed snippet create \"page\" / name $homepage / with title=\"Home\" slug=\"home\"",
                ],
                available_snippet_types: self.registry.get_all_snippet_types(),
            },
            
            ["snippet", "create", snippet_type] => {
                let fields = self.registry.get_required_fields(snippet_type);
                HelpResponse {
                    usage: format!("with {}", fields.join("=<value> ")),
                    description: format!("Required fields for {} snippet", snippet_type),
                    examples: vec![
                        self.generate_example_for_snippet_type(snippet_type)
                    ],
                    available_fields: fields,
                }
            },
            
            ["theme"] => HelpResponse {
                usage: "reed theme <command>",
                description: "Theme management commands",
                examples: vec![
                    "reed theme create \"corporate\" --base-theme",
                    "reed theme assign scope \"berlin\" theme \"corporate-berlin\"",
                ],
                available_commands: vec!["create", "assign", "list", "scaffold"],
            },
            
            _ => HelpResponse::default()
        }
    }
}
```

## Error Message System

### **Localized Error Messages**
```rust
pub struct ErrorMessageSystem {
    user_locale: String,        // "de_DE", "en_US"
    user_role: UserRole,        // Admin, Developer, Editor
}

#[derive(Debug)]
pub enum UserRole {
    Admin,          // German messages, business context
    Developer,      // English messages, technical details
    Editor,         // German messages, user-friendly
}

impl ErrorMessageSystem {
    pub fn format_error(&self, error: &CLIError) -> String {
        match (&self.user_role, &self.user_locale[..2]) {
            (UserRole::Developer, _) => {
                // Developers always get English with technical details
                self.format_technical_error(error)
            },
            
            (UserRole::Admin, "de") | (UserRole::Editor, "de") => {
                // German users get localized messages
                self.format_german_error(error)
            },
            
            _ => {
                // Fallback to English
                self.format_english_error(error)
            }
        }
    }
    
    fn format_technical_error(&self, error: &CLIError) -> String {
        match error {
            CLIError::SnippetNotFound(name) => {
                format!("Error: Snippet '{}' not found in registry.\n\
                         Troubleshooting:\n\
                         - Check: reed registry list\n\
                         - Verify spelling of snippet name\n\
                         - Registry file: config/snippets.csv", name)
            },
            
            CLIError::ValidationFailed(field, reason) => {
                format!("Validation Error: Field '{}' failed validation.\n\
                         Reason: {}\n\
                         Check snippet definition in registry for field requirements.", field, reason)
            },
            
            CLIError::DatabaseConnectionFailed => {
                format!("Database Error: Cannot connect to PostgreSQL.\n\
                         Troubleshooting:\n\
                         - Check PostgreSQL is running\n\
                         - Verify connection string in config/database.toml\n\
                         - Check network connectivity")
            }
        }
    }
    
    fn format_german_error(&self, error: &CLIError) -> String {
        match error {
            CLIError::SnippetNotFound(name) => {
                format!("Fehler: Snippet '{}' wurde nicht gefunden.\n\
                         Verfügbare Snippets anzeigen: reed registry list", name)
            },
            
            CLIError::ValidationFailed(field, _) => {
                format!("Ungültige Eingabe: Das Feld '{}' enthält fehlerhafte Daten.\n\
                         Bitte überprüfen Sie die Eingabe und versuchen Sie es erneut.", field)
            },
            
            CLIError::DatabaseConnectionFailed => {
                format!("Datenbankfehler: Verbindung zur Datenbank fehlgeschlagen.\n\
                         Bitte kontaktieren Sie Ihren Administrator.")
            }
        }
    }
}
```

### **Error Recovery Suggestions**
```rust
impl ErrorMessageSystem {
    pub fn suggest_recovery(&self, error: &CLIError) -> Vec<String> {
        match error {
            CLIError::SnippetNotFound(name) => {
                // Try to find similar snippet names
                let similar = self.find_similar_snippet_names(name);
                let mut suggestions = vec![
                    "reed registry list  # Show all available snippet types".to_string()
                ];
                
                if !similar.is_empty() {
                    suggestions.push(format!("Did you mean: {}", similar.join(", ")));
                }
                
                suggestions
            },
            
            CLIError::SemanticNameNotFound(name) => {
                vec![
                    "reed snippet list  # Show all existing snippets".to_string(),
                    format!("Create new snippet: reed snippet create \"snippet-type\" / name {}", name),
                ]
            },
            
            CLIError::ValidationFailed(field, _) => {
                vec![
                    format!("reed registry get \"snippet-type\" --show-fields  # Show valid values for {}", field),
                    "Check field requirements in snippet definition".to_string(),
                ]
            }
        }
    }
    
    fn find_similar_snippet_names(&self, input: &str) -> Vec<String> {
        self.registry.get_all_snippet_types()
            .iter()
            .filter(|name| {
                // Simple Levenshtein distance or starts_with matching
                self.is_similar(input, name)
            })
            .take(3)
            .cloned()
            .collect()
    }
}
```

## Interactive vs Script Mode Detection

### **Mode Detection & Response Adaptation**
```rust
pub struct CLIModeDetector {
    is_tty: bool,           // Terminal or pipe/script
    has_color_support: bool, // ANSI color support
    is_interactive: bool,    // Interactive shell session
}

impl CLIModeDetector {
    pub fn new() -> Self {
        CLIModeDetector {
            is_tty: atty::is(atty::Stream::Stdout),
            has_color_support: supports_color::on(supports_color::Stream::Stdout).is_some(),
            is_interactive: std::env::var("REED_INTERACTIVE").is_ok() || atty::is(atty::Stream::Stdin),
        }
    }
    
    pub fn get_output_style(&self) -> OutputStyle {
        match (self.is_tty, self.is_interactive) {
            (true, true) => OutputStyle::Interactive {
                colors: self.has_color_support,
                progress_bars: true,
                confirmations: true,
            },
            (true, false) => OutputStyle::Terminal {
                colors: self.has_color_support,
                progress_bars: false,
                confirmations: false,
            },
            (false, _) => OutputStyle::Script {
                colors: false,
                progress_bars: false,
                toml_output: true,    // TOML, not JSON
            }
        }
    }
}

pub enum OutputStyle {
    Interactive { colors: bool, progress_bars: bool, confirmations: bool },
    Terminal { colors: bool, progress_bars: bool, confirmations: bool },
    Script { colors: bool, progress_bars: bool, toml_output: bool },
}
```

### **Response Formatting by Mode**
```rust
impl CLIFormatter {
    pub fn format_response(&self, response: &CommandResponse, style: &OutputStyle) -> String {
        match style {
            OutputStyle::Interactive { colors: true, .. } => {
                // Rich interactive output with colors and formatting
                format!(
                    "✅ {green}Success:{reset} Created snippet {bold}{name}{reset}\n\
                     📍 Path: {blue}{path}{reset}\n\
                     🔗 URL: {cyan}{url}{reset}",
                    green = "\x1b[32m",
                    reset = "\x1b[0m", 
                    bold = "\x1b[1m",
                    blue = "\x1b[34m",
                    cyan = "\x1b[36m",
                    name = response.snippet_name,
                    path = response.path,
                    url = response.url
                )
            },
            
            OutputStyle::Terminal { colors: false, .. } => {
                // Plain terminal output without colors
                format!(
                    "Success: Created snippet {}\n\
                     Path: {}\n\
                     URL: {}",
                    response.snippet_name,
                    response.path, 
                    response.url
                )
            },
            
            OutputStyle::Script { toml_output: true, .. } => {
                // TOML output for scripts (ReedCMS standard)
                toml::to_string_pretty(&response).unwrap()
            },
            
            _ => {
                // Minimal script output
                format!("SUCCESS:{}", response.snippet_name)
            }
        }
    }
}
```

## Interactive REPL Mode

### **REPL Session Management**
```rust
pub struct REPLSession {
    history: Vec<String>,
    current_context: REPLContext,
    autocomplete: CLIAutocompletion,
    mode_detector: CLIModeDetector,
}

#[derive(Debug, Clone)]
pub struct REPLContext {
    pub current_site: Option<String>,
    pub working_snippet: Option<String>,  // Current snippet being edited
    pub theme_context: Option<String>,    // Current theme scope
}

impl REPLSession {
    pub async fn start_interactive_session(&mut self) -> Result<(), Error> {
        println!("🌾 ReedCMS Interactive Shell");
        println!("Type 'help' for commands, 'exit' to quit");
        
        let mut rl = rustyline::Editor::<()>::new();
        
        // Load command history
        let _ = rl.load_history(".reed_history");
        
        loop {
            let prompt = self.generate_prompt();
            
            match rl.readline(&prompt) {
                Ok(line) => {
                    let line = line.trim();
                    
                    if line.is_empty() {
                        continue;
                    }
                    
                    // Add to history
                    rl.add_history_entry(line);
                    
                    // Handle special REPL commands
                    if self.handle_repl_command(line).await? {
                        continue;
                    }
                    
                    // Execute regular CLI command
                    match self.execute_command(line).await {
                        Ok(response) => {
                            println!("{}", response);
                        },
                        Err(e) => {
                            eprintln!("❌ {}", e);
                            
                            // Show recovery suggestions in interactive mode
                            for suggestion in self.error_system.suggest_recovery(&e) {
                                eprintln!("💡 {}", suggestion);
                            }
                        }
                    }
                },
                
                Err(rustyline::error::ReadlineError::Interrupted) => {
                    println!("^C");
                    continue;
                },
                
                Err(rustyline::error::ReadlineError::Eof) => {
                    println!("Goodbye! 👋");
                    break;
                },
                
                Err(err) => {
                    eprintln!("Error: {:?}", err);
                    break;
                }
            }
        }
        
        // Save command history
        let _ = rl.save_history(".reed_history");
        Ok(())
    }
    
    fn generate_prompt(&self) -> String {
        let site = self.current_context.current_site
            .as_deref()
            .unwrap_or("default");
            
        let snippet_info = if let Some(ref snippet) = self.current_context.working_snippet {
            format!(" ({})", snippet)
        } else {
            String::new()
        };
        
        format!("reed[{}{}]> ", site, snippet_info)
    }
    
    async fn handle_repl_command(&mut self, line: &str) -> Result<bool, Error> {
        match line {
            "exit" | "quit" | "q" => {
                return Ok(false); // Exit REPL
            },
            
            "help" => {
                self.show_repl_help();
                return Ok(true);
            },
            
            "clear" => {
                print!("\x1b[2J\x1b[H"); // Clear screen
                return Ok(true);
            },
            
            line if line.starts_with("use site ") => {
                let site_name = line.strip_prefix("use site ").unwrap();
                self.current_context.current_site = Some(site_name.to_string());
                println!("Switched to site: {}", site_name);
                return Ok(true);
            },
            
            _ => Ok(false) // Not a REPL command
        }
    }
}
```

### **REPL-Specific Commands**
```bash
# Interactive REPL session
$ reed
🌾 ReedCMS Interactive Shell
Type 'help' for commands, 'exit' to quit

reed[default]> help
Available commands:
  snippet create/update/delete/list  - Snippet management
  theme create/assign/list           - Theme operations  
  schema bond/tree/list              - UCG operations
  use site <name>                    - Switch working site
  clear                              - Clear screen
  exit                               - Exit REPL

reed[default]> use site production
Switched to site: production

reed[production]> snippet create "hero-banner" /
               > name $new_hero /
               > with title "Production Hero"
✅ Success: Created snippet $new_hero
📍 Path: production/snippets/hero-banner/new_hero

reed[production]> schema bond parent $homepage child $new_hero
✅ Schema bond created: $homepage -> $new_hero

reed[production]> exit
Goodbye! 👋
```

## Neovim-Style Terminal Dropdown System

### **Das Problem mit traditionellen CLIs**

Aktuelle CLI commands erfordern, dass User die exakten Namen auswendig kennen. Bei `reed snippet create "hero-banner"` muss der User wissen, dass es "hero-banner" heißt und nicht "hero" oder "banner". Das führt zu:

- **Guesswork** - User müssen raten wie snippet types heißen
- **Typos** - `reed snippet create "hero-baner"` → Error
- **Documentation dependency** - Ständig nachschlagen welche Optionen verfügbar sind
- **Schlechte UX für Anfänger** - Steile Lernkurve

### **Die Neovim-Inspired Lösung**

**Interactive Terminal Dropdown** wie in modernen Editoren bringt die UX ins 21. Jahrhundert:

1. **User startet interactive mode:** `reed snippet create --interactive`
2. **Terminal zeigt dropdown:** Alle verfügbaren snippet types mit descriptions
3. **Live fuzzy search:** User tippt "hero" → Liste filtert automatisch zu hero-*
4. **Keyboard navigation:** ↑↓ zum navigieren, Enter zum auswählen, Esc zum abbrechen
5. **Context hints:** Zeigt descriptions, current values, recently used items

### **Warum das revolutionär für CLI UX ist**

**Für ReedCMS Anfänger:**
- **Discoverability** - Sehen alle verfügbaren Optionen ohne Documentation
- **Learning by doing** - Descriptions erklären was jeder snippet type macht
- **Error prevention** - Können nur valid selections treffen
- **Confidence building** - Kein "trial and error" mehr

**Für ReedCMS Experten:**
- **Speed** - Fuzzy search macht selection super schnell (2-3 keystrokes)
- **Muscle memory** - Recent/frequent items erscheinen automatisch oben
- **Backward compatibility** - Können weiterhin old-school CLI nutzen
- **Context awareness** - System lernt User preferences

**Für alle User:**
- **Modern UX** - Konsistent mit VSCode, Neovim, modern terminals
- **Visual appeal** - Fühlt sich nicht wie 90er CLI an
- **Reduced cognitive load** - Weniger Dinge zum merken
- **Faster workflows** - Weniger hin-und-her mit documentation

### **Praktischer Workflow Vergleich**

**Traditional CLI (error-prone):**
```
User: reed snippet create "hero-baner"  # Typo!
CLI:  Error: Snippet type 'hero-baner' not found. Use 'reed registry list' to see available types.
User: reed registry list  # Extra step!
CLI:  [shows long list]
User: reed snippet create "hero-banner"  # Finally correct
```

**Interactive Dropdown (foolproof):**
```
User: reed snippet create --interactive
CLI:  [Shows filtered dropdown as user types]
User: he[ro]  # Types partial match
CLI:  [Instantly filters to hero-banner, hero-section]
User: [Enter]  # Selects highlighted option
CLI:  [Continues with field prompts]
```

### **Smart Context Features**

**Recently Used Priority:**
- Snippet types die User häufig nutzt erscheinen oben
- Reduziert selection time für common workflows
- Lernt User preferences over time

**Related Suggestions:**
- Wenn User an "hero-banner" arbeitet, schlägt system "content-section" vor
- Context-aware workflow acceleration
- Intelligent content graph connections

**Progressive Disclosure:**  
- Zeigt nur top 10 items initially, "...5 more" für overflow
- Fuzzy search reduziert clutter instantly
- Clean, focused interface auch bei vielen snippet types

## ReedCMS Whois Command - Entity Intelligence

### **Das Admin-Debug Tool für ReedCMS**

**Problem:** Admins sehen UUIDs in logs, error messages, oder database und wissen nicht was dahinter steckt. Bei `550e8400-e29b-41d4-a716-446655440000` ist völlig unklar ob das ein snippet, user, oder session ist.

**Lösung:** **Universal Whois Command** zeigt alles was ReedCMS über eine Entity weiß:

```bash
reed whois 550e8400-e29b-41d4-a716-446655440000
```

### **Was Whois alles zeigt**

**Entity Identification:**
- Entity type (snippet, user, theme, plugin, etc.)
- Semantic name falls vorhanden ($welcome_hero)
- Creation timestamp und last modified
- Current status (active, deleted, draft, etc.)

**Content Information:**
- Snippet type und registry definition
- All field values (respecting permissions)
- Associated translations wenn vorhanden
- Content size und complexity metrics

**Relationship Graph:**
- Parent entities (welche pages nutzen dieses snippet)
- Child entities (welche sub-snippets gehören dazu)
- UCG path und hierarchy position
- Schema bonds und associations

**Performance & Usage Statistics:**
- View count und last accessed
- Rendering performance metrics
- Cache hit/miss ratios
- Memory usage für diese entity

**Administrative Metadata:**
- Created by welcher user
- Last modified by wem
- Permission settings
- Backup status und recovery info

### **Praktische Use Cases**

**Debugging Production Issues:**
```bash
# Error log zeigt: "Failed to render entity 550e8400..."
$ reed whois 550e8400-e29b-41d4-a716-446655440000

🔍 Entity: $welcome_hero (hero-banner)
├── Status: Active
├── Created: 2025-01-15 14:30:22 by admin@site.com
├── Last Modified: 2025-01-18 09:15:33 by editor@site.com
├── Content Size: 2.3KB
└── Performance: Avg render time 1.2ms, Cache hit rate 94%

📍 Location & Usage:
├── Used on 3 pages: /home, /about, /contact
├── UCG Path: content.1.1 
├── Parent: $homepage (page)
└── Children: None

⚠️  Issues Detected:
├── Missing alt text for background image
└── Title length 89 chars (SEO warning: >60)
```

**Content Audit für Redakteure:**
```bash
$ reed whois $old_product_banner

🔍 Entity: $old_product_banner (hero-banner)
├── Status: Unused (no active references)
├── Created: 2024-08-15 (6 months ago)
├── Last Used: 2024-12-10 (1 month ago)
└── Safe to delete: ✅ Yes

📊 Usage History:
├── Peak usage: November 2024 (1,200 views)
├── Last 30 days: 0 views
└── Replacement: $new_product_hero (suggested)
```

**Performance Investigation:**
```bash
$ reed whois $slow_gallery_snippet

🔍 Entity: $slow_gallery_snippet (image-gallery)
├── Status: Active ⚠️ Performance Issues
├── Performance: Avg render time 245ms (vs 15ms average)
├── Memory Usage: 12MB (vs 2MB average)
└── Cache Efficiency: 23% hit rate (vs 85% site average)

🐌 Performance Issues:
├── Contains 47 high-resolution images (5MB total)
├── No lazy loading implemented
├── Missing image optimization
└── Suggestion: Enable image compression in registry
```

### **Whois Command Implementation**

**Smart Entity Detection:**
```rust
impl WhoisCommand {
    pub async fn analyze_entity(&self, id: &str) -> Result<EntityProfile, Error> {
        // 1. Detect entity type from ID or semantic name
        let entity = self.resolve_entity(id).await?;
        
        // 2. Gather comprehensive information
        let profile = EntityProfile {
            basic_info: self.get_basic_info(&entity).await?,
            content_data: self.get_content_data(&entity).await?,
            relationships: self.get_relationships(&entity).await?,
            performance_stats: self.get_performance_stats(&entity).await?,
            admin_metadata: self.get_admin_metadata(&entity).await?,
            health_check: self.run_health_check(&entity).await?,
        };
        
        Ok(profile)
    }
    
    async fn resolve_entity(&self, id: &str) -> Result<ResolvedEntity, Error> {
        // Handle different input formats
        if id.starts_with('
```rust
pub struct TerminalDropdown {
    items: Vec<SelectionItem>,
    selected_index: usize,
    filter_text: String,
    filtered_items: Vec<usize>,  // Indices of filtered items
}

#[derive(Debug, Clone)]
pub struct SelectionItem {
    pub display_text: String,
    pub value: String,
    pub description: Option<String>,
    pub category: Option<String>,
}

impl TerminalDropdown {
    pub async fn show_selection_menu(
        &mut self, 
        prompt: &str, 
        items: Vec<SelectionItem>
    ) -> Result<Option<String>, Error> {
        self.items = items;
        self.filter_items();
        
        // Enable raw terminal mode
        crossterm::terminal::enable_raw_mode()?;
        
        let mut stdout = std::io::stdout();
        crossterm::execute!(stdout, crossterm::cursor::Hide)?;
        
        loop {
            // Clear and redraw
            self.render_dropdown(&mut stdout, prompt)?;
            
            // Handle key input
            if let crossterm::event::Event::Key(key) = crossterm::event::read()? {
                match key.code {
                    KeyCode::Enter => {
                        let selected = self.get_selected_value();
                        self.cleanup_terminal(&mut stdout)?;
                        return Ok(selected);
                    },
                    
                    KeyCode::Esc => {
                        self.cleanup_terminal(&mut stdout)?;
                        return Ok(None);
                    },
                    
                    KeyCode::Up => {
                        self.move_selection(-1);
                    },
                    
                    KeyCode::Down => {
                        self.move_selection(1);
                    },
                    
                    KeyCode::Char(c) => {
                        self.filter_text.push(c);
                        self.filter_items();
                        self.selected_index = 0;
                    },
                    
                    KeyCode::Backspace => {
                        self.filter_text.pop();
                        self.filter_items();
                        self.selected_index = 0;
                    },
                    
                    _ => {}
                }
            }
        }
    }
    
    fn render_dropdown(&self, stdout: &mut std::io::Stdout, prompt: &str) -> Result<(), Error> {
        // Clear screen area
        crossterm::execute!(
            stdout,
            crossterm::terminal::Clear(crossterm::terminal::ClearType::FromCursorDown)
        )?;
        
        // Show prompt and filter
        println!("{}", prompt);
        println!("Filter: {}_", self.filter_text);
        println!("Use ↑↓ to navigate, Enter to select, Esc to cancel");
        println!("");
        
        // Show filtered items with selection highlight
        for (display_index, &item_index) in self.filtered_items.iter().enumerate() {
            let item = &self.items[item_index];
            
            if display_index == self.selected_index {
                // Highlighted selection (reverse colors)
                print!("\x1b[7m> {}\x1b[0m", item.display_text);
            } else {
                print!("  {}", item.display_text);
            }
            
            // Show description if available
            if let Some(ref desc) = item.description {
                print!(" \x1b[90m({})\x1b[0m", desc);
            }
            
            println!("");
            
            // Limit visible items to prevent screen overflow
            if display_index >= 10 {
                println!("  ... ({} more items)", self.filtered_items.len() - 11);
                break;
            }
        }
        
        // Move cursor back to filter line
        crossterm::execute!(
            stdout,
            crossterm::cursor::MoveTo(8 + self.filter_text.len() as u16, 1)
        )?;
        
        stdout.flush()?;
        Ok(())
    }
    
    fn filter_items(&mut self) {
        if self.filter_text.is_empty() {
            self.filtered_items = (0..self.items.len()).collect();
        } else {
            self.filtered_items = self.items
                .iter()
                .enumerate()
                .filter(|(_, item)| {
                    item.display_text.to_lowercase().contains(&self.filter_text.to_lowercase()) ||
                    item.description.as_ref().map_or(false, |d| d.to_lowercase().contains(&self.filter_text.to_lowercase()))
                })
                .map(|(i, _)| i)
                .collect();
        }
    }
}
```

### **CLI Integration with Dropdown Menus**
```rust
impl CLIApplication {
    pub async fn create_snippet_interactive(&mut self) -> Result<(), Error> {
        let mut dropdown = TerminalDropdown::new();
        
        // 1. Select snippet type with dropdown
        let snippet_types = self.registry.get_all_snippet_types();
        let items: Vec<SelectionItem> = snippet_types.into_iter().map(|st| {
            let description = self.registry.get_snippet_description(&st.name);
            SelectionItem {
                display_text: st.name.clone(),
                value: st.name.clone(),
                description: Some(description),
                category: Some(st.category),
            }
        }).collect();
        
        let selected_type = dropdown.show_selection_menu(
            "🌾 Select snippet type:",
            items
        ).await?;
        
        let snippet_type = match selected_type {
            Some(t) => t,
            None => return Ok(()), // User cancelled
        };
        
        // 2. Interactive field entry with validation
        let required_fields = self.registry.get_required_fields(&snippet_type);
        let mut field_values = HashMap::new();
        
        for field in required_fields {
            let value = self.prompt_for_field(&snippet_type, &field).await?;
            field_values.insert(field, value);
        }
        
        // 3. Generate semantic name suggestion
        let suggested_name = self.suggest_semantic_name(&snippet_type, &field_values);
        println!("Suggested semantic name: ${}", suggested_name);
        
        let semantic_name = inquire::Text::new("Semantic name:")
            .with_initial_value(&format!("${}", suggested_name))
            .prompt()?;
        
        // 4. Create snippet
        self.create_snippet(&snippet_type, &semantic_name, field_values).await?;
        
        println!("✅ Created snippet {} ({})", semantic_name, snippet_type);
        Ok(())
    }
    
    pub async fn update_snippet_interactive(&mut self, semantic_name: &str) -> Result<(), Error> {
        // Load existing snippet
        let snippet = self.get_snippet_by_semantic_name(semantic_name).await?;
        
        // Show current fields with dropdown selection
        let mut dropdown = TerminalDropdown::new();
        let field_items: Vec<SelectionItem> = snippet.fields.iter().map(|(name, value)| {
            SelectionItem {
                display_text: name.clone(),
                value: name.clone(),
                description: Some(format!("Current: {}", value)),
                category: None,
            }
        }).collect();
        
        let field_to_update = dropdown.show_selection_menu(
            &format!("🔧 Select field to update in {}:", semantic_name),
            field_items
        ).await?;
        
        if let Some(field_name) = field_to_update {
            let current_value = snippet.fields.get(&field_name).unwrap();
            let new_value = inquire::Text::new(&format!("New value for '{}':", field_name))
                .with_initial_value(current_value)
                .prompt()?;
            
            self.update_snippet_field(semantic_name, &field_name, &new_value).await?;
            println!("✅ Updated {}.{} = {}", semantic_name, field_name, new_value);
        }
        
        Ok(())
    }
}
```

### **Usage Examples with Dropdown**
```bash
# Interactive snippet creation
$ reed snippet create --interactive

🌾 Select snippet type:
Filter: _
Use ↑↓ to navigate, Enter to select, Esc to cancel

> hero-banner          (Hero section with title and background)
  product-card         (Product showcase with image and price)
  content-section      (Rich text content block)
  navigation-item      (Menu navigation item)
  contact-form         (Contact form with validation)
  
# User types "hero" to filter
🌾 Select snippet type:
Filter: hero_

> hero-banner          (Hero section with title and background)
  hero-section         (Alternative hero layout)

# After selection, interactive field prompts
✅ Selected: hero-banner

Title (required): Welcome to ReedCMS
Subtitle (optional): Modern Content Management
Background URL (optional): /images/hero-bg.jpg

Suggested semantic name: $welcome_hero
Semantic name: $welcome_hero

✅ Created snippet $welcome_hero (hero-banner)

# Interactive update with field selection
$ reed snippet update $welcome_hero --interactive

🔧 Select field to update in $welcome_hero:
Filter: _

> title                (Current: Welcome to ReedCMS)
  subtitle             (Current: Modern Content Management)  
  background_url       (Current: /images/hero-bg.jpg)

# User selects title
New value for 'title': Welcome to the Future of CMS

✅ Updated $welcome_hero.title = Welcome to the Future of CMS
```

### **Smart Suggestions in Dropdown**
```rust
impl TerminalDropdown {
    fn add_smart_suggestions(&mut self, context: &CLIContext) {
        // Add recently used items at top
        let recent_items = context.get_recent_selections();
        for item in recent_items {
            self.items.insert(0, SelectionItem {
                display_text: format!("🕒 {}", item.name),
                value: item.value,
                description: Some("Recently used".to_string()),
                category: Some("recent".to_string()),
            });
        }
        
        // Add related suggestions
        if let Some(current_snippet) = &context.working_snippet {
            let related = self.get_related_snippet_types(current_snippet);
            for related_type in related {
                self.items.push(SelectionItem {
                    display_text: format!("🔗 {}", related_type.name),
                    value: related_type.name,
                    description: Some("Related to current snippet".to_string()),
                    category: Some("related".to_string()),
                });
            }
        }
    }
}
```
```bash
# Interactive output (human-readable)
$ reed snippet list
┌─────────────────┬──────────────┬─────────────┐
│ Name            │ Type         │ Created     │
├─────────────────┼──────────────┼─────────────┤
│ $homepage       │ page         │ 2025-01-15  │
│ $welcome_hero   │ hero-banner  │ 2025-01-15  │
└─────────────────┴──────────────┴─────────────┘

# Script output (TOML format - ReedCMS standard)
$ reed snippet list --format toml
[[snippets]]
name = "homepage"
type = "page" 
created = "2025-01-15"
semantic_name = "$homepage"

[[snippets]]
name = "welcome_hero"
type = "hero-banner"
created = "2025-01-15" 
semantic_name = "$welcome_hero"

# Simple script output (minimal)
$ reed snippet list --format simple
$homepage
$welcome_hero
```

### **Fast Command Execution**
```rust
pub struct CLIPerformance {
    command_cache: HashMap<String, CachedResult>,
    background_preloader: BackgroundPreloader,
}

impl CLIPerformance {
    pub async fn execute_command_fast(&self, command: &str) -> Result<String, Error> {
        // 1. Check command cache first
        if let Some(cached) = self.command_cache.get(command) {
            if !cached.is_expired() {
                return Ok(cached.result.clone());
            }
        }
        
        // 2. Execute command with timeout
        let result = tokio::time::timeout(
            Duration::from_secs(10), 
            self.execute_command_impl(command)
        ).await??;
        
        // 3. Cache result for repeated commands
        self.cache_result(command, &result);
        
        Ok(result)
    }
    
    pub async fn preload_common_data(&mut self) -> Result<(), Error> {
        // Preload frequently accessed data in background
        tokio::spawn(async move {
            let _ = self.preload_snippet_registry().await;
            let _ = self.preload_semantic_names().await;
            let _ = self.preload_theme_list().await;
        });
        
        Ok(())
    }
}
```

### **CLI Startup Optimization**
```rust
// Fast CLI startup - lazy loading
impl CLIApplication {
    pub async fn new() -> Result<Self, Error> {
        // Only load essential components for startup
        let essential_config = load_essential_config().await?;
        
        Ok(CLIApplication {
            config: essential_config,
            registry: None,          // Lazy load when needed
            database: None,          // Lazy load when needed
            redis: None,             // Lazy load when needed
            preload_started: false,
        })
    }
    
    pub async fn ensure_components_loaded(&mut self) -> Result<(), Error> {
        if !self.preload_started {
            // Start background preloading
            self.start_background_preload().await?;
            self.preload_started = true;
        }
        
        // Load components on-demand
        if self.registry.is_none() {
            self.registry = Some(SnippetRegistry::load().await?);
        }
        
        Ok(())
    }
}
```

## User Experience Examples

### **Command Flow Examples**
```bash
# Example 1: New user creates first snippet
$ reed snippet create "hero-banner"
Error: Missing required parameter 'name'. 

💡 Usage: reed snippet create "hero-banner" / name $<semantic-name>
💡 Example: reed snippet create "hero-banner" / name $welcome_hero

$ reed snippet create "hero-banner" / name $welcome_hero
Error: Missing required fields for 'hero-banner': title

💡 Required fields: title, subtitle (optional)
💡 Example: reed snippet create "hero-banner" / name $welcome_hero / with title="Welcome"

$ reed snippet create "hero-banner" / name $welcome_hero / with title="Welcome to ReedCMS"
✅ Success: Created snippet $welcome_hero (hero-banner)
📍 UUID: 550e8400-e29b-41d4-a716-446655440000
🔗 Edit: reed snippet update $welcome_hero

# Example 2: Tab completion in action  
$ reed snippet update $wel<TAB>
$welcome_hero

$ reed snippet update $welcome_hero / set field "title" to <TAB>
<shows current value for context>

# Example 3: Helpful error recovery
$ reed snippet create "hero-baner"  # Typo
Error: Snippet type 'hero-baner' not found.

💡 Did you mean: hero-banner, hero-section
💡 Available types: reed registry list
```

This CLI UX design ensures developers have a smooth, predictable experience with intelligent completion, contextual help, and graceful error handling while maintaining KISS principles.
) {
            // Semantic name: $welcome_hero
            self.resolve_by_semantic_name(id).await
        } else if id.len() == 36 && id.contains('-') {
            // UUID: 550e8400-e29b-41d4-a716-446655440000
            self.resolve_by_uuid(id).await
        } else if id.contains('.') {
            // UCG path: content.1.1
            self.resolve_by_ucg_path(id).await
        } else {
            // Try intelligent search
            self.resolve_by_fuzzy_search(id).await
        }
    }
}
```

**Health Check Integration:**
```bash
$ reed whois $contact_form --health-check

🔍 Entity: $contact_form (contact-form)
├── Status: Active
└── Health Score: 🟡 Warning (73/100)

🩺 Health Check Results:
├── ✅ Schema validation passed
├── ✅ All required fields present  
├── ⚠️  Email validation regex outdated
├── ❌ CSRF protection not enabled
└── ⚠️  Form submission rate: 0.3% (low engagement)

💡 Recommendations:
├── Update email regex in registry
├── Enable CSRF in security settings
└── Consider A/B testing form layout
```

**Batch Whois für Multiple Entities:**
```bash
$ reed whois --batch $hero_banner $main_nav $footer_content

Entity Summary (3 items):
┌─────────────────┬──────────────┬────────┬─────────────┐
│ Name            │ Type         │ Status │ Health      │
├─────────────────┼──────────────┼────────┼─────────────┤
│ $hero_banner    │ hero-banner  │ Active │ ✅ Good     │
│ $main_nav       │ navigation   │ Active │ ⚠️  Warning │
│ $footer_content │ content-sect │ Draft  │ ❌ Issues   │
└─────────────────┴──────────────┴────────┴─────────────┘

Use 'reed whois <name> --detailed' for full analysis
```

### **Integration mit Admin Panel**

**One-Click Debugging:**
- Admin Panel error → Click UUID → Opens whois analysis
- Performance dashboard → Click slow entity → Shows whois details
- Content audit → Click unused entity → Shows whois mit delete recommendation

**Contextual Actions:**
- Whois zeigt "Delete safe" → Button to delete directly
- Whois zeigt performance issues → Link to optimization tools
- Whois zeigt missing fields → Link to edit form

**EPC Chain Analysis mit --epc Flag:**
```bash
$ reed whois $welcome_hero --epc

🔍 Entity: $welcome_hero (hero-banner)
└── Basic Info: Active, created 2025-01-15

🔗 EPC Chain Analysis:
┌─ Root Level
│
├─ content.1 ($homepage - page)
│  ├─ Title: "Welcome to Our Site"
│  ├─ Created: 2025-01-10 by admin@site.com
│  ├─ Status: Published
│  └─ URL: /home
│
├─── content.1.1 ($welcome_hero - hero-banner) ← YOU ARE HERE
│    ├─ Title: "Welcome to the Future"
│    ├─ Weight: 0 (first position)
│    ├─ Created: 2025-01-15 by editor@site.com
│    └─ Rendered on: /home (primary), /about (secondary)
│
├───── content.1.1.1 ($hero_cta_button - call-to-action)
│      ├─ Text: "Get Started Now"
│      ├─ Weight: 0
│      └─ Links to: $contact_form
│
├─── content.1.2 ($main_content - content-section)
│    ├─ Title: "About Our Services"
│    ├─ Weight: 10 (second position)
│    └─ Contains: 3 child snippets
│
└─── content.1.3 ($footer_cta - call-to-action)
     ├─ Text: "Contact Us Today"
     ├─ Weight: 20 (third position)
     └─ Status: Draft

📍 Path Summary:
├─ Full UCG Path: content.1.1
├─ Hierarchy Depth: 2 levels deep
├─ Position in Parent: 1st of 3 siblings
├─ Total Children: 1 child (call-to-action button)
└─ Context: Homepage hero section

🎯 Theme Resolution Chain:
├─ Base Theme: corporate
├─ Context Scope: berlin.christmas
├─ EPC File Resolution:
│  ├─ 1. themes/corporate.berlin.christmas/hero-banner/hero-banner.css ← ACTIVE
│  ├─ 2. themes/corporate.berlin/hero-banner/hero-banner.css
│  ├─ 3. themes/corporate/hero-banner/hero-banner.css
│  └─ 4. snippets/hero-banner/hero-banner.css (fallback)
└─ Template Used: themes/corporate.berlin.christmas/hero-banner/hero-banner.tera
```

**Detailed EPC Navigation:**
```bash
$ reed whois content.1.2 --epc --siblings

🔍 Entity: $main_content (content-section)
└── UCG Path: content.1.2

👥 Sibling Analysis:
├─ Previous: content.1.1 ($welcome_hero - hero-banner)
│  └─ Weight: 0 → Renders BEFORE current entity
├─ Current:  content.1.2 ($main_content - content-section) ← YOU ARE HERE
│  └─ Weight: 10 → Middle position in layout
└─ Next:     content.1.3 ($footer_cta - call-to-action)
   └─ Weight: 20 → Renders AFTER current entity

🏗️ Parent Context:
├─ Parent: content.1 ($homepage - page)
├─ Total Siblings: 3 entities
├─ Rendering Order: hero → content → footer
└─ Page Layout: Top-to-bottom flow

⚡ Performance Impact:
├─ Load Order: 2nd of 3 (after hero)
├─ Lazy Loading: ✅ Enabled (below fold)
├─ Cache Status: ✅ Cached
└─ Bundle: Included in main.css (12KB)
```

**Cross-Reference Analysis:**
```bash
$ reed whois $welcome_hero --epc --references

🔍 Entity: $welcome_hero (hero-banner)

🔗 Where This Entity is Referenced:
├─ Primary Usage:
│  ├─ Page: $homepage (/home)
│  │  ├─ UCG Path: content.1.1
│  │  ├─ Position: Hero section (top of page)
│  │  └─ Visits: 1,247 views last 30 days
│  │
│  └─ Page: $about_page (/about)
│     ├─ UCG Path: about.1.1 
│     ├─ Position: Header section
│     └─ Visits: 432 views last 30 days
│
├─ Theme Overrides:
│  ├─ Corporate Theme: Base template
│  ├─ Berlin Context: Color overrides (--primary: #ff6b6b)
│  └─ Christmas Scope: Background pattern + border
│
├─ Template Dependencies:
│  ├─ Uses Component: ReedSnippet base class
│  ├─ CSS Dependencies: hero-banner.css (4.2KB)
│  ├─ JS Dependencies: hero-banner.js (1.8KB)
│  └─ Image Assets: background.jpg (245KB)
│
└─ Search Index:
   ├─ Indexed Words: "welcome", "future", "modern", "cms"
   ├─ Search Ranking: Position 3 for "welcome"
   └─ Click-through Rate: 12.3% from search

📊 Impact Analysis:
├─ Remove Impact: 🔴 HIGH - Used on 2 major pages
├─ Dependencies: 1 child entity would become orphaned
├─ SEO Impact: 🟡 MEDIUM - Contains indexed keywords
└─ User Experience: 🔴 CRITICAL - Primary homepage element
```

**EPC Chain Debugging:**
```bash
$ reed whois $broken_snippet --epc --debug

🔍 Entity: $broken_snippet (custom-widget)
└── Status: ❌ ERROR - Rendering Failed

🔗 EPC Chain Debug Analysis:
├─ UCG Path: content.1.2.3 
├─ Parent: content.1.2 ($services_section) ✅ EXISTS
├─ Expected Position: 3rd child
└─ Actual Position: ❌ MISSING from parent children list

🐛 Issues Detected:
├─ ❌ UCG Integrity Error: 
│  ├─ Entity exists in entities table
│  ├─ Association missing from associations table
│  └─ Redis cache inconsistent with PostgreSQL
│
├─ ❌ Template Resolution Failed:
│  ├─ Looking for: themes/corporate/custom-widget/custom-widget.tera
│  ├─ File Status: ❌ NOT FOUND
│  └─ Fallback: snippets/custom-widget/custom-widget.tera ❌ NOT FOUND
│
└─ ❌ Registry Definition Missing:
   ├─ Snippet Type: custom-widget
   ├─ Registry Status: ❌ NOT REGISTERED
   └─ Suggestion: Run 'reed registry define custom-widget'

🛠️ Repair Suggestions:
├─ 1. Fix UCG Association: reed schema bond parent $services_section child $broken_snippet
├─ 2. Create Template: reed generate template custom-widget
├─ 3. Register Type: reed registry define custom-widget
└─ 4. Rebuild Cache: reed memory rebuild
```

Diese EPC Chain Analysis macht **UCG debugging** extrem effektiv - Admins sehen sofort wo eine Entity in der Hierarchie steht, welche Siblings sie hat, und wie die Theme-Resolution funktioniert.
```rust
pub struct TerminalDropdown {
    items: Vec<SelectionItem>,
    selected_index: usize,
    filter_text: String,
    filtered_items: Vec<usize>,  // Indices of filtered items
}

#[derive(Debug, Clone)]
pub struct SelectionItem {
    pub display_text: String,
    pub value: String,
    pub description: Option<String>,
    pub category: Option<String>,
}

impl TerminalDropdown {
    pub async fn show_selection_menu(
        &mut self, 
        prompt: &str, 
        items: Vec<SelectionItem>
    ) -> Result<Option<String>, Error> {
        self.items = items;
        self.filter_items();
        
        // Enable raw terminal mode
        crossterm::terminal::enable_raw_mode()?;
        
        let mut stdout = std::io::stdout();
        crossterm::execute!(stdout, crossterm::cursor::Hide)?;
        
        loop {
            // Clear and redraw
            self.render_dropdown(&mut stdout, prompt)?;
            
            // Handle key input
            if let crossterm::event::Event::Key(key) = crossterm::event::read()? {
                match key.code {
                    KeyCode::Enter => {
                        let selected = self.get_selected_value();
                        self.cleanup_terminal(&mut stdout)?;
                        return Ok(selected);
                    },
                    
                    KeyCode::Esc => {
                        self.cleanup_terminal(&mut stdout)?;
                        return Ok(None);
                    },
                    
                    KeyCode::Up => {
                        self.move_selection(-1);
                    },
                    
                    KeyCode::Down => {
                        self.move_selection(1);
                    },
                    
                    KeyCode::Char(c) => {
                        self.filter_text.push(c);
                        self.filter_items();
                        self.selected_index = 0;
                    },
                    
                    KeyCode::Backspace => {
                        self.filter_text.pop();
                        self.filter_items();
                        self.selected_index = 0;
                    },
                    
                    _ => {}
                }
            }
        }
    }
    
    fn render_dropdown(&self, stdout: &mut std::io::Stdout, prompt: &str) -> Result<(), Error> {
        // Clear screen area
        crossterm::execute!(
            stdout,
            crossterm::terminal::Clear(crossterm::terminal::ClearType::FromCursorDown)
        )?;
        
        // Show prompt and filter
        println!("{}", prompt);
        println!("Filter: {}_", self.filter_text);
        println!("Use ↑↓ to navigate, Enter to select, Esc to cancel");
        println!("");
        
        // Show filtered items with selection highlight
        for (display_index, &item_index) in self.filtered_items.iter().enumerate() {
            let item = &self.items[item_index];
            
            if display_index == self.selected_index {
                // Highlighted selection (reverse colors)
                print!("\x1b[7m> {}\x1b[0m", item.display_text);
            } else {
                print!("  {}", item.display_text);
            }
            
            // Show description if available
            if let Some(ref desc) = item.description {
                print!(" \x1b[90m({})\x1b[0m", desc);
            }
            
            println!("");
            
            // Limit visible items to prevent screen overflow
            if display_index >= 10 {
                println!("  ... ({} more items)", self.filtered_items.len() - 11);
                break;
            }
        }
        
        // Move cursor back to filter line
        crossterm::execute!(
            stdout,
            crossterm::cursor::MoveTo(8 + self.filter_text.len() as u16, 1)
        )?;
        
        stdout.flush()?;
        Ok(())
    }
    
    fn filter_items(&mut self) {
        if self.filter_text.is_empty() {
            self.filtered_items = (0..self.items.len()).collect();
        } else {
            self.filtered_items = self.items
                .iter()
                .enumerate()
                .filter(|(_, item)| {
                    item.display_text.to_lowercase().contains(&self.filter_text.to_lowercase()) ||
                    item.description.as_ref().map_or(false, |d| d.to_lowercase().contains(&self.filter_text.to_lowercase()))
                })
                .map(|(i, _)| i)
                .collect();
        }
    }
}
```

### **CLI Integration with Dropdown Menus**
```rust
impl CLIApplication {
    pub async fn create_snippet_interactive(&mut self) -> Result<(), Error> {
        let mut dropdown = TerminalDropdown::new();
        
        // 1. Select snippet type with dropdown
        let snippet_types = self.registry.get_all_snippet_types();
        let items: Vec<SelectionItem> = snippet_types.into_iter().map(|st| {
            let description = self.registry.get_snippet_description(&st.name);
            SelectionItem {
                display_text: st.name.clone(),
                value: st.name.clone(),
                description: Some(description),
                category: Some(st.category),
            }
        }).collect();
        
        let selected_type = dropdown.show_selection_menu(
            "🌾 Select snippet type:",
            items
        ).await?;
        
        let snippet_type = match selected_type {
            Some(t) => t,
            None => return Ok(()), // User cancelled
        };
        
        // 2. Interactive field entry with validation
        let required_fields = self.registry.get_required_fields(&snippet_type);
        let mut field_values = HashMap::new();
        
        for field in required_fields {
            let value = self.prompt_for_field(&snippet_type, &field).await?;
            field_values.insert(field, value);
        }
        
        // 3. Generate semantic name suggestion
        let suggested_name = self.suggest_semantic_name(&snippet_type, &field_values);
        println!("Suggested semantic name: ${}", suggested_name);
        
        let semantic_name = inquire::Text::new("Semantic name:")
            .with_initial_value(&format!("${}", suggested_name))
            .prompt()?;
        
        // 4. Create snippet
        self.create_snippet(&snippet_type, &semantic_name, field_values).await?;
        
        println!("✅ Created snippet {} ({})", semantic_name, snippet_type);
        Ok(())
    }
    
    pub async fn update_snippet_interactive(&mut self, semantic_name: &str) -> Result<(), Error> {
        // Load existing snippet
        let snippet = self.get_snippet_by_semantic_name(semantic_name).await?;
        
        // Show current fields with dropdown selection
        let mut dropdown = TerminalDropdown::new();
        let field_items: Vec<SelectionItem> = snippet.fields.iter().map(|(name, value)| {
            SelectionItem {
                display_text: name.clone(),
                value: name.clone(),
                description: Some(format!("Current: {}", value)),
                category: None,
            }
        }).collect();
        
        let field_to_update = dropdown.show_selection_menu(
            &format!("🔧 Select field to update in {}:", semantic_name),
            field_items
        ).await?;
        
        if let Some(field_name) = field_to_update {
            let current_value = snippet.fields.get(&field_name).unwrap();
            let new_value = inquire::Text::new(&format!("New value for '{}':", field_name))
                .with_initial_value(current_value)
                .prompt()?;
            
            self.update_snippet_field(semantic_name, &field_name, &new_value).await?;
            println!("✅ Updated {}.{} = {}", semantic_name, field_name, new_value);
        }
        
        Ok(())
    }
}
```

### **Usage Examples with Dropdown**
```bash
# Interactive snippet creation
$ reed snippet create --interactive

🌾 Select snippet type:
Filter: _
Use ↑↓ to navigate, Enter to select, Esc to cancel

> hero-banner          (Hero section with title and background)
  product-card         (Product showcase with image and price)
  content-section      (Rich text content block)
  navigation-item      (Menu navigation item)
  contact-form         (Contact form with validation)
  
# User types "hero" to filter
🌾 Select snippet type:
Filter: hero_

> hero-banner          (Hero section with title and background)
  hero-section         (Alternative hero layout)

# After selection, interactive field prompts
✅ Selected: hero-banner

Title (required): Welcome to ReedCMS
Subtitle (optional): Modern Content Management
Background URL (optional): /images/hero-bg.jpg

Suggested semantic name: $welcome_hero
Semantic name: $welcome_hero

✅ Created snippet $welcome_hero (hero-banner)

# Interactive update with field selection
$ reed snippet update $welcome_hero --interactive

🔧 Select field to update in $welcome_hero:
Filter: _

> title                (Current: Welcome to ReedCMS)
  subtitle             (Current: Modern Content Management)  
  background_url       (Current: /images/hero-bg.jpg)

# User selects title
New value for 'title': Welcome to the Future of CMS

✅ Updated $welcome_hero.title = Welcome to the Future of CMS
```

### **Smart Suggestions in Dropdown**
```rust
impl TerminalDropdown {
    fn add_smart_suggestions(&mut self, context: &CLIContext) {
        // Add recently used items at top
        let recent_items = context.get_recent_selections();
        for item in recent_items {
            self.items.insert(0, SelectionItem {
                display_text: format!("🕒 {}", item.name),
                value: item.value,
                description: Some("Recently used".to_string()),
                category: Some("recent".to_string()),
            });
        }
        
        // Add related suggestions
        if let Some(current_snippet) = &context.working_snippet {
            let related = self.get_related_snippet_types(current_snippet);
            for related_type in related {
                self.items.push(SelectionItem {
                    display_text: format!("🔗 {}", related_type.name),
                    value: related_type.name,
                    description: Some("Related to current snippet".to_string()),
                    category: Some("related".to_string()),
                });
            }
        }
    }
}
```
```bash
# Interactive output (human-readable)
$ reed snippet list
┌─────────────────┬──────────────┬─────────────┐
│ Name            │ Type         │ Created     │
├─────────────────┼──────────────┼─────────────┤
│ $homepage       │ page         │ 2025-01-15  │
│ $welcome_hero   │ hero-banner  │ 2025-01-15  │
└─────────────────┴──────────────┴─────────────┘

# Script output (TOML format - ReedCMS standard)
$ reed snippet list --format toml
[[snippets]]
name = "homepage"
type = "page" 
created = "2025-01-15"
semantic_name = "$homepage"

[[snippets]]
name = "welcome_hero"
type = "hero-banner"
created = "2025-01-15" 
semantic_name = "$welcome_hero"

# Simple script output (minimal)
$ reed snippet list --format simple
$homepage
$welcome_hero
```

### **Fast Command Execution**
```rust
pub struct CLIPerformance {
    command_cache: HashMap<String, CachedResult>,
    background_preloader: BackgroundPreloader,
}

impl CLIPerformance {
    pub async fn execute_command_fast(&self, command: &str) -> Result<String, Error> {
        // 1. Check command cache first
        if let Some(cached) = self.command_cache.get(command) {
            if !cached.is_expired() {
                return Ok(cached.result.clone());
            }
        }
        
        // 2. Execute command with timeout
        let result = tokio::time::timeout(
            Duration::from_secs(10), 
            self.execute_command_impl(command)
        ).await??;
        
        // 3. Cache result for repeated commands
        self.cache_result(command, &result);
        
        Ok(result)
    }
    
    pub async fn preload_common_data(&mut self) -> Result<(), Error> {
        // Preload frequently accessed data in background
        tokio::spawn(async move {
            let _ = self.preload_snippet_registry().await;
            let _ = self.preload_semantic_names().await;
            let _ = self.preload_theme_list().await;
        });
        
        Ok(())
    }
}
```

### **CLI Startup Optimization**
```rust
// Fast CLI startup - lazy loading
impl CLIApplication {
    pub async fn new() -> Result<Self, Error> {
        // Only load essential components for startup
        let essential_config = load_essential_config().await?;
        
        Ok(CLIApplication {
            config: essential_config,
            registry: None,          // Lazy load when needed
            database: None,          // Lazy load when needed
            redis: None,             // Lazy load when needed
            preload_started: false,
        })
    }
    
    pub async fn ensure_components_loaded(&mut self) -> Result<(), Error> {
        if !self.preload_started {
            // Start background preloading
            self.start_background_preload().await?;
            self.preload_started = true;
        }
        
        // Load components on-demand
        if self.registry.is_none() {
            self.registry = Some(SnippetRegistry::load().await?);
        }
        
        Ok(())
    }
}
```

## User Experience Examples

### **Command Flow Examples**
```bash
# Example 1: New user creates first snippet
$ reed snippet create "hero-banner"
Error: Missing required parameter 'name'. 

💡 Usage: reed snippet create "hero-banner" / name $<semantic-name>
💡 Example: reed snippet create "hero-banner" / name $welcome_hero

$ reed snippet create "hero-banner" / name $welcome_hero
Error: Missing required fields for 'hero-banner': title

💡 Required fields: title, subtitle (optional)
💡 Example: reed snippet create "hero-banner" / name $welcome_hero / with title="Welcome"

$ reed snippet create "hero-banner" / name $welcome_hero / with title="Welcome to ReedCMS"
✅ Success: Created snippet $welcome_hero (hero-banner)
📍 UUID: 550e8400-e29b-41d4-a716-446655440000
🔗 Edit: reed snippet update $welcome_hero

# Example 2: Tab completion in action  
$ reed snippet update $wel<TAB>
$welcome_hero

$ reed snippet update $welcome_hero / set field "title" to <TAB>
<shows current value for context>

# Example 3: Helpful error recovery
$ reed snippet create "hero-baner"  # Typo
Error: Snippet type 'hero-baner' not found.

💡 Did you mean: hero-banner, hero-section
💡 Available types: reed registry list
```

This CLI UX design ensures developers have a smooth, predictable experience with intelligent completion, contextual help, and graceful error handling while maintaining KISS principles.

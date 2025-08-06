# CLI Output Formatting

## Overview

Output Formatting System für ReedCMS CLI. Unterstützt multiple Formate (Table, JSON, YAML, CSV) mit konsistenter Darstellung.

## Output Formatter Core

```rust
use std::io::{self, Write};
use prettytable::{Table, Row, Cell, format};
use colored::*;

/// Main output formatter
pub struct OutputFormatter {
    format: OutputFormat,
    color_enabled: bool,
    terminal_width: usize,
}

impl OutputFormatter {
    pub fn new(format: OutputFormat) -> Self {
        Self {
            format,
            color_enabled: atty::is(atty::Stream::Stdout),
            terminal_width: term_size::dimensions()
                .map(|(w, _)| w)
                .unwrap_or(80),
        }
    }
    
    /// Format and print command output
    pub fn print(&self, output: CommandOutput) -> Result<()> {
        match output {
            CommandOutput::Success(data) => self.print_data(data),
            CommandOutput::Error(msg) => self.print_error(&msg),
            CommandOutput::Empty => Ok(()),
        }
    }
    
    /// Print formatted data
    fn print_data(&self, data: OutputData) -> Result<()> {
        match self.format {
            OutputFormat::Table => self.print_as_table(data),
            OutputFormat::Json => self.print_as_json(data),
            OutputFormat::Yaml => self.print_as_yaml(data),
            OutputFormat::Csv => self.print_as_csv(data),
            OutputFormat::Plain => self.print_as_plain(data),
        }
    }
    
    /// Print error message
    fn print_error(&self, message: &str) -> Result<()> {
        if self.color_enabled {
            eprintln!("{}: {}", "Error".red().bold(), message);
        } else {
            eprintln!("Error: {}", message);
        }
        Ok(())
    }
}
```

## Table Formatting

```rust
impl OutputFormatter {
    /// Print data as formatted table
    fn print_as_table(&self, data: OutputData) -> Result<()> {
        match data {
            OutputData::Table(table_data) => {
                self.print_table_data(table_data)
            }
            OutputData::List(items) => {
                self.print_list_as_table(items)
            }
            OutputData::Json(value) => {
                self.print_json_as_table(value)
            }
            OutputData::Text(text) => {
                println!("{}", text);
                Ok(())
            }
        }
    }
    
    /// Print table data
    fn print_table_data(&self, data: TableData) -> Result<()> {
        let mut table = Table::new();
        
        // Set table format
        table.set_format(*format::consts::FORMAT_NO_LINESEP_WITH_TITLE);
        
        // Add headers
        let header_cells: Vec<Cell> = data.headers.iter()
            .map(|h| {
                if self.color_enabled {
                    Cell::new(&h.bold().to_string())
                } else {
                    Cell::new(h)
                }
            })
            .collect();
        table.set_titles(Row::new(header_cells));
        
        // Add rows
        for row_data in &data.rows {
            let cells: Vec<Cell> = row_data.iter()
                .enumerate()
                .map(|(i, value)| {
                    let formatted = self.format_cell_value(value, i);
                    Cell::new(&formatted)
                })
                .collect();
            table.add_row(Row::new(cells));
        }
        
        // Print with terminal width consideration
        let table_str = table.to_string();
        if table_str.lines().any(|line| line.len() > self.terminal_width) {
            // Use compact format for wide tables
            self.print_compact_table(&data)?;
        } else {
            table.printstd();
        }
        
        Ok(())
    }
    
    /// Format cell value with optional coloring
    fn format_cell_value(&self, value: &str, column: usize) -> String {
        if !self.color_enabled {
            return value.to_string();
        }
        
        // Apply coloring based on content
        if value.parse::<Uuid>().is_ok() {
            value.cyan().to_string()
        } else if value.contains("true") || value.contains("active") {
            value.green().to_string()
        } else if value.contains("false") || value.contains("inactive") {
            value.red().to_string()
        } else if value.parse::<f64>().is_ok() {
            value.yellow().to_string()
        } else {
            value.to_string()
        }
    }
    
    /// Print compact table for narrow terminals
    fn print_compact_table(&self, data: &TableData) -> Result<()> {
        for (i, row) in data.rows.iter().enumerate() {
            if i > 0 {
                println!("{}", "─".repeat(self.terminal_width.min(40)));
            }
            
            for (j, header) in data.headers.iter().enumerate() {
                if let Some(value) = row.get(j) {
                    let label = if self.color_enabled {
                        header.bold().to_string()
                    } else {
                        header.to_string()
                    };
                    
                    println!("{:15} {}", label + ":", value);
                }
            }
        }
        
        Ok(())
    }
}
```

## JSON Formatting

```rust
impl OutputFormatter {
    /// Print data as JSON
    fn print_as_json(&self, data: OutputData) -> Result<()> {
        let json_value = match data {
            OutputData::Json(value) => value,
            OutputData::Table(table) => self.table_to_json(table),
            OutputData::List(items) => serde_json::Value::Array(
                items.into_iter()
                    .map(serde_json::Value::String)
                    .collect()
            ),
            OutputData::Text(text) => serde_json::Value::String(text),
        };
        
        if self.color_enabled && atty::is(atty::Stream::Stdout) {
            // Pretty print with syntax highlighting
            self.print_colored_json(&json_value)?;
        } else {
            // Plain JSON output
            println!("{}", serde_json::to_string_pretty(&json_value)?);
        }
        
        Ok(())
    }
    
    /// Convert table to JSON
    fn table_to_json(&self, table: TableData) -> serde_json::Value {
        let rows: Vec<serde_json::Value> = table.rows.iter()
            .map(|row| {
                let mut obj = serde_json::Map::new();
                for (i, header) in table.headers.iter().enumerate() {
                    if let Some(value) = row.get(i) {
                        obj.insert(header.clone(), serde_json::Value::String(value.clone()));
                    }
                }
                serde_json::Value::Object(obj)
            })
            .collect();
        
        serde_json::Value::Array(rows)
    }
    
    /// Print JSON with syntax highlighting
    fn print_colored_json(&self, value: &serde_json::Value) -> Result<()> {
        let formatted = serde_json::to_string_pretty(value)?;
        let mut output = String::new();
        
        for line in formatted.lines() {
            let colored_line = self.colorize_json_line(line);
            output.push_str(&colored_line);
            output.push('\n');
        }
        
        print!("{}", output);
        Ok(())
    }
    
    /// Colorize single JSON line
    fn colorize_json_line(&self, line: &str) -> String {
        // Simple JSON syntax highlighting
        let mut result = line.to_string();
        
        // Colorize strings (including keys)
        let string_re = regex::Regex::new(r#""([^"\\]|\\.)*""#).unwrap();
        for mat in string_re.find_iter(line) {
            let original = mat.as_str();
            let colored = if original.ends_with('":') {
                // JSON key
                original.blue().to_string()
            } else {
                // String value
                original.green().to_string()
            };
            result = result.replace(original, &colored);
        }
        
        // Colorize numbers
        let number_re = regex::Regex::new(r"\b\d+\.?\d*\b").unwrap();
        for mat in number_re.find_iter(&result.clone()) {
            let original = mat.as_str();
            let colored = original.yellow().to_string();
            result = result.replace(original, &colored);
        }
        
        // Colorize booleans and null
        result = result.replace("true", &"true".cyan().to_string());
        result = result.replace("false", &"false".cyan().to_string());
        result = result.replace("null", &"null".magenta().to_string());
        
        result
    }
}
```

## YAML and CSV Formatting

```rust
impl OutputFormatter {
    /// Print data as YAML
    fn print_as_yaml(&self, data: OutputData) -> Result<()> {
        let value = self.data_to_json_value(data);
        let yaml = serde_yaml::to_string(&value)?;
        
        if self.color_enabled {
            // Simple YAML highlighting
            for line in yaml.lines() {
                let colored = if line.starts_with("- ") {
                    format!("{} {}", "-".yellow(), &line[2..])
                } else if line.contains(": ") {
                    let parts: Vec<&str> = line.splitn(2, ": ").collect();
                    if parts.len() == 2 {
                        format!("{}: {}", parts[0].blue(), parts[1])
                    } else {
                        line.to_string()
                    }
                } else {
                    line.to_string()
                };
                println!("{}", colored);
            }
        } else {
            print!("{}", yaml);
        }
        
        Ok(())
    }
    
    /// Print data as CSV
    fn print_as_csv(&self, data: OutputData) -> Result<()> {
        let mut writer = csv::Writer::from_writer(io::stdout());
        
        match data {
            OutputData::Table(table) => {
                // Write headers
                writer.write_record(&table.headers)?;
                
                // Write rows
                for row in &table.rows {
                    writer.write_record(row)?;
                }
            }
            OutputData::List(items) => {
                for item in items {
                    writer.write_record(&[item])?;
                }
            }
            _ => {
                // Convert to string and write as single cell
                let value = self.data_to_string(data);
                writer.write_record(&[value])?;
            }
        }
        
        writer.flush()?;
        Ok(())
    }
}
```

## Progress and Status Display

```rust
use indicatif::{ProgressBar, ProgressStyle, MultiProgress};

/// Progress indicator for long operations
pub struct ProgressDisplay {
    multi: MultiProgress,
    main_bar: Option<ProgressBar>,
    spinners: HashMap<String, ProgressBar>,
}

impl ProgressDisplay {
    pub fn new() -> Self {
        Self {
            multi: MultiProgress::new(),
            main_bar: None,
            spinners: HashMap::new(),
        }
    }
    
    /// Create main progress bar
    pub fn create_progress(&mut self, total: u64, message: &str) -> ProgressBar {
        let pb = ProgressBar::new(total);
        pb.set_style(
            ProgressStyle::default_bar()
                .template("{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} {msg}")
                .unwrap()
                .progress_chars("#>-")
        );
        pb.set_message(message.to_string());
        
        let pb = self.multi.add(pb);
        self.main_bar = Some(pb.clone());
        pb
    }
    
    /// Create spinner for subtask
    pub fn create_spinner(&mut self, id: &str, message: &str) -> ProgressBar {
        let spinner = ProgressBar::new_spinner();
        spinner.set_style(
            ProgressStyle::default_spinner()
                .template("{spinner:.green} {msg}")
                .unwrap()
        );
        spinner.set_message(message.to_string());
        
        let spinner = self.multi.add(spinner);
        self.spinners.insert(id.to_string(), spinner.clone());
        spinner
    }
    
    /// Finish progress display
    pub fn finish(&self) {
        if let Some(ref pb) = self.main_bar {
            pb.finish_with_message("Done");
        }
        
        for spinner in self.spinners.values() {
            spinner.finish_and_clear();
        }
    }
}
```

## Interactive Prompts

```rust
use dialoguer::{theme::ColorfulTheme, Input, Select, Confirm, MultiSelect};

/// Interactive prompt helpers
pub struct Prompts;

impl Prompts {
    /// Prompt for text input
    pub fn input(prompt: &str) -> Result<String> {
        Input::<String>::with_theme(&ColorfulTheme::default())
            .with_prompt(prompt)
            .interact_text()
            .map_err(|e| ReedError::Io(e))
    }
    
    /// Prompt for password
    pub fn password(prompt: &str) -> Result<String> {
        dialoguer::Password::with_theme(&ColorfulTheme::default())
            .with_prompt(prompt)
            .interact()
            .map_err(|e| ReedError::Io(e))
    }
    
    /// Prompt for confirmation
    pub fn confirm(prompt: &str, default: bool) -> Result<bool> {
        Confirm::with_theme(&ColorfulTheme::default())
            .with_prompt(prompt)
            .default(default)
            .interact()
            .map_err(|e| ReedError::Io(e))
    }
    
    /// Prompt for selection
    pub fn select<T: ToString>(prompt: &str, items: &[T]) -> Result<usize> {
        Select::with_theme(&ColorfulTheme::default())
            .with_prompt(prompt)
            .items(items)
            .interact()
            .map_err(|e| ReedError::Io(e))
    }
    
    /// Prompt for multiple selections
    pub fn multi_select<T: ToString>(prompt: &str, items: &[T]) -> Result<Vec<usize>> {
        MultiSelect::with_theme(&ColorfulTheme::default())
            .with_prompt(prompt)
            .items(items)
            .interact()
            .map_err(|e| ReedError::Io(e))
    }
}
```

## Status Messages

```rust
/// Formatted status messages
pub struct StatusMessage;

impl StatusMessage {
    /// Print success message
    pub fn success(message: &str) {
        if atty::is(atty::Stream::Stdout) {
            println!("{} {}", "✓".green(), message);
        } else {
            println!("SUCCESS: {}", message);
        }
    }
    
    /// Print warning message
    pub fn warning(message: &str) {
        if atty::is(atty::Stream::Stdout) {
            println!("{} {}", "⚠".yellow(), message.yellow());
        } else {
            println!("WARNING: {}", message);
        }
    }
    
    /// Print error message
    pub fn error(message: &str) {
        if atty::is(atty::Stream::Stderr) {
            eprintln!("{} {}", "✗".red(), message.red());
        } else {
            eprintln!("ERROR: {}", message);
        }
    }
    
    /// Print info message
    pub fn info(message: &str) {
        if atty::is(atty::Stream::Stdout) {
            println!("{} {}", "ℹ".blue(), message);
        } else {
            println!("INFO: {}", message);
        }
    }
    
    /// Print debug message (only if verbose)
    pub fn debug(message: &str, verbose_level: u8) {
        if verbose_level > 0 {
            if atty::is(atty::Stream::Stdout) {
                println!("{} {}", "🔍".dim(), message.dim());
            } else {
                println!("DEBUG: {}", message);
            }
        }
    }
}
```

## Diff Output

```rust
use similar::{ChangeTag, TextDiff};

/// Display diffs for content changes
pub struct DiffDisplay;

impl DiffDisplay {
    /// Show unified diff
    pub fn unified_diff(old: &str, new: &str, context_lines: usize) {
        let diff = TextDiff::from_lines(old, new);
        
        for change in diff.iter_all_changes() {
            let sign = match change.tag() {
                ChangeTag::Delete => "-".red(),
                ChangeTag::Insert => "+".green(),
                ChangeTag::Equal => " ".normal(),
            };
            
            print!("{}{}", sign, change);
        }
    }
    
    /// Show side-by-side diff
    pub fn side_by_side_diff(old: &str, new: &str, width: usize) {
        let half_width = width / 2 - 2;
        let old_lines: Vec<&str> = old.lines().collect();
        let new_lines: Vec<&str> = new.lines().collect();
        
        let max_lines = old_lines.len().max(new_lines.len());
        
        // Header
        println!("{:^width$} │ {:^width$}", "Old", "New", width = half_width);
        println!("{:─^width$}─┼─{:─^width$}", "", "", width = half_width);
        
        // Lines
        for i in 0..max_lines {
            let old_line = old_lines.get(i).unwrap_or(&"");
            let new_line = new_lines.get(i).unwrap_or(&"");
            
            let old_display = Self::truncate_line(old_line, half_width);
            let new_display = Self::truncate_line(new_line, half_width);
            
            if old_line != new_line {
                println!(
                    "{:width$} │ {:width$}",
                    old_display.red(),
                    new_display.green(),
                    width = half_width
                );
            } else {
                println!(
                    "{:width$} │ {:width$}",
                    old_display,
                    new_display,
                    width = half_width
                );
            }
        }
    }
    
    fn truncate_line(line: &str, max_width: usize) -> String {
        if line.len() <= max_width {
            line.to_string()
        } else {
            format!("{}...", &line[..max_width - 3])
        }
    }
}
```

## Summary

Dieses Output Formatting System bietet:
- **Multiple Formats** - Table, JSON, YAML, CSV, Plain
- **Color Support** - Syntax Highlighting und Visual Cues
- **Terminal Awareness** - Anpassung an Terminal-Größe
- **Progress Display** - Bars und Spinners für lange Operations
- **Interactive Prompts** - User Input mit Validation

Output Formatting macht die CLI benutzerfreundlich und professionell.
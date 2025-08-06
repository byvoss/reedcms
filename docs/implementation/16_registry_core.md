# Registry Core

## Overview

Das Registry System definiert Snippet-Typen und deren Felder. Zentrale Verwaltung aller Content-Strukturen.

## Registry Manager

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

/// Central registry for snippet definitions
pub struct RegistryManager {
    definitions: Arc<RwLock<HashMap<String, SnippetDefinition>>>,
    csv_path: PathBuf,
    validators: Arc<FieldValidators>,
}

impl RegistryManager {
    pub fn new(csv_path: PathBuf) -> Self {
        Self {
            definitions: Arc::new(RwLock::new(HashMap::new())),
            csv_path,
            validators: Arc::new(FieldValidators::new()),
        }
    }
    
    /// Initialize registry from CSV
    pub async fn initialize(&self) -> Result<()> {
        let definitions = self.load_from_csv().await?;
        
        let mut registry = self.definitions.write().await;
        registry.clear();
        
        for def in definitions {
            registry.insert(def.name.clone(), def);
        }
        
        tracing::info!("Registry initialized with {} snippet types", registry.len());
        Ok(())
    }
    
    /// Get snippet definition
    pub async fn get_definition(&self, snippet_type: &str) -> Result<SnippetDefinition> {
        let registry = self.definitions.read().await;
        
        registry.get(snippet_type)
            .cloned()
            .ok_or_else(|| ReedError::Configuration(
                format!("Unknown snippet type: {}", snippet_type)
            ))
    }
    
    /// Register new snippet type
    pub async fn register_snippet(
        &self,
        definition: SnippetDefinition,
    ) -> Result<()> {
        // Validate definition
        self.validate_definition(&definition)?;
        
        let mut registry = self.definitions.write().await;
        
        if registry.contains_key(&definition.name) {
            return Err(ReedError::Configuration(
                format!("Snippet type already exists: {}", definition.name)
            ));
        }
        
        registry.insert(definition.name.clone(), definition);
        
        // Persist to CSV
        self.save_to_csv().await?;
        
        Ok(())
    }
}
```

## Snippet Definition Structure

```rust
/// Complete snippet type definition
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnippetDefinition {
    pub name: String,
    pub display_name: String,
    pub description: String,
    pub fields: Vec<FieldDefinition>,
    pub meta: MetaSnippetConfig,
    pub validation_rules: Vec<ValidationRule>,
}

/// Field definition within a snippet
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldDefinition {
    pub name: String,
    pub display_name: String,
    pub field_type: FieldType,
    pub required: bool,
    pub default_value: Option<serde_json::Value>,
    pub validation: Option<String>,
    pub help_text: Option<String>,
    pub localized: bool,
}

/// Field types supported by registry
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum FieldType {
    Text,
    Textarea,
    Number,
    Boolean,
    Date,
    Email,
    Url,
    Select(Vec<SelectOption>),
    Reference(String), // References another entity type
    Json,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SelectOption {
    pub value: String,
    pub label: String,
}

/// Meta configuration for snippets
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MetaSnippetConfig {
    pub is_routable: bool,
    pub is_navigable: bool,
    pub is_searchable: bool,
    pub cache_ttl: Option<u32>,
    pub allowed_child_types: Vec<String>,
    pub max_instances: Option<u32>,
}

/// Validation rules for snippet
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValidationRule {
    pub field: String,
    pub rule_type: ValidationType,
    pub parameters: HashMap<String, serde_json::Value>,
    pub error_message: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ValidationType {
    Required,
    MinLength,
    MaxLength,
    Pattern,
    Range,
    Custom(String),
}
```

## Registry Operations

```rust
impl RegistryManager {
    /// List all registered snippet types
    pub async fn list_snippet_types(&self) -> Vec<String> {
        let registry = self.definitions.read().await;
        registry.keys().cloned().collect()
    }
    
    /// Get field type for specific field
    pub async fn get_field_type(
        &self,
        snippet_type: &str,
        field_name: &str,
    ) -> Result<FieldType> {
        let definition = self.get_definition(snippet_type).await?;
        
        definition.fields.iter()
            .find(|f| f.name == field_name)
            .map(|f| f.field_type.clone())
            .ok_or_else(|| ReedError::Configuration(
                format!("Unknown field: {}.{}", snippet_type, field_name)
            ))
    }
    
    /// Check if snippet type is routable
    pub async fn is_routable(&self, snippet_type: &str) -> Result<bool> {
        let definition = self.get_definition(snippet_type).await?;
        Ok(definition.meta.is_routable)
    }
    
    /// Get required fields for snippet type
    pub async fn get_required_fields(
        &self,
        snippet_type: &str,
    ) -> Result<Vec<String>> {
        let definition = self.get_definition(snippet_type).await?;
        
        Ok(definition.fields.iter()
            .filter(|f| f.required)
            .map(|f| f.name.clone())
            .collect())
    }
    
    /// Get localized fields
    pub async fn get_localized_fields(
        &self,
        snippet_type: &str,
    ) -> Result<Vec<String>> {
        let definition = self.get_definition(snippet_type).await?;
        
        Ok(definition.fields.iter()
            .filter(|f| f.localized)
            .map(|f| f.name.clone())
            .collect())
    }
}
```

## CSV Loading and Saving

```rust
impl RegistryManager {
    /// Load definitions from CSV
    async fn load_from_csv(&self) -> Result<Vec<SnippetDefinition>> {
        let mut definitions = HashMap::new();
        
        // Load snippet registry
        let snippets_file = self.csv_path.join("snippets.csv");
        if snippets_file.exists() {
            self.load_snippets_csv(&snippets_file, &mut definitions).await?;
        }
        
        // Load meta configurations
        let meta_file = self.csv_path.join("meta_snippets.csv");
        if meta_file.exists() {
            self.load_meta_csv(&meta_file, &mut definitions).await?;
        }
        
        // Load validation rules
        let validation_file = self.csv_path.join("validation_rules.csv");
        if validation_file.exists() {
            self.load_validation_csv(&validation_file, &mut definitions).await?;
        }
        
        Ok(definitions.into_values().collect())
    }
    
    async fn load_snippets_csv(
        &self,
        path: &Path,
        definitions: &mut HashMap<String, SnippetDefinition>,
    ) -> Result<()> {
        let mut reader = csv::Reader::from_path(path)?;
        
        for result in reader.records() {
            let record = result?;
            
            let snippet_name = record.get(0).unwrap_or("").to_string();
            let field_name = record.get(1).unwrap_or("").to_string();
            let field_type = self.parse_field_type(record.get(2).unwrap_or("text"))?;
            let required = record.get(3).map(|s| s == "true").unwrap_or(false);
            let default_value = record.get(4)
                .filter(|s| !s.is_empty())
                .and_then(|s| serde_json::from_str(s).ok());
            let localized = record.get(5).map(|s| s == "true").unwrap_or(true);
            
            let definition = definitions.entry(snippet_name.clone())
                .or_insert_with(|| SnippetDefinition {
                    name: snippet_name.clone(),
                    display_name: snippet_name.clone(),
                    description: String::new(),
                    fields: vec![],
                    meta: MetaSnippetConfig {
                        is_routable: false,
                        is_navigable: false,
                        is_searchable: true,
                        cache_ttl: None,
                        allowed_child_types: vec![],
                        max_instances: None,
                    },
                    validation_rules: vec![],
                });
            
            definition.fields.push(FieldDefinition {
                name: field_name.clone(),
                display_name: field_name,
                field_type,
                required,
                default_value,
                validation: None,
                help_text: None,
                localized,
            });
        }
        
        Ok(())
    }
    
    fn parse_field_type(&self, type_str: &str) -> Result<FieldType> {
        match type_str {
            "text" => Ok(FieldType::Text),
            "textarea" => Ok(FieldType::Textarea),
            "number" => Ok(FieldType::Number),
            "boolean" => Ok(FieldType::Boolean),
            "date" => Ok(FieldType::Date),
            "email" => Ok(FieldType::Email),
            "url" => Ok(FieldType::Url),
            "json" => Ok(FieldType::Json),
            s if s.starts_with("reference:") => {
                let entity_type = s.strip_prefix("reference:").unwrap();
                Ok(FieldType::Reference(entity_type.to_string()))
            }
            s if s.starts_with("select:") => {
                let options_str = s.strip_prefix("select:").unwrap();
                let options: Vec<SelectOption> = options_str.split('|')
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
            _ => Err(ReedError::Configuration(format!("Unknown field type: {}", type_str))),
        }
    }
}
```

## Definition Validation

```rust
impl RegistryManager {
    /// Validate snippet definition
    fn validate_definition(&self, definition: &SnippetDefinition) -> Result<()> {
        // Validate name
        if !definition.name.chars().all(|c| c.is_ascii_lowercase() || c == '-') {
            return Err(ReedError::Validation(ValidationError::InvalidCharacters));
        }
        
        // Validate fields
        let mut field_names = HashSet::new();
        for field in &definition.fields {
            // Check duplicate field names
            if !field_names.insert(&field.name) {
                return Err(ReedError::Configuration(
                    format!("Duplicate field name: {}", field.name)
                ));
            }
            
            // Validate field name
            if !field.name.chars().all(|c| c.is_ascii_lowercase() || c == '_') {
                return Err(ReedError::Configuration(
                    format!("Invalid field name: {}", field.name)
                ));
            }
        }
        
        // Validate references
        for field in &definition.fields {
            if let FieldType::Reference(entity_type) = &field.field_type {
                // Check if referenced type exists
                // This would be validated at runtime
            }
        }
        
        Ok(())
    }
}
```

## Registry Queries

```rust
impl RegistryManager {
    /// Find snippets by capability
    pub async fn find_by_capability(
        &self,
        capability: SnippetCapability,
    ) -> Vec<String> {
        let registry = self.definitions.read().await;
        
        registry.iter()
            .filter(|(_, def)| match capability {
                SnippetCapability::Routable => def.meta.is_routable,
                SnippetCapability::Navigable => def.meta.is_navigable,
                SnippetCapability::Searchable => def.meta.is_searchable,
            })
            .map(|(name, _)| name.clone())
            .collect()
    }
    
    /// Get snippets that can contain a specific child type
    pub async fn get_allowed_parents(&self, child_type: &str) -> Vec<String> {
        let registry = self.definitions.read().await;
        
        registry.iter()
            .filter(|(_, def)| {
                def.meta.allowed_child_types.is_empty() ||
                def.meta.allowed_child_types.contains(&child_type.to_string())
            })
            .map(|(name, _)| name.clone())
            .collect()
    }
    
    /// Export registry as JSON
    pub async fn export_json(&self) -> Result<serde_json::Value> {
        let registry = self.definitions.read().await;
        
        let definitions: Vec<&SnippetDefinition> = registry.values().collect();
        Ok(serde_json::to_value(&definitions)?)
    }
}

#[derive(Debug, Clone, Copy)]
pub enum SnippetCapability {
    Routable,
    Navigable,
    Searchable,
}
```

## Summary

Dieses Registry Core Modul bietet:
- **Central Definition Store** - Alle Snippet-Typen an einem Ort
- **CSV-based Configuration** - Versionierbar und editierbar
- **Field Type System** - Strenge Typisierung für Felder
- **Meta Configuration** - Capabilities und Constraints
- **Validation Rules** - Eingebaute Validierung

Das Registry ist die Single Source of Truth für Content-Strukturen.
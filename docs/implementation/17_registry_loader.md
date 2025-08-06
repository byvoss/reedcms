# Registry CSV Loader

## Overview

CSV Loader für das Registry System. Lädt Snippet-Definitionen, Meta-Konfigurationen und Validierungsregeln.

## CSV Loader Implementation

```rust
use csv::Reader;
use std::path::Path;
use std::collections::HashMap;

/// Loads registry data from CSV files
pub struct RegistryLoader {
    base_path: PathBuf,
}

impl RegistryLoader {
    pub fn new(base_path: PathBuf) -> Self {
        Self { base_path }
    }
    
    /// Load complete registry from all CSV files
    pub async fn load_registry(&self) -> Result<RegistryData> {
        let mut data = RegistryData::default();
        
        // Load in specific order for dependencies
        self.load_field_types(&mut data).await?;
        self.load_snippets(&mut data).await?;
        self.load_meta_configs(&mut data).await?;
        self.load_validation_rules(&mut data).await?;
        self.load_relationships(&mut data).await?;
        
        // Post-process to build complete definitions
        let definitions = self.build_definitions(data)?;
        
        Ok(definitions)
    }
}

#[derive(Debug, Default)]
struct RegistryData {
    field_types: HashMap<String, FieldTypeConfig>,
    snippets: HashMap<String, Vec<FieldRecord>>,
    meta_configs: HashMap<String, MetaRecord>,
    validation_rules: HashMap<String, Vec<ValidationRecord>>,
    relationships: HashMap<String, Vec<String>>,
}
```

## Field Types CSV Loading

```rust
/// CSV Format: field_types.csv
/// name,base_type,validation_pattern,error_message
impl RegistryLoader {
    async fn load_field_types(&self, data: &mut RegistryData) -> Result<()> {
        let path = self.base_path.join("field_types.csv");
        
        if !path.exists() {
            return Ok(());
        }
        
        let mut reader = Reader::from_path(&path)?;
        
        for result in reader.records() {
            let record = result?;
            
            let name = record.get(0).unwrap_or("").to_string();
            let base_type = record.get(1).unwrap_or("text").to_string();
            let validation_pattern = record.get(2)
                .filter(|s| !s.is_empty())
                .map(|s| s.to_string());
            let error_message = record.get(3).unwrap_or("Invalid value").to_string();
            
            data.field_types.insert(name.clone(), FieldTypeConfig {
                name,
                base_type,
                validation_pattern,
                error_message,
            });
        }
        
        tracing::info!("Loaded {} field type configurations", data.field_types.len());
        Ok(())
    }
}

#[derive(Debug, Clone)]
struct FieldTypeConfig {
    name: String,
    base_type: String,
    validation_pattern: Option<String>,
    error_message: String,
}
```

## Snippets CSV Loading

```rust
/// CSV Format: snippets.csv
/// snippet_name,field_name,field_type,required,default_value,localized,help_text
impl RegistryLoader {
    async fn load_snippets(&self, data: &mut RegistryData) -> Result<()> {
        let path = self.base_path.join("snippets.csv");
        
        if !path.exists() {
            return Err(ReedError::Configuration(
                "snippets.csv not found".to_string()
            ));
        }
        
        let mut reader = Reader::from_path(&path)?;
        let headers = reader.headers()?.clone();
        
        for result in reader.records() {
            let record = result?;
            
            let snippet_name = record.get(0).unwrap_or("").to_string();
            let field_name = record.get(1).unwrap_or("").to_string();
            let field_type = record.get(2).unwrap_or("text").to_string();
            let required = record.get(3).map(|s| s == "true" || s == "1").unwrap_or(false);
            let default_value = record.get(4)
                .filter(|s| !s.is_empty())
                .map(|s| s.to_string());
            let localized = record.get(5)
                .map(|s| s == "true" || s == "1")
                .unwrap_or(true);
            let help_text = record.get(6)
                .filter(|s| !s.is_empty())
                .map(|s| s.to_string());
            
            let field_record = FieldRecord {
                name: field_name,
                field_type,
                required,
                default_value,
                localized,
                help_text,
            };
            
            data.snippets.entry(snippet_name)
                .or_insert_with(Vec::new)
                .push(field_record);
        }
        
        tracing::info!("Loaded {} snippet definitions", data.snippets.len());
        Ok(())
    }
}

#[derive(Debug, Clone)]
struct FieldRecord {
    name: String,
    field_type: String,
    required: bool,
    default_value: Option<String>,
    localized: bool,
    help_text: Option<String>,
}
```

## Meta Configuration Loading

```rust
/// CSV Format: meta_snippets.csv
/// snippet_name,is_routable,is_navigable,is_searchable,cache_ttl,max_instances
impl RegistryLoader {
    async fn load_meta_configs(&self, data: &mut RegistryData) -> Result<()> {
        let path = self.base_path.join("meta_snippets.csv");
        
        if !path.exists() {
            // Use defaults if no meta config
            return Ok(());
        }
        
        let mut reader = Reader::from_path(&path)?;
        
        for result in reader.records() {
            let record = result?;
            
            let snippet_name = record.get(0).unwrap_or("").to_string();
            let is_routable = record.get(1).map(|s| s == "true").unwrap_or(false);
            let is_navigable = record.get(2).map(|s| s == "true").unwrap_or(false);
            let is_searchable = record.get(3).map(|s| s == "true").unwrap_or(true);
            let cache_ttl = record.get(4)
                .and_then(|s| s.parse::<u32>().ok());
            let max_instances = record.get(5)
                .and_then(|s| s.parse::<u32>().ok());
            
            data.meta_configs.insert(snippet_name, MetaRecord {
                is_routable,
                is_navigable,
                is_searchable,
                cache_ttl,
                max_instances,
            });
        }
        
        tracing::info!("Loaded {} meta configurations", data.meta_configs.len());
        Ok(())
    }
}

#[derive(Debug, Clone)]
struct MetaRecord {
    is_routable: bool,
    is_navigable: bool,
    is_searchable: bool,
    cache_ttl: Option<u32>,
    max_instances: Option<u32>,
}
```

## Validation Rules Loading

```rust
/// CSV Format: validation_rules.csv
/// snippet_name,field_name,rule_type,parameters,error_message
impl RegistryLoader {
    async fn load_validation_rules(&self, data: &mut RegistryData) -> Result<()> {
        let path = self.base_path.join("validation_rules.csv");
        
        if !path.exists() {
            return Ok(());
        }
        
        let mut reader = Reader::from_path(&path)?;
        
        for result in reader.records() {
            let record = result?;
            
            let snippet_name = record.get(0).unwrap_or("").to_string();
            let field_name = record.get(1).unwrap_or("").to_string();
            let rule_type = record.get(2).unwrap_or("").to_string();
            let parameters = record.get(3).unwrap_or("").to_string();
            let error_message = record.get(4).unwrap_or("Validation failed").to_string();
            
            let validation_record = ValidationRecord {
                field_name,
                rule_type,
                parameters: self.parse_parameters(&parameters)?,
                error_message,
            };
            
            data.validation_rules.entry(snippet_name)
                .or_insert_with(Vec::new)
                .push(validation_record);
        }
        
        tracing::info!("Loaded validation rules for {} snippets", data.validation_rules.len());
        Ok(())
    }
    
    fn parse_parameters(&self, params_str: &str) -> Result<HashMap<String, serde_json::Value>> {
        let mut params = HashMap::new();
        
        if params_str.is_empty() {
            return Ok(params);
        }
        
        // Format: key1=value1;key2=value2
        for pair in params_str.split(';') {
            let parts: Vec<&str> = pair.split('=').collect();
            if parts.len() == 2 {
                let key = parts[0].trim();
                let value = parts[1].trim();
                
                // Try to parse as JSON, fallback to string
                let json_value = if let Ok(v) = serde_json::from_str(value) {
                    v
                } else {
                    serde_json::Value::String(value.to_string())
                };
                
                params.insert(key.to_string(), json_value);
            }
        }
        
        Ok(params)
    }
}

#[derive(Debug, Clone)]
struct ValidationRecord {
    field_name: String,
    rule_type: String,
    parameters: HashMap<String, serde_json::Value>,
    error_message: String,
}
```

## Relationship Loading

```rust
/// CSV Format: snippet_relationships.csv
/// parent_type,allowed_child_types
impl RegistryLoader {
    async fn load_relationships(&self, data: &mut RegistryData) -> Result<()> {
        let path = self.base_path.join("snippet_relationships.csv");
        
        if !path.exists() {
            return Ok(());
        }
        
        let mut reader = Reader::from_path(&path)?;
        
        for result in reader.records() {
            let record = result?;
            
            let parent_type = record.get(0).unwrap_or("").to_string();
            let allowed_children = record.get(1).unwrap_or("")
                .split('|')
                .filter(|s| !s.is_empty())
                .map(|s| s.trim().to_string())
                .collect::<Vec<_>>();
            
            data.relationships.insert(parent_type, allowed_children);
        }
        
        tracing::info!("Loaded relationships for {} snippet types", data.relationships.len());
        Ok(())
    }
}
```

## Building Complete Definitions

```rust
impl RegistryLoader {
    /// Build complete snippet definitions from loaded data
    fn build_definitions(&self, data: RegistryData) -> Result<Vec<SnippetDefinition>> {
        let mut definitions = Vec::new();
        
        for (snippet_name, fields) in data.snippets {
            // Get meta config or use defaults
            let meta = data.meta_configs.get(&snippet_name)
                .cloned()
                .unwrap_or_else(|| MetaRecord {
                    is_routable: false,
                    is_navigable: false,
                    is_searchable: true,
                    cache_ttl: None,
                    max_instances: None,
                });
            
            // Get validation rules
            let validation_rules = data.validation_rules.get(&snippet_name)
                .cloned()
                .unwrap_or_default();
            
            // Get allowed children
            let allowed_child_types = data.relationships.get(&snippet_name)
                .cloned()
                .unwrap_or_default();
            
            // Convert fields
            let field_definitions = self.convert_fields(fields, &data.field_types)?;
            
            // Convert validation rules
            let validation_definitions = self.convert_validations(validation_rules)?;
            
            definitions.push(SnippetDefinition {
                name: snippet_name.clone(),
                display_name: self.humanize_name(&snippet_name),
                description: format!("{} snippet type", self.humanize_name(&snippet_name)),
                fields: field_definitions,
                meta: MetaSnippetConfig {
                    is_routable: meta.is_routable,
                    is_navigable: meta.is_navigable,
                    is_searchable: meta.is_searchable,
                    cache_ttl: meta.cache_ttl,
                    allowed_child_types,
                    max_instances: meta.max_instances,
                },
                validation_rules: validation_definitions,
            });
        }
        
        Ok(definitions)
    }
    
    fn convert_fields(
        &self,
        fields: Vec<FieldRecord>,
        field_types: &HashMap<String, FieldTypeConfig>,
    ) -> Result<Vec<FieldDefinition>> {
        fields.into_iter()
            .map(|field| {
                let field_type = self.parse_field_type(&field.field_type)?;
                
                Ok(FieldDefinition {
                    name: field.name.clone(),
                    display_name: self.humanize_name(&field.name),
                    field_type,
                    required: field.required,
                    default_value: field.default_value
                        .and_then(|v| serde_json::from_str(&v).ok()),
                    validation: field_types.get(&field.field_type)
                        .and_then(|ft| ft.validation_pattern.clone()),
                    help_text: field.help_text,
                    localized: field.localized,
                })
            })
            .collect()
    }
    
    fn convert_validations(
        &self,
        rules: Vec<ValidationRecord>,
    ) -> Result<Vec<ValidationRule>> {
        rules.into_iter()
            .map(|rule| {
                Ok(ValidationRule {
                    field: rule.field_name,
                    rule_type: self.parse_validation_type(&rule.rule_type)?,
                    parameters: rule.parameters,
                    error_message: rule.error_message,
                })
            })
            .collect()
    }
    
    fn humanize_name(&self, name: &str) -> String {
        name.split(&['-', '_'][..])
            .map(|word| {
                let mut chars = word.chars();
                match chars.next() {
                    None => String::new(),
                    Some(first) => first.to_uppercase().chain(chars).collect(),
                }
            })
            .collect::<Vec<_>>()
            .join(" ")
    }
    
    fn parse_validation_type(&self, type_str: &str) -> Result<ValidationType> {
        match type_str {
            "required" => Ok(ValidationType::Required),
            "min_length" => Ok(ValidationType::MinLength),
            "max_length" => Ok(ValidationType::MaxLength),
            "pattern" => Ok(ValidationType::Pattern),
            "range" => Ok(ValidationType::Range),
            custom => Ok(ValidationType::Custom(custom.to_string())),
        }
    }
}
```

## CSV Format Examples

```csv
# snippets.csv
snippet_name,field_name,field_type,required,default_value,localized,help_text
hero-banner,title,text,true,,true,Main heading text
hero-banner,subtitle,text,false,,true,Optional subtitle
hero-banner,background_image,url,true,,false,Background image URL
hero-banner,cta_text,text,false,"Learn More",true,Call-to-action button text
hero-banner,cta_link,url,false,,false,Call-to-action link

# meta_snippets.csv
snippet_name,is_routable,is_navigable,is_searchable,cache_ttl,max_instances
hero-banner,false,false,true,300,
page,true,true,true,600,
blog-post,true,true,true,3600,

# validation_rules.csv
snippet_name,field_name,rule_type,parameters,error_message
hero-banner,title,max_length,max=100,Title must be less than 100 characters
hero-banner,background_image,pattern,pattern=^https?://,URL must start with http:// or https://
blog-post,slug,pattern,pattern=^[a-z0-9-]+$,Slug can only contain lowercase letters numbers and hyphens
```

## Summary

Dieser Registry Loader bietet:
- **Multi-file CSV Loading** - Strukturierte Datentrennung
- **Flexible Field Types** - Erweiterbare Typ-Definitionen
- **Validation Rules** - Deklarative Validierung
- **Relationship Management** - Parent-Child Constraints
- **Error Recovery** - Graceful handling fehlender Files

CSV als Configuration Format macht das System einfach editierbar.
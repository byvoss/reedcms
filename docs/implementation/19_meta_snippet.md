# Meta-Snippet Definition

## Overview

Meta-Snippets definieren die Eigenschaften und Capabilities von Snippet-Typen. Sie sind die Blaupause für Content-Strukturen.

## Meta-Snippet System

```rust
use std::collections::HashMap;

/// Meta-snippet manager for snippet type metadata
pub struct MetaSnippetManager {
    registry: Arc<RegistryManager>,
    cache: Arc<RwLock<HashMap<String, MetaSnippet>>>,
}

impl MetaSnippetManager {
    pub fn new(registry: Arc<RegistryManager>) -> Self {
        Self {
            registry,
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Get meta-snippet information
    pub async fn get_meta(&self, snippet_type: &str) -> Result<MetaSnippet> {
        // Check cache first
        {
            let cache = self.cache.read().await;
            if let Some(meta) = cache.get(snippet_type) {
                return Ok(meta.clone());
            }
        }
        
        // Load from registry
        let definition = self.registry.get_definition(snippet_type).await?;
        let meta = self.build_meta_snippet(definition)?;
        
        // Cache for future use
        {
            let mut cache = self.cache.write().await;
            cache.insert(snippet_type.to_string(), meta.clone());
        }
        
        Ok(meta)
    }
    
    /// Build meta-snippet from definition
    fn build_meta_snippet(&self, definition: SnippetDefinition) -> Result<MetaSnippet> {
        Ok(MetaSnippet {
            name: definition.name.clone(),
            display_name: definition.display_name.clone(),
            description: definition.description.clone(),
            capabilities: SnippetCapabilities {
                is_routable: definition.meta.is_routable,
                is_navigable: definition.meta.is_navigable,
                is_searchable: definition.meta.is_searchable,
                is_versionable: true, // Default
                is_translatable: definition.fields.iter().any(|f| f.localized),
            },
            constraints: SnippetConstraints {
                max_instances: definition.meta.max_instances,
                allowed_parents: self.determine_allowed_parents(&definition),
                allowed_children: definition.meta.allowed_child_types.clone(),
                required_fields: definition.fields.iter()
                    .filter(|f| f.required)
                    .map(|f| f.name.clone())
                    .collect(),
                unique_fields: vec![], // Could be extended
            },
            schema: FieldSchema {
                fields: definition.fields.clone(),
                validation_rules: definition.validation_rules.clone(),
                indexable_fields: self.determine_indexable_fields(&definition),
            },
            behaviors: SnippetBehaviors {
                cache_ttl: definition.meta.cache_ttl,
                preview_template: format!("{}.preview.tera", definition.name),
                admin_template: format!("{}.admin.tera", definition.name),
                api_enabled: true, // Default
            },
        })
    }
}
```

## Meta-Snippet Structure

```rust
/// Complete meta-snippet information
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MetaSnippet {
    pub name: String,
    pub display_name: String,
    pub description: String,
    pub capabilities: SnippetCapabilities,
    pub constraints: SnippetConstraints,
    pub schema: FieldSchema,
    pub behaviors: SnippetBehaviors,
}

/// Snippet capabilities
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnippetCapabilities {
    pub is_routable: bool,      // Can have URL/slug
    pub is_navigable: bool,     // Can appear in navigation
    pub is_searchable: bool,    // Include in search index
    pub is_versionable: bool,   // Track version history
    pub is_translatable: bool,  // Has localized fields
}

/// Snippet constraints
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnippetConstraints {
    pub max_instances: Option<u32>,
    pub allowed_parents: Vec<String>,
    pub allowed_children: Vec<String>,
    pub required_fields: Vec<String>,
    pub unique_fields: Vec<String>,
}

/// Field schema information
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldSchema {
    pub fields: Vec<FieldDefinition>,
    pub validation_rules: Vec<ValidationRule>,
    pub indexable_fields: Vec<String>,
}

/// Snippet behaviors
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnippetBehaviors {
    pub cache_ttl: Option<u32>,
    pub preview_template: String,
    pub admin_template: String,
    pub api_enabled: bool,
}
```

## Meta-Snippet Queries

```rust
impl MetaSnippetManager {
    /// Find snippets with specific capability
    pub async fn find_with_capability(
        &self,
        capability: SnippetCapability,
    ) -> Result<Vec<MetaSnippet>> {
        let types = self.registry.find_by_capability(capability).await;
        
        let mut metas = Vec::new();
        for snippet_type in types {
            if let Ok(meta) = self.get_meta(&snippet_type).await {
                metas.push(meta);
            }
        }
        
        Ok(metas)
    }
    
    /// Get all routable snippets
    pub async fn get_routable_snippets(&self) -> Result<Vec<MetaSnippet>> {
        self.find_with_capability(SnippetCapability::Routable).await
    }
    
    /// Check if parent can contain child
    pub async fn can_contain(
        &self,
        parent_type: &str,
        child_type: &str,
    ) -> Result<bool> {
        let parent_meta = self.get_meta(parent_type).await?;
        
        // Empty allowed_children means allow all
        if parent_meta.constraints.allowed_children.is_empty() {
            return Ok(true);
        }
        
        Ok(parent_meta.constraints.allowed_children.contains(&child_type.to_string()))
    }
    
    /// Get field information
    pub async fn get_field_info(
        &self,
        snippet_type: &str,
        field_name: &str,
    ) -> Result<FieldInfo> {
        let meta = self.get_meta(snippet_type).await?;
        
        let field = meta.schema.fields.iter()
            .find(|f| f.name == field_name)
            .ok_or_else(|| ReedError::Configuration(
                format!("Unknown field: {}.{}", snippet_type, field_name)
            ))?;
        
        let rules: Vec<&ValidationRule> = meta.schema.validation_rules.iter()
            .filter(|r| r.field == field_name)
            .collect();
        
        Ok(FieldInfo {
            definition: field.clone(),
            is_required: meta.constraints.required_fields.contains(&field_name.to_string()),
            is_unique: meta.constraints.unique_fields.contains(&field_name.to_string()),
            is_indexable: meta.schema.indexable_fields.contains(&field_name.to_string()),
            validation_rules: rules.into_iter().cloned().collect(),
        })
    }
}

#[derive(Debug, Clone)]
pub struct FieldInfo {
    pub definition: FieldDefinition,
    pub is_required: bool,
    pub is_unique: bool,
    pub is_indexable: bool,
    pub validation_rules: Vec<ValidationRule>,
}
```

## Helper Methods

```rust
impl MetaSnippetManager {
    /// Determine allowed parent types
    fn determine_allowed_parents(&self, definition: &SnippetDefinition) -> Vec<String> {
        // This could be loaded from relationships or determined by rules
        // For now, return common parent types
        if definition.meta.is_routable {
            vec!["page".to_string(), "section".to_string()]
        } else {
            vec![] // Can be child of any type
        }
    }
    
    /// Determine indexable fields
    fn determine_indexable_fields(&self, definition: &SnippetDefinition) -> Vec<String> {
        definition.fields.iter()
            .filter(|f| {
                // Index searchable text fields
                matches!(f.field_type, FieldType::Text | FieldType::Textarea) ||
                // Index reference fields for relationships
                matches!(f.field_type, FieldType::Reference(_)) ||
                // Index select fields for filtering
                matches!(f.field_type, FieldType::Select(_))
            })
            .map(|f| f.name.clone())
            .collect()
    }
    
    /// Validate snippet instance against meta
    pub async fn validate_instance(
        &self,
        snippet_type: &str,
        instance_data: &HashMap<String, serde_json::Value>,
    ) -> Result<ValidationResult> {
        let meta = self.get_meta(snippet_type).await?;
        let mut result = ValidationResult::valid();
        
        // Check required fields
        for required_field in &meta.constraints.required_fields {
            if !instance_data.contains_key(required_field) {
                result = result.merge(ValidationResult::invalid(ValidationError {
                    field: required_field.clone(),
                    message: format!("Required field '{}' is missing", required_field),
                    code: "required".to_string(),
                }));
            }
        }
        
        // Check field types
        for field in &meta.schema.fields {
            if let Some(value) = instance_data.get(&field.name) {
                // Validate against field type
                // This would use the FieldValidators
            }
        }
        
        Ok(result)
    }
}
```

## Meta-Snippet Export

```rust
impl MetaSnippetManager {
    /// Export meta-snippet as JSON schema
    pub async fn export_json_schema(
        &self,
        snippet_type: &str,
    ) -> Result<serde_json::Value> {
        let meta = self.get_meta(snippet_type).await?;
        
        let mut properties = serde_json::Map::new();
        let mut required = Vec::new();
        
        for field in &meta.schema.fields {
            let field_schema = self.field_to_json_schema(&field);
            properties.insert(field.name.clone(), field_schema);
            
            if field.required {
                required.push(field.name.clone());
            }
        }
        
        Ok(serde_json::json!({
            "$schema": "http://json-schema.org/draft-07/schema#",
            "title": meta.display_name,
            "description": meta.description,
            "type": "object",
            "properties": properties,
            "required": required,
            "additionalProperties": false,
        }))
    }
    
    fn field_to_json_schema(&self, field: &FieldDefinition) -> serde_json::Value {
        let mut schema = match &field.field_type {
            FieldType::Text | FieldType::Textarea => serde_json::json!({
                "type": "string",
                "description": field.help_text,
            }),
            FieldType::Number => serde_json::json!({
                "type": "number",
                "description": field.help_text,
            }),
            FieldType::Boolean => serde_json::json!({
                "type": "boolean",
                "description": field.help_text,
            }),
            FieldType::Date => serde_json::json!({
                "type": "string",
                "format": "date-time",
                "description": field.help_text,
            }),
            FieldType::Email => serde_json::json!({
                "type": "string",
                "format": "email",
                "description": field.help_text,
            }),
            FieldType::Url => serde_json::json!({
                "type": "string",
                "format": "uri",
                "description": field.help_text,
            }),
            FieldType::Select(options) => {
                let values: Vec<&str> = options.iter()
                    .map(|opt| opt.value.as_str())
                    .collect();
                serde_json::json!({
                    "type": "string",
                    "enum": values,
                    "description": field.help_text,
                })
            }
            FieldType::Reference(entity_type) => serde_json::json!({
                "type": "string",
                "format": "uuid",
                "description": format!("Reference to {} entity", entity_type),
            }),
            FieldType::Json => serde_json::json!({
                "type": "object",
                "description": field.help_text,
            }),
        };
        
        // Add default value if present
        if let Some(default) = &field.default_value {
            schema["default"] = default.clone();
        }
        
        schema
    }
}
```

## Summary

Dieses Meta-Snippet System bietet:
- **Complete Type Information** - Alle Eigenschaften eines Snippet-Typs
- **Capability Queries** - Finde Snippets nach Fähigkeiten
- **Constraint Validation** - Überprüfung von Parent-Child Beziehungen
- **Schema Export** - JSON Schema für externe Tools
- **Field Information** - Detaillierte Feld-Metadaten

Meta-Snippets sind das Herzstück der Content-Struktur.
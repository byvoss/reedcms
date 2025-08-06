# UCG CSV Recovery System

## Overview

CSV Recovery System für das Universal Content Graph. Lädt die komplette UCG-Struktur aus CSV-Dateien als Source of Truth.

## CSV Loader Core

```rust
use csv;
use std::path::{Path, PathBuf};

/// Loads UCG structure from CSV files
pub struct CsvLoader {
    config_path: PathBuf,
    redis: Arc<RedisManager>,
    memory: Arc<MemoryManager>,
}

impl CsvLoader {
    pub fn new(
        config_path: PathBuf,
        redis: Arc<RedisManager>,
        memory: Arc<MemoryManager>,
    ) -> Self {
        Self {
            config_path,
            redis,
            memory,
        }
    }
    
    /// Load all CSV data and rebuild UCG
    pub async fn load_all(&self) -> Result<CsvLoadResult> {
        tracing::info!("Starting CSV recovery from: {:?}", self.config_path);
        
        let mut result = CsvLoadResult::default();
        
        // Load in correct order for dependencies
        result.themes = self.load_themes().await?;
        result.snippets = self.load_snippets().await?;
        result.entities = self.load_entities().await?;
        result.associations = self.load_associations().await?;
        result.translations = self.load_translations().await?;
        
        // Build search index from loaded data
        result.search_index = self.build_search_index(&result.entities).await?;
        
        tracing::info!(
            "CSV recovery complete: {} entities, {} associations",
            result.entities.len(),
            result.associations.len()
        );
        
        Ok(result)
    }
    
    /// Rebuild complete UCG from CSV
    pub async fn rebuild_ucg(&self) -> Result<()> {
        // Load all data
        let data = self.load_all().await?;
        
        // Clear Redis (nuclear option)
        self.redis.flushdb().await?;
        
        // Store entities
        for entity in &data.entities {
            self.memory.store_ucg_entity(entity).await?;
        }
        
        // Store associations
        for assoc in &data.associations {
            self.memory.store_ucg_association(assoc).await?;
        }
        
        // Store search index
        for (word, entity_ids) in &data.search_index {
            self.memory.store_search_index(word, entity_ids).await?;
        }
        
        tracing::info!("UCG rebuilt from CSV successfully");
        Ok(())
    }
}

#[derive(Debug, Default)]
pub struct CsvLoadResult {
    pub entities: Vec<Entity>,
    pub associations: Vec<Association>,
    pub themes: Vec<Theme>,
    pub snippets: Vec<SnippetDefinition>,
    pub translations: HashMap<String, HashMap<String, String>>,
    pub search_index: HashMap<String, Vec<Uuid>>,
}
```

## Entity CSV Loading

```rust
impl CsvLoader {
    /// Load entities from CSV
    async fn load_entities(&self) -> Result<Vec<Entity>> {
        let file_path = self.config_path.join("entities.csv");
        
        if !file_path.exists() {
            tracing::warn!("No entities.csv found, skipping");
            return Ok(vec![]);
        }
        
        let mut reader = csv::Reader::from_path(&file_path)?;
        let mut entities = Vec::new();
        
        for result in reader.records() {
            let record = result?;
            
            // Parse CSV columns
            let id = Uuid::parse_str(record.get(0).unwrap_or(""))?;
            let entity_type = self.parse_entity_type(record.get(1).unwrap_or("snippet"))?;
            let semantic_name = record.get(2)
                .filter(|s| !s.is_empty())
                .and_then(|s| SemanticName::new(s).ok());
            
            let data_json = record.get(3).unwrap_or("{}");
            let data = serde_json::from_str(data_json)?;
            
            let created_by = record.get(4).unwrap_or("system").to_string();
            let created_at = record.get(5)
                .and_then(|s| DateTime::parse_from_rfc3339(s).ok())
                .map(|dt| dt.with_timezone(&Utc))
                .unwrap_or_else(Utc::now);
            
            entities.push(Entity {
                id,
                entity_type,
                semantic_name,
                created_at,
                updated_at: created_at,
                created_by,
                data,
            });
        }
        
        tracing::info!("Loaded {} entities from CSV", entities.len());
        Ok(entities)
    }
    
    fn parse_entity_type(&self, type_str: &str) -> Result<EntityType> {
        match type_str {
            "snippet" => Ok(EntityType::Snippet),
            "theme" => Ok(EntityType::Theme),
            "page" => Ok(EntityType::Page),
            "user" => Ok(EntityType::User),
            "plugin" => Ok(EntityType::Plugin),
            other => Ok(EntityType::Custom(other.to_string())),
        }
    }
}
```

## Association CSV Loading

```rust
impl CsvLoader {
    /// Load associations from CSV
    async fn load_associations(&self) -> Result<Vec<Association>> {
        let file_path = self.config_path.join("associations.csv");
        
        if !file_path.exists() {
            tracing::warn!("No associations.csv found, skipping");
            return Ok(vec![]);
        }
        
        let mut reader = csv::Reader::from_path(&file_path)?;
        let mut associations = Vec::new();
        
        for result in reader.records() {
            let record = result?;
            
            let parent_id = Uuid::parse_str(record.get(0).unwrap_or(""))?;
            let child_id = Uuid::parse_str(record.get(1).unwrap_or(""))?;
            let path = record.get(2).unwrap_or("").to_string();
            let association_type = self.parse_association_type(record.get(3).unwrap_or("contains"));
            let weight = record.get(4)
                .and_then(|s| s.parse::<i32>().ok())
                .unwrap_or(0);
            
            associations.push(Association {
                id: Uuid::new_v4(),
                parent_id,
                child_id,
                association_type,
                weight,
                path,
            });
        }
        
        tracing::info!("Loaded {} associations from CSV", associations.len());
        Ok(associations)
    }
    
    fn parse_association_type(&self, type_str: &str) -> AssociationType {
        match type_str {
            "contains" => AssociationType::Contains,
            "references" => AssociationType::References,
            "extends" => AssociationType::Extends,
            other => AssociationType::Custom(other.to_string()),
        }
    }
}
```

## Theme CSV Loading

```rust
impl CsvLoader {
    /// Load themes from CSV
    async fn load_themes(&self) -> Result<Vec<Theme>> {
        let file_path = self.config_path.join("themes.csv");
        
        if !file_path.exists() {
            return Ok(vec![Theme {
                name: "base".to_string(),
                parent: None,
                context: ThemeContext {
                    context_type: "default".to_string(),
                    context_value: "default".to_string(),
                },
                active: true,
            }]);
        }
        
        let mut reader = csv::Reader::from_path(&file_path)?;
        let mut themes = Vec::new();
        
        for result in reader.records() {
            let record = result?;
            
            let name = record.get(0).unwrap_or("base").to_string();
            let parent = record.get(1)
                .filter(|s| !s.is_empty())
                .map(|s| s.to_string());
            
            let context_type = record.get(2).unwrap_or("default").to_string();
            let context_value = record.get(3).unwrap_or("default").to_string();
            let active = record.get(4)
                .map(|s| s == "true" || s == "1")
                .unwrap_or(true);
            
            themes.push(Theme {
                name,
                parent,
                context: ThemeContext {
                    context_type,
                    context_value,
                },
                active,
            });
        }
        
        tracing::info!("Loaded {} themes from CSV", themes.len());
        Ok(themes)
    }
}
```

## Snippet Registry Loading

```rust
#[derive(Debug, Clone)]
pub struct SnippetDefinition {
    pub name: String,
    pub fields: Vec<FieldDefinition>,
    pub meta: SnippetMetaDefinition,
}

#[derive(Debug, Clone)]
pub struct FieldDefinition {
    pub name: String,
    pub field_type: String,
    pub required: bool,
    pub default_value: Option<String>,
    pub validation: Option<String>,
}

#[derive(Debug, Clone)]
pub struct SnippetMetaDefinition {
    pub is_routable: bool,
    pub is_navigable: bool,
    pub is_searchable: bool,
}

impl CsvLoader {
    /// Load snippet definitions from registry
    async fn load_snippets(&self) -> Result<Vec<SnippetDefinition>> {
        let file_path = self.config_path.join("snippets.csv");
        
        if !file_path.exists() {
            tracing::warn!("No snippets.csv found, skipping");
            return Ok(vec![]);
        }
        
        let mut reader = csv::Reader::from_path(&file_path)?;
        let mut snippets = HashMap::new();
        
        for result in reader.records() {
            let record = result?;
            
            let snippet_name = record.get(0).unwrap_or("").to_string();
            let field_name = record.get(1).unwrap_or("").to_string();
            let field_type = record.get(2).unwrap_or("text").to_string();
            let required = record.get(3).map(|s| s == "true").unwrap_or(false);
            let default_value = record.get(4)
                .filter(|s| !s.is_empty())
                .map(|s| s.to_string());
            
            let snippet = snippets.entry(snippet_name.clone())
                .or_insert_with(|| SnippetDefinition {
                    name: snippet_name,
                    fields: vec![],
                    meta: SnippetMetaDefinition {
                        is_routable: false,
                        is_navigable: false,
                        is_searchable: true,
                    },
                });
            
            snippet.fields.push(FieldDefinition {
                name: field_name,
                field_type,
                required,
                default_value,
                validation: None,
            });
        }
        
        let snippets: Vec<_> = snippets.into_values().collect();
        tracing::info!("Loaded {} snippet definitions from CSV", snippets.len());
        Ok(snippets)
    }
}
```

## Translation Loading

```rust
impl CsvLoader {
    /// Load translations from CSV files
    async fn load_translations(&self) -> Result<HashMap<String, HashMap<String, String>>> {
        let mut all_translations = HashMap::new();
        
        // Load global translations
        let global_dir = self.config_path.clone();
        self.load_translation_dir(&global_dir, &mut all_translations).await?;
        
        // Load snippet-specific translations
        let snippets_dir = self.config_path.parent()
            .map(|p| p.join("snippets"))
            .unwrap_or_default();
            
        if snippets_dir.exists() {
            self.load_translation_dir(&snippets_dir, &mut all_translations).await?;
        }
        
        tracing::info!("Loaded translations for {} locales", all_translations.len());
        Ok(all_translations)
    }
    
    async fn load_translation_dir(
        &self,
        dir: &Path,
        translations: &mut HashMap<String, HashMap<String, String>>,
    ) -> Result<()> {
        let pattern = format!("{}/**/translations.*.csv", dir.display());
        
        for entry in glob::glob(&pattern)? {
            if let Ok(path) = entry {
                self.load_translation_file(&path, translations).await?;
            }
        }
        
        Ok(())
    }
    
    async fn load_translation_file(
        &self,
        path: &Path,
        translations: &mut HashMap<String, HashMap<String, String>>,
    ) -> Result<()> {
        // Extract locale from filename
        let filename = path.file_name()
            .and_then(|f| f.to_str())
            .ok_or_else(|| ReedError::Internal("Invalid filename".to_string()))?;
        
        let locale = filename
            .strip_prefix("translations.")
            .and_then(|s| s.strip_suffix(".csv"))
            .ok_or_else(|| ReedError::Internal("Invalid translation filename".to_string()))?;
        
        let mut reader = csv::Reader::from_path(path)?;
        let locale_translations = translations.entry(locale.to_string())
            .or_insert_with(HashMap::new);
        
        for result in reader.records() {
            let record = result?;
            
            let key = record.get(0).unwrap_or("").to_string();
            let value = record.get(1).unwrap_or("").to_string();
            
            if !key.is_empty() {
                locale_translations.insert(key, value);
            }
        }
        
        Ok(())
    }
}
```

## Search Index Building

```rust
impl CsvLoader {
    /// Build search index from entities
    async fn build_search_index(
        &self,
        entities: &[Entity],
    ) -> Result<HashMap<String, Vec<Uuid>>> {
        let mut index: HashMap<String, Vec<Uuid>> = HashMap::new();
        
        for entity in entities {
            // Extract searchable text from entity
            let text = self.extract_searchable_text(entity);
            
            // Tokenize and index
            for word in self.tokenize(&text) {
                index.entry(word.to_lowercase())
                    .or_insert_with(Vec::new)
                    .push(entity.id);
            }
        }
        
        // Remove duplicates
        for entity_list in index.values_mut() {
            entity_list.sort();
            entity_list.dedup();
        }
        
        tracing::info!("Built search index with {} unique words", index.len());
        Ok(index)
    }
    
    fn extract_searchable_text(&self, entity: &Entity) -> String {
        let mut text = String::new();
        
        // Add semantic name
        if let Some(ref name) = entity.semantic_name {
            text.push_str(name.as_str());
            text.push(' ');
        }
        
        // Extract text from data fields
        if let Some(obj) = entity.data.as_object() {
            for (key, value) in obj {
                if let Some(str_val) = value.as_str() {
                    if key == "title" || key == "content" || key == "description" {
                        text.push_str(str_val);
                        text.push(' ');
                    }
                }
            }
        }
        
        text
    }
    
    fn tokenize(&self, text: &str) -> Vec<&str> {
        text.split_whitespace()
            .filter(|word| word.len() >= 3)  // Minimum word length
            .collect()
    }
}
```

## Summary

Dieses CSV Recovery System bietet:
- **Complete UCG Recovery** - Alle Daten aus CSV wiederherstellen
- **Ordered Loading** - Korrekte Reihenfolge für Dependencies
- **Search Index Building** - Automatischer Aufbau aus Entity-Daten
- **Translation Support** - Multi-Layer Translation Loading
- **Robust Error Handling** - Graceful Degradation bei fehlenden Files

CSV als Source of Truth macht das System resilient und versionierbar.
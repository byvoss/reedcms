# ReedCMS Registry System

## Performance Intelligence Layer

Das Registry System sitzt zwischen dem Universal Content Graph und den User Operations. Es liefert Type Definitions, Validation Rules und Performance Hints - ohne die Core-Patterns zu verkomplizieren.

**Single Source of Truth:** Alle Snippet-Typen, Field-Schemas und Meta-Properties werden einmal in der Registry definiert und generieren automatisch Templates, Validierung und UI-Komponenten.

## Registry Architektur

### Drei-Schichten Registry System

```
┌─────────────────────────────────────────────────────────┐
│                User-Defined Layer                       │
│   Custom snippet types created via Admin/CLI           │
│   Stored in CSV, auto-generates files                  │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                Built-in Layer                           │
│   Core meta-snippets: page, menu, navigation-item      │
│   System-defined, cannot be overridden                 │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│              Universal Foundation                       │
│   Every snippet type works with same UCG patterns      │
│   Registry provides performance and validation hints   │
└─────────────────────────────────────────────────────────┘
```

## CSV als Source of Truth

### snippets.csv Format

```csv
snippet_name,field_name,field_type,field_schema,required,default_value
hero-banner,title,String,"min:3;max:100",true,"Default Title"
hero-banner,subtitle,String,"max:200",false,""
hero-banner,priority,Number,"min:1;max:10",false,5
hero-banner,theme,String,"enum:light,dark",false,"light"
```

### Field Schema Syntax

Die `field_schema` Spalte definiert Validierungsregeln:

**String Validierung:**
- `min:3` - Mindestlänge
- `max:100` - Maximallänge
- `pattern:^[a-z]+$` - Regex Pattern
- `enum:val1,val2,val3` - Erlaubte Werte
- `url` - Muss gültige URL sein
- `email` - Muss gültige E-Mail sein

**Number Validierung:**
- `min:0` - Minimalwert
- `max:100` - Maximalwert
- `step:0.01` - Erlaubte Schritte

**Array Validierung:**
- `type:String` - Element-Typ
- Nested Objects mit Semikolon getrennt

### Komplexe Field Schemas

```csv
# Nested Object Arrays
snippet_name,field_name,field_type,field_schema,required
accordeon,items,Array,"title:String(min:1;max:80);content:String(min:10;max:500);open:Boolean",true
product-grid,products,Array,"name:String(required);price:Number(min:0);category:String(enum:electronics,clothing)",true
```

## Meta-Snippet Definition

```rust
pub struct MetaSnippetDefinition {
    pub name: String,
    pub is_routable: bool,              // Kann URL haben
    pub is_navigable: bool,             // In Navigation
    pub is_searchable: bool,            // In Suche
    pub required_fields: Vec<String>,   
    pub indexable_fields: Vec<String>,  // DB Indexes
    pub max_nesting_depth: Option<u8>,
    pub description: String,
    pub icon: Option<String>,
    pub composition: Option<CompositionRule>,
}
```

## Built-in Meta-Snippets

### Page
```rust
MetaSnippetDefinition {
    name: "page".into(),
    is_routable: true,
    is_navigable: true,
    is_searchable: true,
    required_fields: vec!["title".into(), "slug".into()],
    indexable_fields: vec!["slug".into(), "title".into()],
    max_nesting_depth: Some(5),
    description: "Website page with URL routing".into(),
    icon: Some("document".into()),
    composition: None,
}
```

### Menu
```rust
MetaSnippetDefinition {
    name: "menu".into(),
    is_routable: false,
    is_navigable: false,
    is_searchable: false,
    required_fields: vec!["name".into()],
    indexable_fields: vec!["name".into()],
    max_nesting_depth: Some(3),
    description: "Navigation menu container".into(),
    icon: Some("menu".into()),
    composition: None,
}
```

## Registry Implementation

```rust
pub struct MetaSnippetRegistry {
    definitions: HashMap<String, MetaSnippetDefinition>,
    semantic_name_index: HashMap<String, String>,
    routing_cache: HashMap<String, String>,
}

impl MetaSnippetRegistry {
    // Performance: Nur routable Snippets abfragen
    pub fn get_routable_types(&self) -> Vec<String> {
        self.definitions.values()
            .filter(|def| def.is_routable)
            .map(|def| def.name.clone())
            .collect()
    }
    
    // Smart Indexing: Nur deklarierte Felder
    pub fn get_indexable_fields(&self, snippet_type: &str) -> Vec<String> {
        self.definitions.get(snippet_type)
            .map(|def| def.indexable_fields.clone())
            .unwrap_or_default()
    }
    
    // Type Safety: Validierung gegen Registry
    pub fn validate_snippet_data(&self, snippet_type: &str, data: &Value) -> Result<(), ValidationError> {
        let definition = self.definitions.get(snippet_type)
            .ok_or(ValidationError::UnknownType(snippet_type.to_string()))?;
        
        for required_field in &definition.required_fields {
            if !data.get(required_field).is_some() {
                return Err(ValidationError::MissingRequiredField(required_field.clone()));
            }
        }
        
        Ok(())
    }
}
```

## Performance Optimierungen

### Registry-gesteuerte Indexes

```sql
-- Nur routable Snippets bekommen Slug-Index
CREATE INDEX idx_snippets_routable_slug 
ON snippet_content(snippet_type, (additional_data->>'slug')) 
WHERE snippet_type IN (SELECT name FROM registry WHERE is_routable = true);

-- Nur searchable Snippets bekommen Search-Indexes
CREATE INDEX idx_snippets_searchable_title
ON snippet_content(snippet_type, (additional_data->>'title'))
WHERE snippet_type IN (SELECT name FROM registry WHERE is_searchable = true);
```

### Smart Query Optimization

```rust
// Fast Page Resolution mit Registry
pub async fn resolve_route(slug: &str) -> Option<Snippet> {
    let routable_types = registry.get_routable_types();
    
    // Nur Snippets abfragen die routable SIND
    let snippet = query!(
        "SELECT * FROM snippet_content 
         WHERE snippet_type = ANY($1) AND additional_data->>'slug' = $2",
        &routable_types[..], slug
    ).fetch_optional(&pool).await?;
    
    snippet
}
```

## Validierung zur Build-Zeit

```rust
pub fn validate_csv_schema(csv_path: &str) -> Result<(), SchemaError> {
    let csv_content = fs::read_to_string(csv_path)?;
    
    for record in csv::Reader::from_reader(csv_content.as_bytes()).deserialize() {
        let row: SnippetDefinition = record?;
        
        // Validiere Field Schema Syntax
        if let Some(schema) = row.field_schema {
            validate_schema_syntax(&schema)?;
        }
        
        // Prüfe Konsistenz
        if row.required && row.default_value.is_none() {
            return Err(SchemaError::RequiredFieldWithoutDefault(row.field_name));
        }
    }
    
    Ok(())
}
```

## Auto-Generation

Die Registry generiert automatisch:

1. **Web Component Definitionen** (siehe T07)
2. **Admin UI Forms** mit korrekten Input-Types
3. **Validation Rules** für Client und Server
4. **Database Indexes** für Performance
5. **TypeScript Interfaces** für Type-Safety

## Registry Update Flow

```bash
# 1. CSV editieren
$ vim config/snippets.csv

# 2. Validierung
$ reed registry validate

# 3. Generation
$ reed registry generate
✓ Generated 15 web components
✓ Created admin forms
✓ Updated TypeScript definitions

# 4. Database Sync
$ reed registry sync
✓ Updated PostgreSQL UCG schema
✓ Created performance indexes
✓ Warmed Redis cache
```

Die Registry macht ReedCMS intelligent - sie weiß, welche Snippets routable sind, welche Felder indexiert werden müssen und wie Daten validiert werden. Alles aus simplen CSV-Definitionen.
# ReedCMS-04-Registry.md

## Registry Philosophy

**Performance Intelligence Layer:** The Registry sits between the Universal Content Graph foundation and user operations, providing type definitions, validation rules, and performance optimization hints without adding complexity to the core data patterns.

**Single Source of Truth:** All snippet types, field schemas, and meta-properties are defined once in the Registry and automatically generate templates, validation, and UI components.

## Registry Architecture

### Three-Layer Registry System
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

## Core Data Structures

### Meta-Snippet Definition
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct MetaSnippetDefinition {
    pub name: String,                    // "hero-banner", "product-showcase"
    pub is_routable: bool,              // Can have URL slug for routing
    pub is_navigable: bool,             // Can appear in navigation menus
    pub is_searchable: bool,            // Include in search index
    pub required_fields: Vec<String>,    // Validation at creation
    pub indexable_fields: Vec<String>,   // Database indexes for fast queries
    pub max_nesting_depth: Option<u8>,  // Schema nesting limits
    pub description: String,             // Admin UI description
    pub icon: Option<String>,           // Admin UI icon
    pub composition: Option<CompositionRule>, // For composite snippets
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CompositionRule {
    pub composed_of: Vec<String>,       // Child snippet types
    pub layout: String,                 // "side-by-side", "stacked", "grid"
    pub constraints: Option<String>,    // JSON schema for composition rules
}
```

### Registry Implementation
```rust
pub struct MetaSnippetRegistry {
    definitions: HashMap<String, MetaSnippetDefinition>,
    semantic_name_index: HashMap<String, String>, // semantic_name -> snippet_id
    routing_cache: HashMap<String, String>,       // slug -> snippet_id (routable only)
}

impl MetaSnippetRegistry {
    // Performance optimization: only query routable snippets
    pub fn get_routable_types(&self) -> Vec<String> {
        self.definitions.values()
            .filter(|def| def.is_routable)
            .map(|def| def.name.clone())
            .collect()
    }
    
    // Smart indexing: only index declared fields
    pub fn get_indexable_fields(&self, snippet_type: &str) -> Vec<String> {
        self.definitions.get(snippet_type)
            .map(|def| def.indexable_fields.clone())
            .unwrap_or_default()
    }
    
    // Type safety: validate snippet data against registry
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

## Built-in Meta-Snippets

### Page Meta-Snippet
```rust
MetaSnippetDefinition {
    name: "page".into(),
    is_routable: true,
    is_navigable: false,
    is_searchable: true,
    required_fields: vec!["title".into(), "slug".into()],
    indexable_fields: vec!["slug".into(), "title".into(), "meta_description".into()],
    max_nesting_depth: None,
    description: "Website page with URL routing".into(),
    icon: Some("document".into()),
    composition: None, // Not a composite snippet
}
```

**Field Definition (page.js):**
```javascript
export const title = String;
export const slug = String;
export const meta_description = String;
export const content = Array; // Association to other snippets
```

### Menu Meta-Snippet
```rust
MetaSnippetDefinition {
    name: "menu".into(),
    is_routable: false,
    is_navigable: false,
    is_searchable: false,
    required_fields: vec!["name".into()],
    indexable_fields: vec!["name".into()],
    max_nesting_depth: Some(5),
    description: "Navigation menu container".into(),
    icon: Some("menu".into()),
    composition: None,
}
```

**Field Definition (menu.js):**
```javascript
export const name = String;
export const items = Array; // Association to navigation-items
```

### Navigation-Item Meta-Snippet
```rust
MetaSnippetDefinition {
    name: "navigation-item".into(),
    is_routable: false,
    is_navigable: true,
    is_searchable: false,
    required_fields: vec!["label".into()],
    indexable_fields: vec!["label".into(), "url".into()],
    max_nesting_depth: Some(3),
    description: "Single navigation menu item".into(),
    icon: Some("link".into()),
    composition: None,
}
```

**Field Definition (navigation-item.js):**
```javascript
export const label = String;
export const url = String;
export const target = String; // "_blank", "_self"
export const children = Array; // Nested navigation-items
```

## Field Schema System

### Field Type Definitions

ReedCMS supports sophisticated field schemas with nested structures and validation rules:

#### Basic Field Types
```csv
# Basic types with validation
snippet_name,field_name,field_type,field_schema,required,default_value
hero-banner,title,String,"min:3;max:100",true,"Default Title"
hero-banner,priority,Number,"min:1;max:10",false,5
hero-banner,published,Boolean,,false,false
hero-banner,tags,Array,"type:String;min:1;max:5",false,"[]"
```

#### Complex Array Schemas
```csv
# Nested object arrays with full schema definitions
snippet_name,field_name,field_type,field_schema,required,default_value
accordeon,items,Array,"title:String(min:1;max:80);content:String(min:10;max:500);open:Boolean;priority:Number(min:1;max:100)",true,"[]"
product-grid,products,Array,"name:String(required);price:Number(min:0);image:String(url);category:String(enum:electronics,clothing,books);tags:Array(type:String)",true,"[]"
navigation,menu_items,Array,"label:String(required;max:50);url:String(url);target:String(enum:_self,_blank);children:Array(type:Object;max:3)",false,"[]"
```

#### Advanced Schema Constraints
```csv
# Complex validation and relationships
snippet_name,field_name,field_type,field_schema,required,default_value
contact-form,fields,Array,"type:String(enum:text,email,tel,textarea);name:String(pattern:^[a-zA-Z_][a-zA-Z0-9_]*$);label:String(required);required:Boolean;validation:String",true,"[]"
image-gallery,images,Array,"src:String(url;required);alt:String(max:200);caption:String;width:Number(min:100;max:4000);height:Number(min:100;max:4000);format:String(enum:jpg,png,webp)",true,"[]"
pricing-table,tiers,Array,"name:String(required;max:30);price:Number(min:0);currency:String(enum:EUR,USD,GBP);features:Array(type:String);highlight:Boolean;cta_text:String(max:20)",false,"[]"
```

### Schema Validation Implementation

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldSchema {
    pub field_type: FieldType,
    pub constraints: Vec<FieldConstraint>,
    pub nested_schema: Option<Box<FieldSchema>>, // For Arrays
    pub object_fields: Option<HashMap<String, FieldSchema>>, // For Objects
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum FieldType {
    String,
    Number,
    Boolean,
    Array,
    Object,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum FieldConstraint {
    Required,
    Min(i64),
    Max(i64),
    Pattern(String),
    Enum(Vec<String>),
    Url,
    Email,
}

impl FieldSchema {
    pub fn parse_from_csv(schema_str: &str) -> Result<Self, SchemaError> {
        // Parse complex schema string like:
        // "title:String(min:1;max:80);content:String(min:10;max:500);open:Boolean"
        
        if schema_str.is_empty() {
            return Ok(FieldSchema::simple_string());
        }

        // Handle Array schemas
        if schema_str.starts_with("type:") {
            return Self::parse_array_schema(schema_str);
        }

        // Handle Object schemas (semicolon-separated fields)
        if schema_str.contains(';') && schema_str.contains(':') {
            return Self::parse_object_schema(schema_str);
        }

        // Handle simple constraint schemas
        Self::parse_simple_schema(schema_str)
    }

    fn parse_object_schema(schema_str: &str) -> Result<Self, SchemaError> {
        let mut object_fields = HashMap::new();
        
        for field_def in schema_str.split(';') {
            let parts: Vec<&str> = field_def.split(':').collect();
            if parts.len() != 2 {
                return Err(SchemaError::InvalidFormat(field_def.to_string()));
            }
            
            let field_name = parts[0].trim();
            let field_schema_str = parts[1].trim();
            
            let field_schema = Self::parse_field_with_constraints(field_schema_str)?;
            object_fields.insert(field_name.to_string(), field_schema);
        }
        
        Ok(FieldSchema {
            field_type: FieldType::Object,
            constraints: vec![],
            nested_schema: None,
            object_fields: Some(object_fields),
        })
    }

    fn parse_field_with_constraints(field_str: &str) -> Result<Self, SchemaError> {
        // Parse "String(min:1;max:80;required)" format
        let (type_str, constraints_str) = if field_str.contains('(') {
            let parts: Vec<&str> = field_str.splitn(2, '(').collect();
            (parts[0], parts[1].trim_end_matches(')'))
        } else {
            (field_str, "")
        };

        let field_type = match type_str {
            "String" => FieldType::String,
            "Number" => FieldType::Number,
            "Boolean" => FieldType::Boolean,
            "Array" => FieldType::Array,
            "Object" => FieldType::Object,
            _ => return Err(SchemaError::UnknownType(type_str.to_string())),
        };

        let mut constraints = vec![];
        if !constraints_str.is_empty() {
            for constraint in constraints_str.split(';') {
                constraints.push(Self::parse_constraint(constraint)?);
            }
        }

        Ok(FieldSchema {
            field_type,
            constraints,
            nested_schema: None,
            object_fields: None,
        })
    }

    fn parse_constraint(constraint_str: &str) -> Result<FieldConstraint, SchemaError> {
        if constraint_str == "required" {
            return Ok(FieldConstraint::Required);
        }
        if constraint_str == "url" {
            return Ok(FieldConstraint::Url);
        }
        if constraint_str == "email" {
            return Ok(FieldConstraint::Email);
        }

        if let Some(value) = constraint_str.strip_prefix("min:") {
            return Ok(FieldConstraint::Min(value.parse()?));
        }
        if let Some(value) = constraint_str.strip_prefix("max:") {
            return Ok(FieldConstraint::Max(value.parse()?));
        }
        if let Some(value) = constraint_str.strip_prefix("pattern:") {
            return Ok(FieldConstraint::Pattern(value.to_string()));
        }
        if let Some(value) = constraint_str.strip_prefix("enum:") {
            let options: Vec<String> = value.split(',').map(|s| s.trim().to_string()).collect();
            return Ok(FieldConstraint::Enum(options));
        }

        Err(SchemaError::UnknownConstraint(constraint_str.to_string()))
    }
}
```

### Schema Validation at Runtime

```rust
impl FieldSchema {
    pub fn validate_value(&self, value: &Value) -> Result<(), ValidationError> {
        match (&self.field_type, value) {
            (FieldType::String, Value::String(s)) => {
                self.validate_string_constraints(s)?;
            },
            (FieldType::Number, Value::Number(n)) => {
                self.validate_number_constraints(n.as_f64().unwrap())?;
            },
            (FieldType::Boolean, Value::Bool(_)) => {
                // Boolean values are always valid
            },
            (FieldType::Array, Value::Array(arr)) => {
                self.validate_array_constraints(arr)?;
                
                // Validate each array item against nested schema
                if let Some(nested_schema) = &self.nested_schema {
                    for item in arr {
                        nested_schema.validate_value(item)?;
                    }
                }
            },
            (FieldType::Object, Value::Object(obj)) => {
                self.validate_object_constraints(obj)?;
            },
            _ => return Err(ValidationError::TypeMismatch),
        }
        
        Ok(())
    }

    fn validate_string_constraints(&self, value: &str) -> Result<(), ValidationError> {
        for constraint in &self.constraints {
            match constraint {
                FieldConstraint::Min(min) => {
                    if value.len() < *min as usize {
                        return Err(ValidationError::TooShort(*min));
                    }
                },
                FieldConstraint::Max(max) => {
                    if value.len() > *max as usize {
                        return Err(ValidationError::TooLong(*max));
                    }
                },
                FieldConstraint::Pattern(pattern) => {
                    let regex = regex::Regex::new(pattern)?;
                    if !regex.is_match(value) {
                        return Err(ValidationError::PatternMismatch(pattern.clone()));
                    }
                },
                FieldConstraint::Enum(options) => {
                    if !options.contains(&value.to_string()) {
                        return Err(ValidationError::InvalidOption(options.clone()));
                    }
                },
                FieldConstraint::Url => {
                    if !value.starts_with("http://") && !value.starts_with("https://") {
                        return Err(ValidationError::InvalidUrl);
                    }
                },
                FieldConstraint::Email => {
                    if !value.contains('@') || !value.contains('.') {
                        return Err(ValidationError::InvalidEmail);
                    }
                },
                _ => {}, // Other constraints don't apply to strings
            }
        }
        Ok(())
    }

    fn validate_array_constraints(&self, value: &Vec<Value>) -> Result<(), ValidationError> {
        for constraint in &self.constraints {
            match constraint {
                FieldConstraint::Min(min) => {
                    if value.len() < *min as usize {
                        return Err(ValidationError::TooFewItems(*min));
                    }
                },
                FieldConstraint::Max(max) => {
                    if value.len() > *max as usize {
                        return Err(ValidationError::TooManyItems(*max));
                    }
                },
                _ => {}, // Other constraints don't apply to arrays
            }
        }
        Ok(())
    }
}
```

### Auto-Generated Form Fields

```rust
impl FieldSchema {
    pub fn generate_form_field(&self, field_name: &str) -> FormField {
        match self.field_type {
            FieldType::String => {
                let mut field = FormField::text(field_name);
                
                for constraint in &self.constraints {
                    match constraint {
                        FieldConstraint::Min(min) => field.min_length(*min as usize),
                        FieldConstraint::Max(max) => field.max_length(*max as usize),
                        FieldConstraint::Pattern(pattern) => field.pattern(pattern),
                        FieldConstraint::Enum(options) => return FormField::select(field_name, options),
                        FieldConstraint::Email => return FormField::email(field_name),
                        FieldConstraint::Url => return FormField::url(field_name),
                        _ => {},
                    }
                }
                
                field
            },
            FieldType::Number => {
                let mut field = FormField::number(field_name);
                
                for constraint in &self.constraints {
                    match constraint {
                        FieldConstraint::Min(min) => field.min(*min),
                        FieldConstraint::Max(max) => field.max(*max),
                        _ => {},
                    }
                }
                
                field
            },
            FieldType::Boolean => FormField::checkbox(field_name),
            FieldType::Array => {
                if let Some(nested_schema) = &self.nested_schema {
                    FormField::repeater(field_name, nested_schema.generate_form_field("item"))
                } else {
                    FormField::text_array(field_name)
                }
            },
            FieldType::Object => {
                if let Some(object_fields) = &self.object_fields {
                    let mut form_fields = vec![];
                    for (field_name, field_schema) in object_fields {
                        form_fields.push(field_schema.generate_form_field(field_name));
                    }
                    FormField::fieldset(field_name, form_fields)
                } else {
                    FormField::json(field_name)
                }
            },
        }
    }
}
```

## User-Defined Meta-Snippets

### Simple Snippet Definition
```csv
# snippets.csv
snippet_name,field_name,field_type,field_schema,required,default_value,searchable
hero-banner,title,String,"min:3;max:100",true,"Welcome",true
hero-banner,subtitle,String,"max:200",false,"",true
hero-banner,background_image,String,"url",false,"",false
hero-banner,closeable,Boolean,,false,false,false
hero-banner,animation_duration,Number,"min:100;max:5000",false,300,false
```

### Advanced Schema with CSS Controls
```csv
# CSS-steuerbare Snippets mit Critical CSS Controls
snippet_name,field_name,field_type,field_schema,required,default_value,css_control
text-with-image,title,String,"min:1;max:80",true,"Text Block",false
text-with-image,content,String,"min:10;max:2000",true,"Lorem ipsum...",false
text-with-image,image_url,String,"url",true,"",false
text-with-image,image_position,String,"enum:left,right,top,bottom",false,"left",false
text-with-image,border_style,String,"enum:none,solid,dashed,dotted",false,"none",false

# CSS Control Fields (max 10 per snippet)
text-with-image,css_margin_top,String,"pattern:^[0-9]+(px|rem|em|%)$",false,"2rem",true
text-with-image,css_padding,String,"pattern:^[0-9]+(px|rem|em|%)$",false,"1rem",true
text-with-image,css_border_radius,String,"pattern:^[0-9]+(px|rem|em|%)$",false,"0",true
text-with-image,css_max_width,String,"pattern:^[0-9]+(px|rem|em|%|vw)$",false,"100%",true
text-with-image,css_text_align,String,"enum:left,center,right,justify",false,"left",true
text-with-image,css_background_color,String,"pattern:^#[0-9a-fA-F]{6}$",false,"",true
text-with-image,css_box_shadow,String,,false,"",true
text-with-image,css_transform,String,,false,"",true
text-with-image,css_opacity,String,"pattern:^0(\.[0-9]+)?|1(\.0)?$",false,"1",true
text-with-image,css_z_index,String,"pattern:^-?[0-9]+$",false,"",true
```

### CSS Control Registry Implementation
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct MetaSnippetDefinition {
    pub name: String,
    pub is_routable: bool,
    pub is_navigable: bool,
    pub is_searchable: bool,
    pub required_fields: Vec<String>,
    pub indexable_fields: Vec<String>,
    pub css_control_fields: Vec<CssControlField>, // New: CSS controls
    pub max_nesting_depth: Option<u8>,
    pub description: String,
    pub icon: Option<String>,
    pub composition: Option<CompositionRule>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CssControlField {
    pub field_name: String,           // "css_margin_top"
    pub css_property: String,         // "margin-top" 
    pub default_value: Option<String>, // "2rem"
    pub validation_pattern: Option<String>, // CSS value validation
    pub description: String,          // "Top margin spacing"
}

impl MetaSnippetDefinition {
    pub fn get_css_controls(&self) -> &[CssControlField] {
        &self.css_control_fields
    }
    
    pub fn validate_css_control_count(&self) -> Result<(), RegistryError> {
        if self.css_control_fields.len() > 10 {
            return Err(RegistryError::TooManyCssControls(self.css_control_fields.len()));
        }
        Ok(())
    }
    
    pub fn generate_css_object(&self, field_values: &HashMap<String, String>) -> CssObject {
        let mut css = CssObject::new();
        
        for control in &self.css_control_fields {
            if let Some(value) = field_values.get(&control.field_name) {
                if !value.is_empty() {
                    css.insert(control.css_property.clone(), value.clone());
                }
            }
        }
        
        css
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CssObject {
    properties: HashMap<String, String>,
}

impl CssObject {
    pub fn new() -> Self {
        Self {
            properties: HashMap::new(),
        }
    }
    
    pub fn insert(&mut self, property: String, value: String) {
        self.properties.insert(property, value);
    }
    
    pub fn to_style_string(&self) -> String {
        self.properties
            .iter()
            .map(|(prop, value)| format!("{}: {};", prop, value))
            .collect::<Vec<_>>()
            .join(" ")
    }
    
    // For Tera template access
    pub fn get(&self, property: &str) -> Option<&String> {
        self.properties.get(property)
    }
}
```

**Auto-Generated Registry Entry:**
```rust
MetaSnippetDefinition {
    name: "hero-banner".into(),
    is_routable: false,
    is_navigable: false,
    is_searchable: true, // At least one field is searchable
    required_fields: vec!["title".into()],
    indexable_fields: vec!["title".into(), "subtitle".into()], // Searchable fields
    max_nesting_depth: Some(1),
    description: "Hero banner with title and optional subtitle".into(),
    icon: Some("image".into()),
    composition: None,
}
```

### Composite Snippet Definition
```csv
# snippets.csv
snippet_name,field_name,field_type,required,default_value,composition
text-with-picture,layout,String,true,"side-by-side",""
text-with-picture,text_position,String,false,"left",""
text-with-picture,composed_of,Array,false,"","text-snippet,picture-snippet"
```

**Auto-Generated Registry Entry:**
```rust
MetaSnippetDefinition {
    name: "text-with-picture".into(),
    is_routable: false,
    is_navigable: false,
    is_searchable: false,
    required_fields: vec!["layout".into()],
    indexable_fields: vec![],
    max_nesting_depth: Some(1),
    description: "Text block with accompanying image".into(),
    icon: Some("image-text".into()),
    composition: Some(CompositionRule {
        composed_of: vec!["text-snippet".into(), "picture-snippet".into()],
        layout: "side-by-side".into(),
        constraints: Some(r#"{"text_position": ["left", "right", "top", "bottom"]}"#.into()),
    }),
}
```

## Registry-Driven Performance Optimizations

### Smart Database Indexing
```sql
-- Registry-driven index creation
-- Only routable snippets get slug index
CREATE INDEX idx_snippets_routable_slug 
ON snippet_content(snippet_type, (additional_data->>'slug')) 
WHERE snippet_type IN (SELECT name FROM registry WHERE is_routable = true);

-- Only searchable snippets get search-field indexes  
CREATE INDEX idx_snippets_searchable_title
ON snippet_content(snippet_type, (additional_data->>'title'))
WHERE snippet_type IN (SELECT name FROM registry WHERE is_searchable = true);

-- Custom indexes based on indexable_fields
CREATE INDEX idx_snippets_product_category
ON snippet_content((additional_data->>'category'))
WHERE snippet_type = 'product-showcase';
```

### Routing Performance
```rust
// Fast page resolution using registry
pub async fn resolve_route(slug: &str) -> Option<Snippet> {
    let routable_types = registry.get_routable_types();
    
    // Only query snippets that CAN be routable
    let snippet = query!(
        "SELECT * FROM snippet_content 
         WHERE snippet_type = ANY($1) AND additional_data->>'slug' = $2",
        &routable_types[..], slug
    ).fetch_optional(&pool).await?;
    
    snippet
}
```

### Search Index Optimization
```rust
// Only index searchable snippet types
impl SearchIndexManager {
    pub async fn update_search_index(&self, snippet: &Snippet) -> Result<(), Error> {
        // Check registry before indexing
        if !self.registry.is_searchable(&snippet.snippet_type) {
            return Ok(()); // Skip non-searchable types
        }
        
        // Get searchable fields from registry
        let searchable_fields = self.registry.get_searchable_fields(&snippet.snippet_type);
        
        // Only index declared searchable fields
        for field in searchable_fields {
            if let Some(content) = snippet.data.get(&field) {
                self.index_field_content(&snippet.id, &field, content).await?;
            }
        }
        
        Ok(())
    }
}
```

## Composite Snippet System

### Auto-Creation of Child Snippets
```rust
// When user creates composite snippet instance
pub async fn create_composite_snippet(
    snippet_type: &str, 
    data: &Value,
    registry: &MetaSnippetRegistry
) -> Result<CompositeSnippetResult, Error> {
    
    let definition = registry.get_definition(snippet_type)?;
    
    if let Some(composition) = &definition.composition {
        // Create parent snippet
        let parent_snippet = Snippet::new(snippet_type, data);
        
        // Auto-create required child snippets
        let mut child_snippets = Vec::new();
        for (index, child_type) in composition.composed_of.iter().enumerate() {
            let child_snippet = Snippet::new(child_type, &Value::Object(Map::new()));
            
            // Create association using UCG pattern
            Association::create(
                parent_snippet.id.clone(),
                child_snippet.id.clone(),
                index as i32, // weight based on composition order
                format!("{}.{}", parent_snippet.semantic_name.unwrap(), index + 1)
            ).await?;
            
            child_snippets.push(child_snippet);
        }
        
        Ok(CompositeSnippetResult { 
            parent: parent_snippet, 
            children: child_snippets 
        })
    } else {
        // Create simple snippet
        Ok(CompositeSnippetResult::single(Snippet::new(snippet_type, data)))
    }
}
```

## Registry Lifecycle

### Bootstrap Process
1. **System Startup:** Load built-in meta-snippet definitions
2. **CSV Import:** Load user-defined snippets from CSV files
3. **Index Generation:** Create database indexes based on registry metadata
4. **Validation Setup:** Configure runtime validation rules
5. **Cache Warming:** Populate Redis with registry-optimized data

### Runtime Updates
```rust
// Registry updates trigger system-wide changes
pub async fn register_meta_snippet(definition: MetaSnippetDefinition) -> Result<(), RegistryError> {
    // Validate definition
    validate_meta_snippet(&definition)?;
    
    // Update registry
    registry.insert(definition.name.clone(), definition.clone());
    
    // Rebuild indexes if needed
    if !definition.indexable_fields.is_empty() {
        rebuild_indexes_for_type(&definition.name).await?;
    }
    
    // Update routing cache if routable
    if definition.is_routable {
        invalidate_routing_cache().await?;
    }
    
    // Update search index if searchable
    if definition.is_searchable {
        rebuild_search_index_for_type(&definition.name).await?;
    }
    
    Ok(())
}
```

### Validation Rules
```rust
// Registry ensures data consistency
#[derive(Debug)]
pub enum RegistryError {
    UnknownMetaSnippet(String),
    MissingRequiredField(String),
    ExceededNestingDepth(u8),
    InvalidComposition(String),
    CircularDependency(Vec<String>),
}

// Composition validation prevents circular dependencies
pub fn validate_composition(
    name: &str, 
    composed_of: &[String], 
    registry: &MetaSnippetRegistry
) -> Result<(), RegistryError> {
    let mut visited = HashSet::new();
    let mut stack = vec![name.to_string()];
    
    while let Some(current) = stack.pop() {
        if visited.contains(&current) {
            return Err(RegistryError::CircularDependency(visited.into_iter().collect()));
        }
        visited.insert(current.clone());
        
        if let Some(def) = registry.get_definition(&current) {
            if let Some(composition) = &def.composition {
                stack.extend(composition.composed_of.iter().cloned());
            }
        }
    }
    
    Ok(())
}
```

## CSV Integration

### Registry Export/Import
```csv
# registry.csv - Complete registry definitions
name,is_routable,is_navigable,is_searchable,required_fields,indexable_fields,max_depth,description,composition_types,composition_layout
page,true,false,true,"title,slug","slug,title,meta_description",,Website page with URL routing,,
menu,false,false,false,name,name,5,Navigation menu container,,
navigation-item,false,true,false,label,"label,url",3,Single navigation menu item,,
hero-banner,false,false,true,title,"title,subtitle",1,Hero banner component,,
text-with-picture,false,false,false,layout,,1,Text block with image,"text-snippet,picture-snippet",side-by-side
```

### Deployment Integration
```rust
// CSV-driven registry initialization
pub async fn initialize_registry_from_csv(csv_path: &str) -> Result<MetaSnippetRegistry, Error> {
    let csv_content = fs::read_to_string(csv_path).await?;
    let mut registry = MetaSnippetRegistry::with_builtins(); // Load built-ins first
    
    for record in csv::Reader::from_reader(csv_content.as_bytes()).deserialize() {
        let definition: MetaSnippetDefinition = record?;
        registry.register(definition).await?;
    }
    
    Ok(registry)
}
```

## Registry as Single Source of Truth

### Generated Components
- **Template Files:** Auto-generated from field definitions
- **JavaScript Classes:** Web Components with proper typing
- **CSS Scaffolding:** Basic structure and BEM classes
- **Admin Forms:** UI components for content editing
- **Validation Rules:** Client and server-side validation
- **Database Indexes:** Performance optimization hints

### Cross-System Integration
- **Performance queries** consult registry before database
- **Validation rules** derived from registry definitions  
- **Database indexes** generated from registry metadata
- **Admin UI** renders forms based on registry field definitions
- **API responses** include registry hints for client optimization
- **Search system** only indexes registry-declared searchable types

This registry system provides the intelligence layer that makes the Universal Content Graph both performant and developer-friendly while maintaining architectural simplicity and KISS-brain principles.

---

**Next: ReedCMS-05-CLI.md** - Learn how to use the command-line interface for daily development workflows and content management.
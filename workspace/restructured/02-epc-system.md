# Explicit Path Chain (EPC) System

## Definition

Das Explicit Path Chain (EPC) System ist ReedCMS's branded File Resolution System, das explizite Pfad-Hierarchien für Theme-Overrides verwendet.

**Innovation:** Selbst-bauende File Resolution Chains basierend auf UCG-Kontext statt hardcodierter Theme-Vererbung.

**Prinzip:** Die spezifischste Datei gewinnt, mit Fallback durch die explizite Pfad-Kette zum Standard.

## Core Implementation

```rust
// EPC (Explicit Path Chain) - ReedCMS branded file resolution system
pub fn resolve_explicit_path_chain(
    snippet_name: &str, 
    file_name: &str, 
    theme_chain: &[String]
) -> Option<PathBuf> {
    
    // Explicit path chain from most specific to default
    for theme in theme_chain.iter().rev() {
        let explicit_path = Path::new("themes")
            .join(theme)                    // Explicit theme path
            .join(snippet_name)             // Explicit snippet path  
            .join(file_name);               // Explicit file path
            
        if explicit_path.exists() {
            return Some(explicit_path);     // First explicit match wins
        }
    }
    
    // Default explicit path
    let default_path = Path::new("snippets")
        .join(snippet_name)
        .join(file_name);
        
    if default_path.exists() {
        Some(default_path)
    } else {
        None
    }
}
```

## Resolution Pattern

### Beispiel Theme Chain
```
Theme Chain: ["corporate", "corporate.berlin", "corporate.berlin.christmas"]
Looking for: hero-banner.css
```

### EPC Resolution Order
```
1. themes/corporate.berlin.christmas/hero-banner/hero-banner.css  ← Most specific (wins)
2. themes/corporate.berlin/hero-banner/hero-banner.css           ← Fallback if #1 missing  
3. themes/corporate/hero-banner/hero-banner.css                  ← Fallback if #2 missing
4. snippets/hero-banner/hero-banner.css                          ← Default fallback
```

## Theme Chain Building

Das Theme Chain wird dynamisch vom UCG aufgebaut:

```rust
// Theme Chain ist UCG-driven, nicht configuration files
pub async fn build_theme_chain(scope: &str) -> Vec<String> {
    let mut chain = Vec::new();
    
    // Split scope into dimensions
    // "corporate.berlin.christmas" → ["corporate", "berlin", "christmas"]
    let parts: Vec<&str> = scope.split('.').collect();
    
    // Build cumulative chain
    // ["corporate", "corporate.berlin", "corporate.berlin.christmas"]
    for i in 0..parts.len() {
        let theme_name = parts[0..=i].join(".");
        chain.push(theme_name);
    }
    
    chain
}
```

## Multi-Dimensional Themes

EPC unterstützt beliebig viele Dimensionen:

```
corporate                      → Base theme
corporate.berlin               → Location-specific
corporate.christmas            → Seasonal variation
corporate.berlin.christmas     → Combined dimensions
corporate.berlin.christmas.b2b → Multi-dimensional
```

Jede Dimension fügt eine neue Ebene in der Resolution Chain hinzu.

## File Type Resolution

EPC funktioniert für alle Dateitypen gleich:

```rust
// Resolve any file type through EPC
let css_file = resolve_explicit_path_chain("hero-banner", "hero-banner.css", &theme_chain);
let js_file = resolve_explicit_path_chain("hero-banner", "hero-banner.js", &theme_chain);
let tera_file = resolve_explicit_path_chain("hero-banner", "hero-banner.tera", &theme_chain);
```

## Hot Reload Integration

EPC ist vollständig in das Hot Reload System integriert:

```rust
impl UniversalHotReload {
    async fn handle_file_change(&mut self, changed_file: PathBuf) -> Result<(), Error> {
        // Extract info from changed file
        let snippet_name = self.extract_snippet_name(&changed_file);
        let file_name = changed_file.file_name().unwrap().to_str().unwrap();
        
        // Use EPC to resolve current winning file
        let current_theme_chain = self.get_current_theme_chain().await;
        let winning_file = resolve_explicit_path_chain(&snippet_name, file_name, &current_theme_chain);
        
        // Only trigger action if changed file is the winning file in EPC
        if winning_file.as_ref() == Some(&changed_file) {
            self.execute_file_type_action(&changed_file).await?;
        }
    }
}
```

## Feature Toggle Power

EPC ermöglicht Feature Toggles ohne Code-Änderungen:

```rust
// EPC automatically resolves features based on scope
pub async fn render_page_with_features(user_id: &str) -> Result<String, Error> {
    // Get user's feature scope
    let scope = FEATURE_TOGGLE.get_user_scope(user_id).await?;
    
    // Build EPC chain from UCG + scope
    let theme_chain = UCG.build_theme_chain(&scope).await?;
    
    // EPC resolves files - new features automatically served!
    let template_context = build_context_with_epc(&theme_chain).await?;
    
    render_template_with_context(template_context).await
}
```

## Directory Structure Example

```
themes/
├── corporate/                           # Base theme
│   └── hero-banner/
│       ├── hero-banner.tera
│       ├── hero-banner.css
│       └── hero-banner.js
├── corporate.berlin/                    # Location override
│   └── hero-banner/
│       └── hero-banner.css              # Only CSS override
└── corporate.berlin.christmas/          # Seasonal override
    └── hero-banner/
        ├── hero-banner.tera             # Template override
        └── hero-banner.css              # CSS override
```

## Performance Characteristics

- **Resolution Speed:** < 0.01ms pro Datei (filesystem cache)
- **Theme Chain Building:** O(n) wo n = Anzahl der Dimensionen
- **Cache Strategy:** Theme chains werden pro Request gecached

## Benefits

- **Zero Configuration:** Keine XML/YAML Theme-Inheritance Files
- **Dynamic Scoping:** Neue Themes automatisch in Chain integriert
- **Partial Overrides:** Nur geänderte Files überschreiben
- **Clean Separation:** Jedes Theme isoliert in eigenem Ordner
- **Feature Toggles:** Scope-basierte Features ohne Conditional Logic

## Zusammenfassung

Das EPC System bindet das "File Chaos" durch:
- **Explizite Pfade:** Keine Magie, nur klare Datei-Resolution
- **UCG-Integration:** Theme Chains dynamisch aus Content Graph
- **Performance:** Filesystem-basiert, kein Database Lookup
- **Flexibilität:** Beliebige Theme-Dimensionen möglich

Dies macht Theme-Management in ReedCMS vorhersagbar und KISS-konform.
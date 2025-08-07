# ReedCMS Translation System - Multi-Layer i18n

## Multi-Layer Translation Architecture

ReedCMS nutzt ein flexibles Multi-Layer System mit klarer Priorität:

```
1. Global Translations (Höchste Priorität)
   └── config/translations.de_DE.csv
   
2. Theme Translations (mit Vererbung)
   ├── themes/corporate/translations.de_DE.csv
   └── themes/corporate.berlin/translations.de_DE.csv
   
3. Template Translations
   └── themes/corporate/templates/product/translations.de_DE.csv
   
4. Snippet Translations
   └── snippets/hero-banner/translations.de_DE.csv
   
5. Plugin Translations (Niedrigste Priorität)
   └── plugins/simple-seo/translations.de_DE.csv
```

**Priorität**: Global > Theme > Template > Snippet > Plugin

### Theme Translation Vererbung

Bei Theme-Hierarchien werden ALLE Ebenen geladen:
- `corporate` → Basis-Übersetzungen
- `corporate.berlin` → Ergänzt/überschreibt Basis
- `corporate.berlin.christmas` → Ergänzt/überschreibt beide

## Locale Setup & Detection

### CMS Setup Configuration

```
Languages Configuration
Global Method: ○ Header    ● Path    ○ Subdomain

Default Language: 🇩🇪 German (de_DE)

Additional Languages:
┌─────────────────────────────────────────────────────────────┐
│ [+] Add Language                                            │
├─────────────────┬─────────────────────────────────────────┤
│ Language        │ Identifier                              │
├─────────────────┼─────────────────────────────────────────┤
│ 🇺🇸 English     │ /en/ (auto-generated from method)        │
│ 🇫🇷 French      │ /fr/                                    │ 
└─────────────────┴─────────────────────────────────────────┘
```

### Locale Detection Methods

**Ein globaler Switch für ALLE Sprachen** - keine Methoden-Mischung:

```rust
pub enum LocaleMethod {
    Header,      // Browser Accept-Language Header
    Path,        // URL: /de/products, /en/products
    Subdomain,   // URL: de.site.com, en.site.com
}
```

### Fallback Chain

```
1. User-gewählte Sprache (Cookie/Session)
2. URL/Subdomain/Header (je nach Method)
3. Default Language (im Setup definiert)
4. English (Ultimate Fallback)
```

## CSV Translation Files

### File-per-Language Struktur

```
config/
├── translations.de_DE.csv    # Globale deutsche Übersetzungen
├── translations.en_US.csv    # Globale englische Übersetzungen
└── translations.fr_FR.csv    # Globale französische Übersetzungen

themes/corporate/
├── translations.de_DE.csv    # Theme-spezifisch
└── translations.en_US.csv

themes/corporate.berlin/
└── translations.de_DE.csv    # Erweitert corporate

snippets/hero-banner/
├── translations.de_DE.csv    # Snippet-spezifisch
└── translations.en_US.csv
```

### CSV Format

```csv
# translations.de_DE.csv
key,value,context,updated
navigation.home,"Startseite","Main navigation",2024-01-15
navigation.about,"Über uns","Main navigation",2024-01-15
form.submit,"Absenden","Generic form button",2024-01-16
hero.welcome,"Willkommen bei {site_name}","Homepage hero",2024-01-17
```

### Wichtige Regeln

- **Nur konfigurierte Sprachen**: Keine eu_ES.csv wenn Baskisch nicht im Setup
- **Optionale Files**: Nicht jede Sprache muss jedes File haben
- **UTF-8 Encoding**: Immer UTF-8 für Umlaute/Sonderzeichen
- **Variablen**: {variable_name} Syntax für dynamische Inhalte

## Tera Template Integration

### Die string() Function

**Saubere Syntax ohne key= Parameter:**

```tera
{# Einfache Übersetzungen #}
<nav>
    <a href="/">{{ string('navigation.home') }}</a>
    <a href="/about">{{ string('navigation.about') }}</a>
</nav>

{# Mit Variablen #}
<h1>{{ string('hero.welcome', site_name=site.name) }}</h1>

{# Dynamische Keys #}
{% set button_key = "form." ~ action %}
<button>{{ string(button_key) }}</button>
```

### Tera Function Registration

```rust
pub fn register_tera_functions(tera: &mut Tera) {
    tera.register_function("string", translation_function);
}

fn translation_function(args: &HashMap<String, Value>) -> Result<Value, Error> {
    // Positional argument "0" = translation key
    let key = args.get("0")
        .and_then(|v| v.as_str())
        .ok_or_else(|| Error::msg("Missing key"))?;
    
    let locale = get_current_locale_from_context()?;
    let translation = TRANSLATION_RESOLVER.resolve(key, locale)?;
    
    // Variable replacement
    let mut result = translation;
    for (k, v) in args {
        if k != "0" {
            result = result.replace(&format!("{{{}}}", k), &v.to_string());
        }
    }
    
    Ok(Value::String(result))
}
```

## Translation Resolver

### Multi-Layer Resolution Logic

```rust
pub struct TranslationResolver {
    global: HashMap<String, HashMap<String, String>>,
    theme: HashMap<String, HashMap<String, String>>,
    snippet: HashMap<String, HashMap<String, String>>,
    plugin: HashMap<String, HashMap<String, String>>,
}

impl TranslationResolver {
    pub fn resolve(&self, key: &str, locale: &str, theme: &str) -> Result<String, Error> {
        // 1. Check global translations
        if let Some(trans) = self.global.get(locale)?.get(key) {
            return Ok(trans.clone());
        }
        
        // 2. Check theme hierarchy (e.g., corporate.berlin.christmas)
        let theme_parts: Vec<&str> = theme.split('.').collect();
        for i in (1..=theme_parts.len()).rev() {
            let theme_key = theme_parts[..i].join(".");
            if let Some(trans) = self.theme.get(&format!("{}.{}", theme_key, locale))?.get(key) {
                return Ok(trans.clone());
            }
        }
        
        // 3. Check snippet translations
        if let Some(trans) = self.snippet.get(&format!("{}.{}", locale, key))?.get(key) {
            return Ok(trans.clone());
        }
        
        // 4. Check plugin translations
        if let Some(trans) = self.plugin.get(&format!("{}.{}", locale, key))?.get(key) {
            return Ok(trans.clone());
        }
        
        // 5. Fallback to key
        Ok(format!("[{}]", key))
    }
}
```

## File Discovery & Loading

### Automatische Discovery

```rust
pub async fn discover_translation_files() -> Result<TranslationMap, Error> {
    let mut translations = TranslationMap::new();
    let configured_locales = get_configured_locales().await?;
    
    for locale in configured_locales {
        // 1. Global translations
        let global_path = format!("config/translations.{}.csv", locale);
        if Path::new(&global_path).exists() {
            load_csv_translations(&global_path, &mut translations.global)?;
        }
        
        // 2. Theme translations (mit Hierarchie)
        for entry in glob("themes/*/translations.{}.csv", &locale)? {
            let theme_name = extract_theme_name(&entry)?;
            load_csv_translations(&entry, &mut translations.theme)?;
        }
        
        // 3. Snippet translations
        for entry in glob("snippets/*/translations.{}.csv", &locale)? {
            let snippet_name = extract_snippet_name(&entry)?;
            load_csv_translations(&entry, &mut translations.snippet)?;
        }
        
        // 4. Plugin translations
        for entry in glob("plugins/*/translations.{}.csv", &locale)? {
            let plugin_name = extract_plugin_name(&entry)?;
            load_csv_translations(&entry, &mut translations.plugin)?;
        }
    }
    
    Ok(translations)
}
```

### CSV Parser

```rust
pub fn load_csv_translations(path: &str, target: &mut HashMap<String, String>) -> Result<(), Error> {
    let mut reader = csv::Reader::from_path(path)?;
    
    for record in reader.records() {
        let record = record?;
        if record.len() >= 2 {
            let key = record[0].to_string();
            let value = record[1].to_string();
            target.insert(key, value);
        }
    }
    
    Ok(())
}
```

## Hot Reload Support

### File Watcher Integration

```rust
pub async fn watch_translation_files() -> Result<(), Error> {
    let (tx, rx) = channel();
    let mut watcher = notify::watcher(tx, Duration::from_millis(500))?;
    
    // Watch all translation directories
    watcher.watch("config/", RecursiveMode::NonRecursive)?;
    watcher.watch("snippets/", RecursiveMode::Recursive)?;
    watcher.watch("plugins/", RecursiveMode::Recursive)?;
    
    loop {
        match rx.recv() {
            Ok(event) => {
                if is_translation_file(&event.path) {
                    reload_translations().await?;
                    notify_hot_reload("translations").await?;
                }
            }
            Err(e) => error!("Watch error: {}", e),
        }
    }
}

fn is_translation_file(path: &Path) -> bool {
    path.file_name()
        .and_then(|n| n.to_str())
        .map(|n| n.starts_with("translations.") && n.ends_with(".csv"))
        .unwrap_or(false)
}
```

## Performance Optimierung

### Caching Strategy

```rust
pub struct TranslationCache {
    // locale -> key -> translation
    cache: Arc<RwLock<HashMap<String, HashMap<String, String>>>>,
    // Track file modification times
    file_stamps: Arc<RwLock<HashMap<String, SystemTime>>>,
}

impl TranslationCache {
    pub async fn get_or_load(&self, key: &str, locale: &str) -> Result<String, Error> {
        // Fast path: Check cache
        {
            let cache = self.cache.read().await;
            if let Some(locale_cache) = cache.get(locale) {
                if let Some(translation) = locale_cache.get(key) {
                    return Ok(translation.clone());
                }
            }
        }
        
        // Slow path: Load and cache
        self.ensure_locale_loaded(locale).await?;
        
        self.cache.read().await
            .get(locale)
            .and_then(|lc| lc.get(key))
            .map(|t| t.clone())
            .ok_or_else(|| Error::MissingTranslation(key.to_string()))
    }
}
```

### Bulk Loading

Beim Seitenaufruf werden alle benötigten Translations auf einmal geladen:
- Extrahiert alle Keys aus Template und Snippets
- Lädt alle Übersetzungen in einem Batch
- Cached für nachfolgende Zugriffe

## Praktische Beispiele

### Multi-Language Navigation

```tera
{# navigation.tera #}
<nav class="main-nav">
    <ul>
        <li><a href="/">{{ string('nav.home') }}</a></li>
        <li><a href="/{{ locale }}/products">{{ string('nav.products') }}</a></li>
        <li><a href="/{{ locale }}/about">{{ string('nav.about') }}</a></li>
    </ul>
    
    {# Language Switcher #}
    <div class="language-switcher">
        {% for lang in available_languages %}
        <a href="{{ current_path | with_locale(lang.code) }}" 
           class="{% if lang.code == locale %}active{% endif %}">
            {{ lang.flag }} {{ lang.name }}
        </a>
        {% endfor %}
    </div>
</nav>
```

### Form mit Validierung

```tera
{# contact-form.tera #}
<form method="post">
    <label>{{ string('form.name.label') }}</label>
    <input type="text" name="name" required>
    {% if errors.name %}
        <span class="error">{{ string(errors.name) }}</span>
    {% endif %}
    
    <button type="submit">{{ string('form.submit') }}</button>
</form>
```

## Admin UI Integration

### Translation Editor für Redakteure

Das Admin UI bietet kontextabhängige Translation-Editoren:

**1. Theme-Level Editor**
- Zeigt alle Übersetzungen des Themes und seiner Hierarchie
- Tabs für jede konfigurierte Sprache
- Direkte CSV-Bearbeitung mit Speichern-Button
- Neue Keys und Sprachen hinzufügbar

**2. Template-Level Editor**
- Zeigt nur die im Template verwendeten Translation-Keys
- Von Theme/Global überschriebene Werte sind ausgegraut
- Info-Button erklärt warum Werte nicht editierbar sind
- Template-Translations nur sichtbar wenn höhere Layer den Key nicht definieren

**3. Snippet-Level Editor**
- Zeigt Translation-Keys des Snippets
- Global/Theme/Template Werte sind ausgegraut mit Layer-Indikator
- Klare Hierarchie-Visualisierung: 🌍 > 🎨 > 📄 > 📦
- Warnung: Snippet-Translations nur bei standalone Nutzung aktiv

**Wichtigste UI-Regel**: Eine niedrigere Ebene kann NIEMALS eine höhere überschreiben. Der Editor zeigt dies durch:
- Ausgegraute, nicht-editierbare Felder
- Visuelle Layer-Indikatoren
- Info-Dialoge statt Override-Optionen

## Zusammenfassung

ReedCMS Translation System bietet:

1. **Multi-Layer Priority**: Global > Theme > Template > Snippet > Plugin
2. **Theme Vererbung**: Hierarchische Theme-Translations
3. **Clean Tera Syntax**: `{{ string('key') }}` ohne Boilerplate
4. **UI Editor**: Grafische CSV-Bearbeitung für Redakteure
5. **Context-Aware**: Template-spezifische Translation Views
6. **File-per-Language**: Übersichtliche CSV-Struktur
7. **Hot Reload**: Änderungen sofort sichtbar
8. **Performance**: Intelligentes Caching

Das System ist KISS-konform: Ein Key, multiple Quellen, klare Priorität, einfache UI-Bearbeitung.

### Layer-Visualisierung im UI

```
🌍 Global    → Immer editierbar, höchste Priorität
🎨 Theme     → Ausgegraut in Template/Snippet Views  
📄 Template  → Ausgegraut in Snippet View
📦 Snippet   → Nur in eigener View editierbar
🔌 Plugin    → Niedrigste Priorität
```

Keine Ebene kann eine höhere Ebene überschreiben - Übersetzungen werden nur verwendet, wenn höhere Ebenen den Key nicht definieren.
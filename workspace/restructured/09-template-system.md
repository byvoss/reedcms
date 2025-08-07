# ReedCMS Template System - Tera + Web Components + CSS Layers

## CSS Layer Architecture - Das zentrale Konzept

ReedCMS nutzt eine **4-Layer CSS Architektur** für absolute Kontrolle und Vorhersagbarkeit:

```css
/* ReedCMS Layer-Hierarchie - IMMER in dieser Reihenfolge */
@layer kernel, bridge, snippet, theme;
```

### Layer-Definitionen

1. **kernel** - ReedCMS Core Styles (von Rust generiert)
   - Layer-Definition (@layer kernel, bridge, snippet, theme)
   - Reset Styles
   - Core Layout System
   - ReedCMS Variablen
   - Nie direkt modifiziert

2. **bridge** - Externe Libraries (gekapselt)
   ```css
   @layer bridge {
       @layer bootstrap {
           /* Bootstrap Framework */
       }
       @layer tailwind {
           /* Tailwind Utilities */
       }
   }
   ```

3. **snippet** - Component Defaults
   ```css
   @layer snippet {
       hero-banner {
           /* Default Component Styles */
       }
   }
   ```

4. **theme** - Developer Overrides
   ```css
   @layer theme {
       /* Custom Project Styles */
   }
   ```

### Warum CSS Layers?

- **Vorhersagbare Spezifität**: kernel < bridge < snippet < theme
- **Zero Conflicts**: Externe Libraries in bridge gekapselt
- **Developer Control**: Theme Layer überschreibt ALLES
- **Performance**: Browser-native Cascade Control

### Kernel CSS - Von Rust generiert

```css
/* /assets/kernel.css - Systemimmanent */
@layer kernel, bridge, snippet, theme;

@layer kernel {
    :root {
        --reed-spacing-unit: 0.25rem;
        --reed-font-base: system-ui, sans-serif;
        --color-surface: #fff;
    }
    *, *::before, *::after { box-sizing: border-box; }
    body { margin: 0; font-family: var(--reed-font-base); }
}
```

## Template Engine - Tera Integration

### Basis-Template (Kernel - Nicht editierbar)

```tera
{# kernel/base.tera - ReedCMS Foundation, nicht editierbar #}
<!DOCTYPE html>
<html lang="{{ locale }}">
<head>
    <meta charset="UTF-8">
    <title>{{ page.title }} - {{ site_name }}</title>
    
    {# Kernel CSS - Von Rust generiert, enthält Layer-Definition #}
    <link rel="stylesheet" href="/assets/kernel.css">
    
    {# Bridge Layer - Externe Libraries #}
    {% block bridge_css %}
    <link rel="stylesheet" href="/assets/bridge/bootstrap.css">
    {% endblock %}
    
    {# Snippet CSS - Component Defaults #}
    <link rel="stylesheet" href="/assets/snippets.css">
    
    {# Theme CSS - Project Overrides #}
    <link rel="stylesheet" href="/assets/theme.css">
    
    {# WCAG Detection - Jeder Seitenaufruf #}
    <script>
    (function() {
        const detected = window.speechSynthesis?.getVoices().length > 0;
        window.reedReader = { active: detected };
        document.cookie = `reed_reader=${detected ? 'active' : 'inactive'}; path=/; max-age=31536000; SameSite=Lax`;
    })();
    </script>
</head>
<body>
    {% block content %}{% endblock %}
    
    {# Web Components laden #}
    <script type="module" src="/assets/components.js"></script>
</body>
</html>
```

### Template Verzeichnisstruktur

```
kernel/                           # ReedCMS Core - Nicht editierbar
├── base.tera                     # Basis-Template mit allen Essentials
└── error.tera                    # Error Pages

snippets/                         # Content Logic (Templates + Assets)
├── hero-banner/
│   ├── hero-banner.tera         # Snippet Template
│   ├── hero-banner.js           # Web Component
│   └── hero-banner.css          # Base Styles
└── product-card/
    ├── product-card.tera
    ├── product-card.js
    └── product-card.css

themes/[theme-name]/              # Theme-spezifische Assets
├── templates/                   # Theme-eigene Tera Templates
│   ├── page.tera               # Page Template
│   ├── article.tera            # Article Template
│   └── landing.tera            # Landing Template
└── snippets/                    # Visual Overrides für Snippets
    └── hero-banner/
        ├── hero-banner.css      # Theme-spezifische Styles
        └── hero-banner.js       # Theme-spezifisches Verhalten
```

### Developer Templates (Editierbar)

Entwickler arbeiten in themes/ mit eigenen .tera Templates:

```tera
{# themes/corporate/templates/page.tera - Theme-eigenes Template #}
{% extends "kernel/base.tera" %}

{% block content %}
<main class="corporate-layout">
    {# Meta-Snippet mit allen Eigenschaften aus CMS #}
    {{ render_snippet(page_meta_snippet) }}
    
    {# Oder freies Komponieren mit einzelnen Snippets #}
    {% for snippet in page.snippets %}
        {{ render_snippet(snippet) }}
    {% endfor %}
</main>
{% endblock %}
```

### Snippet-Erstellung im CMS

Snippets werden **ausschließlich über das Admin-UI** erstellt:

1. **Einfache Snippets**: Felder werden zu Einheiten gebündelt
   - Automatische Web Component Generation
   - Speicherung in `snippets/[name]/`

2. **Meta-Snippets**: Snippets in Snippets komponiert
   - Snippet + Felder + Snippet + ...
   - Für größere Blöcke oder ganze Seiten
   - Eigenschaften werden im CMS definiert

### Theme-Template Verwendung

```tera
{# themes/berlin/templates/article.tera - Artikel-Template für Berlin #}
{% extends "kernel/base.tera" %}

{% block content %}
    {# Meta-Snippet direkt übergeben mit allen CMS-Eigenschaften #}
    {{ render_snippet(article_layout_snippet) }}
{% endblock %}
```

```tera
{# themes/corporate/templates/landing.tera - Freies Komponieren #}
{% extends "kernel/base.tera" %}

{% block content %}
    {# Freie Komposition einzelner Snippets #}
    {{ render_snippet(hero_banner) }}
    
    <section class="features">
        {% for feature in feature_snippets %}
            {{ render_snippet(feature) }}
        {% endfor %}
    </section>
    
    {{ render_snippet(cta_section) }}
{% endblock %}
```

### Template & Snippet Management

**WICHTIG**: Snippets und Templates werden IMMER via UI oder CLI angelegt:

```bash
# Template global erstellen mit initialer Theme-Zuweisung
reed template create --name=product --theme=corporate

# Mit spezifischem Scope:
reed template create --name=product --theme=corporate.berlin.christmas

# Template später umziehen/kopieren
reed template assign --name=product --from=corporate --to=berlin
reed template copy --name=product --from=corporate --to=corporate.summer

# Erstellt automatisch:
# - themes/[theme]/templates/product.tera
# - Globale Registrierung im 4-Layer System (CSV)
# - Template bleibt global verfügbar für Neuzuweisungen
```

Templates sind **global registriert** und definieren die **Struktur**:
- Template = Die strukturelle Tera-Datei (was wo angezeigt wird)
- Theme = Das Styling und kleine Anpassungen (wie es aussieht)
- Ein Template kann von mehreren Themes genutzt werden
- Themes passen Templates visuell an, ändern aber nicht die Struktur

### Seiten-Erstellung im CMS

Beim Anlegen einer Seite im CMS wird ein Template zugewiesen - dieses kann aber **beliebig wiederverwendet** werden:

```
Beispiel: Multi-Standort mit einheitlichem Look
├── /berlin/products → Template: "product-grid"
├── /munich/products → Template: "product-grid" (gleiches Template!)
├── /hamburg/products → Template: "product-grid-harbor" (leicht angepasst)
```

### Template-Vererbung via UCG

```bash
# Neues Template mit Vererbung anlegen
reed template create --name=product-grid-harbor --inherits=product-grid

# Im UI: "Erbt von: [Autovervollständigung mit allen Templates]"
```

```csv
# template_associations.csv - Vererbung in UCG
parent_template,child_template,inheritance_type,override_sections
product-grid,product-grid-harbor,extends,"header,footer"
```

```tera
{# themes/hamburg/templates/product-grid-harbor.tera #}
{% extends "themes/corporate/templates/product-grid.tera" %}
{% block header %}
    {{ super() }} {# 90% vom Parent #}
    <div class="wave-animation"></div>
{% endblock %}
```

Die Template-Auflösung bietet zwei Wege:

**1. Dediziertes Theme-Template:**
- Theme: `corporate.berlin.christmas`
- Template: `product`
- Pfad: `themes/corporate.berlin.christmas/templates/product.tera`

**2. Basis-Theme mit In-Template Scoping:**
- Theme: `corporate`
- Template: `product`
- Pfad: `themes/corporate/templates/product.tera`
- Scope-Kontrolle im Template selbst:

```tera
{# themes/corporate/templates/product.tera #}
{% extends "kernel/base.tera" %}

{% block content %}
    {# Scope-basierte Snippet-Auswahl im Template #}
    {% if page.scope contains "berlin" %}
        {% include "snippets(berlin)" with snippet=hero %}
    {% elif page.scope contains "christmas" %}
        {% include "snippets(christmas)" with snippet=hero %}
    {% else %}
        {{ render_snippet(hero) }}
    {% endif %}
{% endblock %}
```

Beide Ansätze sind valide:
- **Dediziert**: Klare Trennung, mehr Dateien
- **Flexibel**: Weniger Dateien, Logik im Template

### Context Scoping System

ReedCMS Templates unterstützen multi-dimensionale Scopes:

```tera
{# Scope-basierte Template Inklusion #}
{% include "snippets(berlin.christmas.b2b)" with snippet=child %}
```

**Fallback Chain:**
1. `hero-banner.berlin.christmas.b2b.tera`
2. `hero-banner.berlin.christmas.tera`
3. `hero-banner.berlin.b2b.tera`
4. `hero-banner.berlin.tera`
5. `hero-banner.christmas.tera`
6. `hero-banner.b2b.tera`
7. `hero-banner.tera`

## Web Components - @component Directive

### Component Definition in Tera

```tera
{# @component name: "hero-banner" attributes: ["title", "subtitle", "theme"] slots: ["actions"] #}
<hero-banner 
    title="{{ snippet.content.title | escape }}"
    theme="{{ snippet.content.theme | default('default') }}"
    scope="{{ scope }}">
    
    <div slot="actions">
        {% if snippet.content.cta_text %}
        <button>{{ snippet.content.cta_text }}</button>
        {% endif %}
    </div>
</hero-banner>
```

### Rust-generierte Web Component

```javascript
// Von Rust aus @component generiert
export class HeroBannerSnippet extends ReedSnippet {
    static get observedAttributes() {
        return ['title', 'subtitle', 'theme', 'alignment'];
    }
    
    render() {
        const { title, subtitle, theme, alignment } = this.getAttributes();
        this.shadowRoot.innerHTML = `
            <style>
                @layer snippet {
                    :host { display: block; text-align: var(--alignment, center); }
                    .hero-container { padding: var(--hero-padding, 2rem); }
                }
            </style>
            <div class="hero-container" theme="${theme}">
                <h1>${title}</h1>
                <p>${subtitle}</p>
                <slot name="actions"></slot>
            </div>
        `;
        this.style.setProperty('--alignment', alignment);
    }
}
customElements.define('hero-banner', HeroBannerSnippet);
```

### Component CSS mit Layers

```css
@layer snippet {
    hero-banner {
        --hero-padding: 3rem 1rem;
        --hero-bg: var(--color-surface);
        display: block;
    }
    
    hero-banner[theme="dark"] { --hero-bg: var(--color-surface-dark); }
    hero-banner[theme="christmas"] { --hero-bg: linear-gradient(135deg, #1e3c72, #2a5298); }
    
    /* Breakpoint Classes */
    .phone hero-banner { --hero-padding: 2rem 1rem; }
    .wcag hero-banner { --hero-padding: 2rem 1rem; }
}
```

## Template Functions

### String Translation Function

```tera
{# Saubere Syntax ohne key= #}
{{ string('navigation.home') }}
{{ string('welcome.title') }}
```

### Snippet Rendering

```tera
{# Custom Tera Function für Snippets #}
{{ render_snippet(child) }}
```

### JSON Encoding für Web Components

```tera
{# Sicheres JSON für Component-Attributes #}
config="{{ config | json_encode | escape_html_attr }}"
```

## Progressive Enhancement Pattern

```tera
{# @component name: "accordeon-snippet" #}
<accordeon-snippet title="{{ snippet.content.title | escape }}">
    {# Fallback ohne JS #}
    <details>
        <summary>{{ snippet.content.title }}</summary>
        {% for item in snippet.content.items %}
            <h3>{{ item.title }}</h3>
            <p>{{ item.content }}</p>
        {% endfor %}
    </details>
    
    {# Enhanced mit JS #}
    <template slot="content">
        {% for item in snippet.content.items %}
        <accordeon-item title="{{ item.title | escape }}"></accordeon-item>
        {% endfor %}
    </template>
</accordeon-snippet>
```

## 4-Layer Template Registration

```csv
# templates.csv - Globale Registry
template_name,template_type,description,active
page,standard,"Standard page template",true
article,blog,"Blog article template",true
product,ecommerce,"Product detail template",true

# template_assignments.csv - Theme-Zuweisungen
template_name,theme_name,implementation_path
page,corporate,themes/corporate/templates/page.tera
page,berlin,themes/berlin/templates/page.tera
```

## Template Context Building

```rust
// 4-Layer Integration für Templates
pub struct TemplateContext {
    page: PageWithContent,
    children: Vec<ChildWithContent>,
    locale: String,
    scope: String,
    theme: String,
    template: String,
}

impl TemplateContext {
    pub async fn build_from_route(route: &str, locale: &str) -> Result<Self, TemplateError> {
        // Redis: Page Structure (sub-millisecond)
        let page_structure = resolve_page_structure_by_slug(route).await?;
        
        // PostgreSQL UCG: Association IDs
        let association_ids = collect_association_ids(&page_structure);
        
        // PostgreSQL Main: Content Data
        let content_map = batch_fetch_content(&association_ids, locale).await?;
        
        // Merge für Template Context
        let page = PageWithContent::merge(page_structure, &content_map)?;
        let children = ChildWithContent::merge_vec(hierarchy, &content_map)?;
        
        Ok(TemplateContext {
            page,
            children,
            locale: locale.to_string(),
            scope: determine_scope(&page),
            theme: page.theme.clone(),
            template: page.template.clone(),
        })
    }
}
```

## DOM Output Beispiel

```html
<hero-banner title="Willkommen" theme="dark" scope="berlin.christmas">
    #shadow-root (open)
        <style>@layer snippet { /* Component styles */ }</style>
        <div class="hero-container" theme="dark">
            <h1>Willkommen</h1>
            <slot name="actions"></slot>
        </div>
    <div slot="actions"><button>Jetzt starten</button></div>
</hero-banner>
```

## Development Workflow

### Hot Reload mit File Watchers

```rust
match path.extension().and_then(|ext| ext.to_str()) {
    Some("tera") => {
        TERA_ENGINE.clear_template_cache();
        self.websocket.send_page_refresh().await?;
    },
    Some("css") => self.websocket.send_css_refresh().await?,
    Some("js") => self.websocket.send_page_refresh().await?,
    _ => {}
}
```

### Template Sicherheit

- Circular Dependency Detection zur Build-Zeit
- Tera Syntax Validation vor Deployment
- Scope Collision Prevention zwischen Themes
- Automatic HTML Escaping in allen Outputs

## Zusammenfassung

ReedCMS Template System kombiniert:

1. **CSS Layer Architecture** für vorhersagbare Styles
2. **Tera Templates** für Server-Side Rendering
3. **Web Components** via @component Directive
4. **Context Scoping** für flexible Template-Auswahl
5. **Progressive Enhancement** für Accessibility
6. **4-Layer Dataflow** für Performance

Das Resultat: Ein Template System das gleichzeitig simpel und mächtig ist, mit Zero JavaScript Boilerplate und vollständiger Developer Control.
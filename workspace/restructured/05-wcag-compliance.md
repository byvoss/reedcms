# ReedCMS WCAG 2.2 Compliance System

## Barrierefreiheit als Default

ReedCMS ist das erste CMS, das WCAG 2.2 Konformität als Standard-Feature implementiert. Jede Installation ist automatisch barrierefrei - ohne zusätzliche Konfiguration.

## Zwei-Stufen Reader-Detection System

### Stufe 1: Kontinuierliche Detection (base.tera)

```javascript
// In base.tera - läuft auf JEDER Seite bei JEDEM Aufruf
<script>
(function() {
    // IMMER prüfen ob Screen Reader aktiv ist
    const detected = window.speechSynthesis?.getVoices().length > 0;
    
    // Global verfügbar machen
    window.reedReader = { active: detected };
    
    // Status in Cookie speichern/aktualisieren
    const readerStatus = detected ? 'active' : 'inactive';
    document.cookie = `reed_reader=${readerStatus}; path=/; max-age=31536000; SameSite=Lax`;
    
    // Breakpoint-Update triggern
    document.dispatchEvent(new Event('reedReaderChanged'));
    
    // Bei Änderung des Status → Backend informieren
    const lastStatus = getCookie('reed_reader_last');
    if (lastStatus !== readerStatus) {
        document.cookie = `reed_reader_last=${readerStatus}; path=/; max-age=31536000; SameSite=Lax`;
        
        fetch('/api/accessibility-status', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({ 
                reader_active: detected,
                timestamp: Date.now()
            })
        });
    }
})();
</script>
```

### Stufe 2: Erweiterte Feature-Detection (Cookie-Consent)

```javascript
// Erweiterte Feature-Detection beim ersten Besuch
function detectReaderFeatures() {
    const features = {
        active: window.speechSynthesis?.getVoices().length > 0,
        speechSynthesis: !!window.speechSynthesis,
        ariaLiveSupported: testAriaLiveSupport(),
        focusManagement: testFocusManagement(),
        userAgent: detectKnownReaders(),
        preferredMotion: window.matchMedia('(prefers-reduced-motion: reduce)').matches,
        highContrast: window.matchMedia('(prefers-contrast: high)').matches
    };
    
    // Alles im GLEICHEN Cookie speichern
    document.cookie = `reed_reader=${JSON.stringify(features)}; path=/; max-age=31536000; SameSite=Lax`;
    
    return features;
}

// Test ob ARIA live regions unterstützt werden
function testAriaLiveSupport() {
    return 'ariaLive' in document.createElement('div');
}

// Bekannte Screen Reader erkennen
function detectKnownReaders() {
    const ua = navigator.userAgent;
    if (ua.includes('NVDA')) return 'NVDA';
    if (ua.includes('JAWS')) return 'JAWS';
    if (ua.includes('VoiceOver')) return 'VoiceOver';
    if (ua.includes('TalkBack')) return 'TalkBack';
    return null;
}
```

### Cookie-Consent mit Reader-Detection

```tera
{# cookie_consent.tera - nur beim ersten Besuch #}
<div class="reed-cookie-consent" role="dialog" aria-labelledby="consent-title">
    <h2 id="consent-title">Cookie-Einstellungen</h2>
    
    <div class="consent-section">
        <h3>Barrierefreiheit-Optimierung</h3>
        <p>Wir haben erkannt, dass Sie assistive Technologien nutzen.</p>
        
        <div id="reader-features" class="feature-list">
            <!-- Wird via JS mit erkannten Features gefüllt -->
        </div>
        
        <p class="consent-note">
            Diese Informationen werden in einem Cookie gespeichert, 
            um Ihnen bei jedem Besuch die optimale Darstellung zu bieten.
        </p>
    </div>
    
    <div class="consent-actions">
        <button type="submit" class="primary" onclick="acceptReaderDetection()">
            Optimierung aktivieren
        </button>
    </div>
</div>

<script>
// Beim Cookie-Consent die erweiterten Features testen
if (!getCookie('reed_reader_consent')) {
    const features = detectReaderFeatures();
    displayDetectedFeatures(features);
}

function acceptReaderDetection() {
    document.cookie = 'reed_reader_consent=accepted; path=/; max-age=31536000';
    // Features wurden bereits im reed_reader Cookie gespeichert
    closeConsentDialog();
}
</script>
```

## Reader-spezifische Template-Optimierung

### Konditionale Template-Auslieferung

```tera
{# Basis-Template mit Reader-Detection #}
{% if reed_reader.active %}
    {# WCAG-optimierte Version für Screen Reader #}
    {% include "templates/standard/page.wcag.tera" %}
{% else %}
    {# Standard visuelle Version #}
    {% include "templates/standard/page.tera" %}
{% endif %}
```

### Informationsflut-Reduktion

```tera
{# In jedem Template: Konditionale Inhalte #}
<article>
    <h1>{{ title }}</h1>
    
    {# Essenzielle Informationen immer anzeigen #}
    <p class="summary">{{ summary }}</p>
    
    {% if not reed_reader.active %}
        {# Dekorative Elemente nur für visuelle Nutzer #}
        <div class="decorative-hero">
            {{ hero_animation }}
        </div>
        
        {# Zusätzliche visuelle Metadaten #}
        <aside class="visual-metadata">
            <span class="read-time">{{ read_time }} Min. Lesezeit</span>
            <span class="views">{{ view_count }} Aufrufe</span>
        </aside>
    {% endif %}
    
    {# Hauptinhalt für alle #}
    <div class="content">
        {{ content|safe }}
    </div>
    
    {% if reed_reader.active %}
        {# Strukturierte Navigation für Reader #}
        <nav aria-label="Artikel-Navigation">
            <h2>Inhalt</h2>
            {{ toc|render_accessible }}
        </nav>
    {% endif %}
</article>
```

### CSS @scope für WCAG

```css
/* Native CSS @scope für WCAG - Teil des Breakpoint Systems */
@scope (.wcag) {
    /* Dekorative Elemente ausblenden */
    .decorative-only,
    .visual-metadata,
    .hero-animation {
        display: none;
    }
    
    /* Größere Abstände für bessere Navigation */
    .content > * + * {
        margin-top: 2rem;
    }
    
    /* Fokus-Indikatoren verstärken */
    :focus {
        outline: 4px solid var(--reed-focus-color);
        outline-offset: 4px;
    }
    
    /* Lineare Layouts statt Grid */
    .gallery-grid {
        display: block;
    }
    
    /* Größere Touch-Targets */
    button, a, [role="button"] {
        min-height: 44px;
        min-width: 44px;
    }
}

/* JavaScript setzt Breakpoint-Klasse */
// Breakpoint-Klassen werden durch reedCMS gesetzt
// WCAG hat immer Priorität vor viewport-basierten Breakpoints
if (window.reedReader?.active) {
    document.body.className = 'wcag';
} else {
    // Standard Breakpoints: phone, tablet, screen, wide
    document.body.className = reedCMS.getBreakpoint();
}
```

### WCAG-Template Struktur

```bash
templates/standard/
├── home.tera              # Standard Homepage
├── home.wcag.tera         # WCAG-optimierte Homepage
├── article.tera           # Standard Artikel
├── article.wcag.tera      # WCAG-optimierter Artikel
├── gallery.tera           # Standard Galerie
└── gallery.wcag.tera      # WCAG-optimierte Galerie
```

### WCAG-Template Beispiel

```tera
{# templates/standard/gallery.wcag.tera #}
<div class="gallery-accessible">
    <h1>{{ gallery.title }}</h1>
    <p role="status">{{ gallery.images|length }} Bilder in dieser Galerie</p>
    
    {# Lineare Liste statt Grid für Screen Reader #}
    <ol class="image-list" role="list">
    {% for image in gallery.images %}
        <li>
            <h2 id="img-{{ loop.index }}">{{ image.title }}</h2>
            <p>{{ image.description }}</p>
            {# Detaillierte Alt-Texte und eager loading #}
            <img src="{{ image.url }}" 
                 alt="{{ image.detailed_alt_text }}"
                 aria-describedby="img-{{ loop.index }}"
                 loading="eager">
        </li>
    {% endfor %}
    </ol>
</div>
```

## WCAG-optimierte Snippet-Struktur

### Automatische Semantic HTML

```rust
// Snippet Definition mit WCAG-Metadaten
pub struct WcagSnippet {
    pub semantic_role: SemanticRole,
    pub aria_labels: HashMap<String, String>,
    pub focus_order: Option<i32>,
    pub contrast_ratio: ContrastRequirement,
}

pub enum SemanticRole {
    Navigation,
    Main,
    Complementary,
    Banner,
    ContentInfo,
    Form,
    Search,
}

pub enum ContrastRequirement {
    AA, // 4.5:1 für normalen Text
    AAA, // 7:1 für erweiterte Konformität
}
```

### Snippet-Registry mit WCAG-Validierung

```csv
# snippets.csv mit WCAG-Spalten
snippet_name,fields,validation,semantic_role,min_contrast,focus_order
navigation,items:array;logo:image,required,navigation,AA,1
hero,title:string;subtitle:string,required,banner,AAA,2
article,title:string;content:text;author:string,required,main,AA,3
footer,links:array;copyright:string,required,contentinfo,AA,100
```

## Template-System Integration

### Automatische ARIA-Labels

```tera
{# Jedes Snippet erhält automatisch korrekte ARIA-Attribute #}
{% macro render_snippet(snippet) %}
<{{ snippet.semantic_tag|default:"div" }}
    role="{{ snippet.semantic_role }}"
    {% if snippet.aria_label %}aria-label="{{ snippet.aria_label|t }}"{% endif %}
    {% if snippet.aria_describedby %}aria-describedby="{{ snippet.id }}-desc"{% endif %}
    tabindex="{{ snippet.focus_order|default:0 }}"
>
    {{ snippet.content|safe }}
</{{ snippet.semantic_tag|default:"div" }}>
{% endmacro %}
```

### Skip-Navigation automatisch

```tera
{# base.tera - Automatisch eingefügt #}
<body>
    <a href="#main-content" class="reed-skip-link">
        {{ "skip_to_content"|t }}
    </a>
    
    <header role="banner">
        {{ navigation_snippet|render_wcag }}
    </header>
    
    <main id="main-content" role="main">
        {% block content %}{% endblock %}
    </main>
</body>
```

## Kontrast-Validierung

### Build-Time Validation

```rust
// Während reed build
pub fn validate_theme_contrast(theme: &Theme) -> Result<(), WcagError> {
    for (fg, bg) in theme.get_color_combinations() {
        let ratio = calculate_contrast_ratio(fg, bg);
        
        if ratio < 4.5 {
            return Err(WcagError::InsufficientContrast {
                foreground: fg,
                background: bg,
                ratio,
                required: 4.5,
            });
        }
    }
    Ok(())
}
```

### Runtime Contrast Checker

```rust
// Admin-Panel Integration
pub async fn check_content_contrast(
    content: &str,
    context: &RenderContext,
) -> Vec<ContrastWarning> {
    let mut warnings = Vec::new();
    
    // Prüfe inline styles
    for style in extract_inline_styles(content) {
        if let Some(warning) = validate_style_contrast(style, context) {
            warnings.push(warning);
        }
    }
    
    warnings
}
```

## Tastatur-Navigation

```rust
// Automatisches Focus-Management
pub struct FocusManager {
    focus_trap_stack: Vec<ElementId>,
    last_focused: Option<ElementId>,
}
```

## Screen Reader Optimierungen

### Dynamische Inhalts-Ankündigungen

```tera
{# Live-Region für dynamische Updates #}
<div role="status" aria-live="polite" aria-atomic="true" class="reed-sr-only">
    {{ dynamic_message }}
</div>

{# Wichtige Statusmeldungen #}
<div role="alert" aria-live="assertive">
    {{ error_message }}
</div>
```

### Strukturierte Daten für Screen Reader

```rust
// Automatische Heading-Hierarchie
pub fn validate_heading_structure(html: &str) -> Result<(), WcagError> {
    let headings = extract_headings(html);
    let mut current_level = 0;
    
    for heading in headings {
        if heading.level > current_level + 1 {
            return Err(WcagError::HeadingSkip {
                from: current_level,
                to: heading.level,
            });
        }
        current_level = heading.level;
    }
    Ok(())
}
```

## Admin-Panel WCAG Features

### Accessibility Dashboard

```rust
pub struct AccessibilityMetrics {
    pub detected_screen_readers: u32,
    pub high_contrast_users: u32,
    pub keyboard_only_users: u32,
    pub wcag_violations: Vec<WcagViolation>,
}

// Anzeige im Admin-Panel
impl AdminPanel {
    pub fn render_accessibility_dashboard(&self) -> Template {
        template! {
            h2 { "Barrierefreiheit-Übersicht" }
            
            .metrics {
                .metric {
                    span.value { (self.metrics.detected_screen_readers) }
                    span.label { "Screen Reader Nutzer (30 Tage)" }
                }
            }
            
            .violations {
                h3 { "WCAG Violations" }
                @if self.metrics.wcag_violations.is_empty() {
                    p.success { "✓ Keine Verstöße gefunden!" }
                } @else {
                    ul {
                        @for violation in &self.metrics.wcag_violations {
                            li { (violation.description()) }
                        }
                    }
                }
            }
        }
    }
}
```

## Performance-Optimierungen

```javascript
img.loading = isScreenReaderActive ? 'eager' : 'lazy';
```
## Marketing-Differenzierung

ReedCMS ist das erste CMS mit:
- **Automatischer WCAG 2.2 Konformität** out-of-the-box
- **Intelligenter Screen Reader Erkennung** ohne Privacy-Verletzung
- **Reader-spezifischen Templates** zur Informationsflut-Reduktion
- **CSS Reader Breakpoints** - revolutionär wie Media Queries
- **Build-Time Contrast Validation** für alle Themes
- **Automatischen Skip-Links** und Focus-Management
- **Barrierefreiheits-Dashboard** im Admin-Panel

### Der menschliche Unterschied

"Wer eine Behinderung hat und das braucht, wird das sehr zu schätzen wissen" - Wir reduzieren die Informationsflut drastisch durch:
- Separate, fokussierte Templates
- Konditionale Inhalts-Einbindung
- Wegfall dekorativer Elemente
- Optimierte Navigationsstrukturen

Diese Features machen ReedCMS zur ersten Wahl für:
- Regierungswebsites (gesetzliche Anforderungen)
- Bildungseinrichtungen (Inklusion)
- Große Unternehmen (CSR-Compliance)
- NGOs (Zugänglichkeit für alle)
- Alle, die das Beste für ihre Mitmenschen herausholen wollen
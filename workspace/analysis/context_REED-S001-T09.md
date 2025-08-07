# Context Check für [REED-S001-T09] - Template System (Tera + Web Components)

## Ticket-Kontext
- **Dokument:** workspace/restructured/09-template-system.md
- **Fokus:** NUR Template-Engine und Web Component Integration
- **WICHTIG:** CSS Layer Architektur als zentrales Thema

## Input-Dokumente
1. `reedcms_06_templates.md` - Template System
2. `reedcms_qa_part2.md` - Template Integration

## Besondere Anforderungen
- CSS Layer Architektur (kernel, bridge, snippet, template)
- Bridge Layer für externe Libraries
- Tera + Web Components Combo
- Progressive Enhancement
- Ab hier ALLE CSS-Beispiele mit Layer-Pattern

## Relevante Architektur-Entscheidungen
- **CSS Layers:** @layer kernel, bridge, snippet, template;
- **Kein @scope:** Firefox Support fehlt
- **Rust generiert Web Components:** Automatisch aus @component
- **Bridge Layer:** Explizit für Bootstrap, Tailwind etc.

## Pattern aus CSS_LAYER_PATTERN.md
```css
/* ReedCMS Layer-Hierarchie */
@layer kernel, bridge, snippet, template;

/* Entwickler sehen nur bridge, snippet, template */
@layer snippet {
    hero-banner { /* Component styles */ }
}
```
# CSS Layer Pattern - Konsistenz-Referenz

## Die ReedCMS Layer-Hierarchie

```css
@layer kernel, bridge, snippet, template;
```

1. **kernel** - Von Rust vorgegeben, ReedCMS Core Styles
2. **bridge** - Externe Libraries (gekapselt in Sub-Layers)
3. **snippet** - Component Defaults aus .css Dateien
4. **template** - Developer Overrides

## Wo bereits angepasst werden muss:

### T05 - WCAG Compliance (workspace/restructured/05-wcag-compliance.md)
- Zeile 179ff: Layer-Definition anpassen
- Hinweis auf T09 für Details

### T07 - Snippet System (workspace/restructured/07-snippet-system.md)
- Zeile 410ff: Layer-Architektur bereits erklärt
- Muss auf kernel + bridge erweitert werden
- Verweis auf T09 für vollständige Erklärung

## Standard CSS-Beispiel ab jetzt:

```css
/* ReedCMS Layer-Hierarchie (siehe T09 für Details) */
@layer kernel, bridge, snippet, template;

/* Kernel wird von Rust bereitgestellt */
/* Bridge für externe Libraries */
@layer bridge {
    /* @layer bootstrap { } */
    /* @layer tailwind { } */
}

/* Snippet Layer - Component Defaults */
@layer snippet {
    /* Component styles */
}

/* Template Layer - Developer Overrides */
@layer template {
    /* Custom styles */
}
```

## Wichtig:
- T09 wird die zentrale Referenz für Layer-Architektur
- Alle anderen Tickets verweisen auf T09
- Konsistenz in ALLEN CSS-Beispielen
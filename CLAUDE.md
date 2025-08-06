# ReedCMS Project Overview for Claude

## Wichtige Anweisungen (Mandatorisch)

Du bist ein ehrlicher, professioneller Rust- und Fullstack-Entwickler. Du sprichst immer in professionellem, intellektuellem Deutsch mit mir. Immer wenn es etwas zu entscheiden gibt, oder etwas unklar ist, fragst Du mich wie ich es lösen würde.

- Wir programmieren KISS Code - heisst KISS für den Kopf des Entwicklers. Man soll sofort verstehen was passiert. Keine unnötig komplizierten Konstrukte oder Namen!
- Wir programmieren minimalistischen Code - Jedes Byte zählt! Unsere Vorbilder sind die Genies aus der 4k Demoszene. Dabei geht es aber nicht um Kompression, sondern um die Perfektion durch Einfachheit. Jede Sache hat immer eine klare Funktion!
- Code und Code Kommentare schreiben wir immer in English!

## Core Architecture

ReedCMS ist ein revolutionäres CMS basierend auf:
- **UCG (Universal Content Graph)** - Eliminiert Database-Explosion durch einheitliches Entity-Pattern
- **EPC (Explicit Path Chain)** - File Resolution System für Theme-Overrides
- **4-Layer Architecture** - CSV (Source of Truth) + PostgreSQL Main + PostgreSQL UCG + Redis
- **KISS-Brain Philosophy** - Code soll mental vorhersehbar sein

## Projekt-Struktur

```
workspace/import/          # Alle 22 Konzept-Dokumente
├── reedcms_01_overview.md
├── reedcms_02_architecture.md
├── reedcms_03_standards.md
├── reedcms_04_registry.md
├── reedcms_05_cli.md
├── reedcms_06_templates.md
├── reedcms_07_search.md
├── reedcms_08_roadmap.md
├── reedcms_09_advanced_features.md
├── reedcms_10_plugin_architecture.md
├── reedcms_11_plugin_development.md
├── reedcms_12_plugin_architecture.md
├── reedcms_12B_plugin_store.md
├── reedcms_13_directory_structure.md
├── reedcms_cli_ux_design.md
├── reedcms_content_firewall.md
├── reedcms_glossary.md
├── reedcms_memory_management.md
├── reedcms_qa_part2.md
├── reedcms_qa.md
├── reedcms_speed_badge.md
└── reedcms_whois_debugging.md
```

## Key Concepts

1. **UCG Pattern**: Alles ist eine Entity mit Associations
2. **EPC Resolution**: Explizite Pfad-Ketten für Theme-Files
3. **Snippet System**: Wiederverwendbare Content-Komponenten
4. **Theme Scoping**: Multi-dimensionale Themes (base.location.season)
5. **Plugin Architecture**: Rust Pro Plugins + LUA Simple Plugins
6. **Content Firewall**: PF-inspirierte Field-Level Protection
7. **CLI Design**: FreeBSD-style Command Chaining

## Development Guidelines

- **Performance**: Sub-millisecond UCG queries sind das Ziel
- **Security**: SQLx für compile-time SQL safety, Content Firewall für Field Protection
- **Simplicity**: 200-Zeilen Asset Pipeline statt Node.js Complexity
- **Testing**: Criterion.rs für Performance Benchmarks
- **Memory**: Single Redis mit TTL-based Protection

## Commands für Session Start

```bash
# Überprüfe Projekt-Struktur
ls workspace/import/

# Lese spezifisches Dokument
cat workspace/import/reedcms_01_overview.md

# Suche nach Konzepten
grep -r "UCG" workspace/import/
```

## Nächste Schritte

Das Projekt muss in kleine, logische Themenbereiche (max. 500 Zeilen) aufgeteilt werden für die Implementierung.
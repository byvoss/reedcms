# Neue Struktur für ReedCMS Dokumentation

## docs/concepts/01_core_principles.md
- **Quelle:** reedcms_01_overview.md (Zeilen 10-90)
- **Quelle:** reedcms_03_standards.md (Zeilen 200-318)
- **Quelle:** reedcms_glossary.md (Zeilen 68-90)
- **Inhalt:** KISS-Brain Philosophy, 4k Demo Scene, Anti-Flatterfat Rule
- **Zeilenschätzung:** ~200 Zeilen

## docs/concepts/02_ucg_system.md
- **Quelle:** reedcms_01_overview.md (Zeilen 91-150)
- **Quelle:** reedcms_02_architecture.md (Zeilen 180-350)
- **Quelle:** reedcms_glossary.md (Zeilen 25-45)
- **Inhalt:** UCG (Universal Content Graph) komplett erklärt
- **Zeilenschätzung:** ~300 Zeilen

## docs/concepts/03_4layer_architecture.md
- **Quelle:** reedcms_01_overview.md (Zeilen 151-200)
- **Quelle:** reedcms_02_architecture.md (Zeilen 20-179)
- **Quelle:** reedcms_glossary.md (Zeilen 46-67)
- **Inhalt:** CSV + PostgreSQL Main + PostgreSQL UCG + Redis
- **Zeilenschätzung:** ~250 Zeilen

## docs/architecture/04_database_design.md
- **Quelle:** reedcms_02_architecture.md (Zeilen 351-579)
- **Quelle:** reedcms_qa.md (Zeilen 295-360)
- **Inhalt:** PostgreSQL Schema, Flat Tables, Connection Management
- **Zeilenschätzung:** ~300 Zeilen

## docs/architecture/05_registry_system.md
- **Quelle:** reedcms_04_registry.md (Zeilen 1-480)
- **Inhalt:** Meta-Snippets, CSV Registry, Field Definitions, Validation
- **Zeilenschätzung:** ~480 Zeilen

## docs/architecture/06_template_system.md
- **Quelle:** reedcms_06_templates.md (Zeilen 1-350)
- **Quelle:** reedcms_qa_part2.md (Zeilen 900-1030)
- **Inhalt:** Tera Templates, Web Components, render_snippet, i18n
- **Zeilenschätzung:** ~450 Zeilen

## docs/architecture/07_search_system.md
- **Quelle:** reedcms_07_search.md (Zeilen 1-536)
- **Inhalt:** Redis Search Index, Word Lists, Fuzzy Search
- **Zeilenschätzung:** ~500 Zeilen (knapp über Limit, aber thematisch geschlossen)

## docs/architecture/08_plugin_system.md
- **Quelle:** reedcms_10_plugin_architecture.md (Zeilen 1-316)
- **Quelle:** reedcms_qa.md (Zeilen 40-125)
- **Inhalt:** Unix Sockets, Binary Protocol, LUA/Rust Plugins
- **Zeilenschätzung:** ~400 Zeilen

## docs/development/09_cli_reference.md
- **Quelle:** reedcms_05_cli.md (Zeilen 1-300)
- **Quelle:** reedcms_13_directory_structure.md (Zeilen 220-300)
- **Inhalt:** Command Reference, FreeBSD Style, Basic Examples
- **Zeilenschätzung:** ~380 Zeilen

## docs/development/10_cli_ux_guide.md
- **Quelle:** reedcms_cli_ux_design.md (Zeilen 1-500)
- **Inhalt:** Autocompletion, Error Messages, Interactive Mode
- **Zeilenschätzung:** ~500 Zeilen

## docs/development/11_terminal_features.md
- **Quelle:** reedcms_cli_ux_design.md (Zeilen 501-1000)
- **Inhalt:** Neovim Dropdowns, REPL Mode, Mode Detection
- **Zeilenschätzung:** ~500 Zeilen

## docs/development/12_whois_debugging.md
- **Quelle:** reedcms_cli_ux_design.md (Zeilen 1001-1863)
- **Quelle:** reedcms_whois_debugging.md (Zeilen 1-169)
- **Inhalt:** Whois Command komplett mit allen Parametern
- **Zeilenschätzung:** ~500 Zeilen (gekürzt auf Essenz)

## docs/development/13_plugin_development.md
- **Quelle:** reedcms_11_plugin_development.md (Zeilen 1-400)
- **Inhalt:** LUA Plugin Guide, Rust Plugin Guide, SDK Usage
- **Zeilenschätzung:** ~400 Zeilen

## docs/development/14_directory_structure.md
- **Quelle:** reedcms_13_directory_structure.md (Zeilen 1-219)
- **Quelle:** reedcms_13_directory_structure.md (Zeilen 301-402)
- **Inhalt:** Directory Philosophy, File Organization, Build System
- **Zeilenschätzung:** ~320 Zeilen

## docs/development/15_coding_standards.md
- **Quelle:** reedcms_03_standards.md (Zeilen 1-199)
- **Inhalt:** Language Standards, Code Style, File Naming
- **Zeilenschätzung:** ~200 Zeilen

## docs/features/16_content_management.md
- **Quelle:** reedcms_09_advanced_features.md (Zeilen 1-200)
- **Quelle:** reedcms_04_registry.md (Zeilen 350-450)
- **Inhalt:** Content Types, Multi-language, Live Preview
- **Zeilenschätzung:** ~300 Zeilen

## docs/features/17_theme_system.md
- **Quelle:** reedcms_06_templates.md (Zeilen 351-506)
- **Quelle:** reedcms_qa_part2.md (Zeilen 1031-1250)
- **Inhalt:** Theme Scoping, EPC Resolution, Feature Toggles
- **Zeilenschätzung:** ~375 Zeilen

## docs/features/18_security_firewall.md
- **Quelle:** reedcms_content_firewall.md (Zeilen 1-300)
- **Inhalt:** PF-inspired Rules, Field Protection, CLI Management
- **Zeilenschätzung:** ~300 Zeilen

## docs/features/19_security_implementation.md
- **Quelle:** reedcms_content_firewall.md (Zeilen 301-553)
- **Quelle:** reedcms_qa_part2.md (Zeilen 161-533)
- **Inhalt:** Authentication, CSRF, SQL Prevention, Headers
- **Zeilenschätzung:** ~450 Zeilen

## docs/operations/20_memory_management.md
- **Quelle:** reedcms_memory_management.md (Zeilen 1-347)
- **Inhalt:** Redis Strategy, TTL Protection, Recovery
- **Zeilenschätzung:** ~350 Zeilen

## docs/operations/21_performance_monitoring.md
- **Quelle:** reedcms_speed_badge.md (Zeilen 1-241)
- **Quelle:** reedcms_qa_part2.md (Zeilen 534-690)
- **Inhalt:** Speed Badge, Benchmarking, Performance Targets
- **Zeilenschätzung:** ~400 Zeilen

## docs/operations/22_deployment_backup.md
- **Quelle:** reedcms_09_advanced_features.md (Zeilen 201-400)
- **Quelle:** reedcms_qa.md (Zeilen 361-440)
- **Inhalt:** Backup System, Deploy Management, Migration
- **Zeilenschätzung:** ~280 Zeilen

## docs/ecosystem/23_plugin_store.md
- **Quelle:** reedcms_12B_plugin_store.md (Zeilen 1-300)
- **Inhalt:** Store Architecture, Quality Process, Business Model
- **Zeilenschätzung:** ~300 Zeilen

## docs/ecosystem/24_plugin_ecosystem.md
- **Quelle:** reedcms_12B_plugin_store.md (Zeilen 301-493)
- **Quelle:** reedcms_11_plugin_development.md (Zeilen 401-619)
- **Inhalt:** Developer Onboarding, Best Practices, Community
- **Zeilenschätzung:** ~400 Zeilen

## docs/reference/25_glossary.md
- **Quelle:** reedcms_glossary.md (Zeilen 91-291)
- **Inhalt:** Begriffserklärungen, Acronyme, Konzepte
- **Zeilenschätzung:** ~200 Zeilen

## docs/reference/26_roadmap.md
- **Quelle:** reedcms_08_roadmap.md (Zeilen 1-419)
- **Inhalt:** 6-Phasen Plan, Feature Timeline, Zielgruppen
- **Zeilenschätzung:** ~420 Zeilen

## docs/reference/27_qa_architecture.md
- **Quelle:** reedcms_qa.md (Zeilen 1-294)
- **Quelle:** reedcms_qa.md (Zeilen 441-680)
- **Inhalt:** Architecture Q&A, Design Decisions
- **Zeilenschätzung:** ~490 Zeilen

## docs/reference/28_qa_implementation.md
- **Quelle:** reedcms_qa_part2.md (Zeilen 1-160)
- **Quelle:** reedcms_qa_part2.md (Zeilen 691-900)
- **Inhalt:** Implementation Q&A, Technical Details
- **Zeilenschätzung:** ~370 Zeilen

## Gelöschte/Nicht übernommene Inhalte
- **reedcms_12_plugin_architecture.md** - Duplikat von reedcms_10
- Marketing-lastige Wiederholungen aus overview
- Redundante UCG-Erklärungen (konsolidiert in 02_ucg_system.md)

## Statistik
- **Original:** 22 Dateien, ~11.500 Zeilen
- **Neu:** 28 fokussierte Dokumente, ~10.000 Zeilen
- **Durchschnitt:** ~360 Zeilen pro Dokument
- **Einsparung:** ~1.500 Zeilen durch Deduplizierung
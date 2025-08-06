# Überlappungsmatrix der ReedCMS Dokumentation

## Haupt-Überlappungen nach Konzepten

### UCG (Universal Content Graph)
**Primäre Quellen:**
- reedcms_01_overview.md - Einführung und Marketing-Perspektive
- reedcms_02_architecture.md - Technische Implementation
- reedcms_glossary.md - Definition und Erklärung

**Sekundäre Erwähnungen:**
- reedcms_04_registry.md - UCG-Integration mit Registry
- reedcms_06_templates.md - UCG-aware Template Functions
- reedcms_07_search.md - UCG für Search Index
- reedcms_qa.md - UCG Recovery Strategy

### 4-Layer Architecture
**Primäre Quellen:**
- reedcms_01_overview.md - Konzept-Einführung
- reedcms_02_architecture.md - Detaillierte Erklärung

**Sekundäre Erwähnungen:**
- reedcms_glossary.md - Kurzdefinition
- reedcms_memory_management.md - Redis Layer Details
- reedcms_qa.md - Layer-spezifische Fragen

### CLI Commands & Syntax
**Primäre Quellen:**
- reedcms_05_cli.md - Command Reference
- reedcms_cli_ux_design.md - UX Philosophy und Details

**Sekundäre Erwähnungen:**
- reedcms_13_directory_structure.md - CLI Syntax Standards
- reedcms_content_firewall.md - Firewall CLI Commands
- reedcms_whois_debugging.md - Whois Command Details

### Template System
**Primäre Quellen:**
- reedcms_06_templates.md - Vollständige Dokumentation

**Sekundäre Erwähnungen:**
- reedcms_03_standards.md - Tera Template Standards
- reedcms_04_registry.md - Template Integration
- reedcms_qa_part2.md - Template Hot Reload

### Plugin Architecture
**Identische Duplikation:**
- reedcms_10_plugin_architecture.md
- reedcms_12_plugin_architecture.md
(Komplett identischer Inhalt)

**Ergänzende Dokumentation:**
- reedcms_11_plugin_development.md - Development Guide
- reedcms_12B_plugin_store.md - Store Ecosystem

### Performance & Memory
**Primäre Quellen:**
- reedcms_memory_management.md - Memory Strategy
- reedcms_speed_badge.md - Performance Gamification

**Sekundäre Erwähnungen:**
- reedcms_01_overview.md - Performance Targets
- reedcms_02_architecture.md - Performance Characteristics
- reedcms_07_search.md - Search Performance
- reedcms_qa_part2.md - Performance Benchmarking

### Security & Content Protection
**Primäre Quellen:**
- reedcms_content_firewall.md - Content Firewall System
- reedcms_qa_part2.md - Security Implementation Details

**Sekundäre Erwähnungen:**
- reedcms_09_advanced_features.md - Content Security Features
- reedcms_11_plugin_development.md - Plugin Security

## Detail-Überlappungsmatrix

| Dokument | UCG | 4-Layer | CLI | Templates | Plugins | Security | Performance |
|----------|-----|---------|-----|-----------|---------|----------|-------------|
| 01_overview | *** | *** | * | * | * | | ** |
| 02_architecture | *** | *** | | ** | * | * | ** |
| 03_standards | | * | * | ** | | | * |
| 04_registry | ** | * | * | ** | | | |
| 05_cli | * | | *** | * | * | | |
| 06_templates | ** | | | *** | * | | |
| 07_search | ** | * | * | | | | *** |
| 08_roadmap | * | * | | | * | * | |
| 09_advanced | * | * | * | * | | ** | |
| 10_plugin_arch | | | * | | *** | * | * |
| 11_plugin_dev | | | * | | *** | * | |
| 12B_store | | | | | *** | * | |
| 13_directory | * | | ** | * | * | | |
| cli_ux_design | * | | *** | * | | * | * |
| content_firewall | | | ** | * | * | *** | |
| glossary | *** | *** | * | * | * | | * |
| memory_mgmt | * | ** | * | | | | *** |
| qa | ** | ** | * | * | ** | * | * |
| qa_part2 | ** | * | ** | ** | * | *** | ** |
| speed_badge | | | | | | | *** |
| whois_debug | ** | | *** | | | | * |

Legende: *** = Hauptthema, ** = Wichtige Erwähnung, * = Nebensächliche Erwähnung

## Konsolidierungspotenzial

### Hohe Redundanz (sollte konsolidiert werden):
1. **Plugin Architecture Duplikat** - reedcms_10 und reedcms_12 sind identisch
2. **UCG Erklärungen** - Verteilt über 6+ Dokumente, könnte in einem UCG-Konzept-Dokument vereint werden
3. **CLI Syntax** - Überlappung zwischen reference und UX design

### Mittlere Redundanz (kann getrennt bleiben):
1. **Performance Informationen** - Verschiedene Aspekte in verschiedenen Kontexten
2. **Security Features** - Content Firewall vs. allgemeine Security
3. **Template System** - Kern-Doku vs. Integration in anderen Komponenten

### Sinnvolle Verteilung (sollte so bleiben):
1. **4-Layer Architecture** - Overview für Konzept, Architecture für Details
2. **Development Standards** - Eigenes Dokument mit Referenzen
3. **Q&A Dokumente** - Spezifische Problemlösungen und Entscheidungen
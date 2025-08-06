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

## WICHTIG: Ticket-System und Arbeitsweise

### Beim Session-Start SOFORT einlesen (in dieser Reihenfolge):
1. **workspace/tickets/VALIDATION_CHECKLIST.md** - Goldene Regeln der Arbeit
2. **workspace/tickets/DEPENDENCY_FLOW.md** - System-Erklärung und Sprint-Erkennung
3. **workspace/tickets/SPRINT_CHECKLIST.md** - Sprint-Prozess verstehen
4. **workspace/tickets/ticket_log.csv** - Identifiziere aktiven Sprint via "active" Status
5. **workspace/tickets/REED-S###/REED-S###.md** - Sprint-Status des aktiven Sprints
6. **workspace/tickets/REED-S###/REED-S###-T00.md** - Meta-Prozess des aktiven Sprints

### Selbstprüfender Ticket-Zyklus

Wir arbeiten in einem selbstprüfenden, selbstdokumentierenden Ticket-System:

```
workspace/tickets/
├── ticket_log.csv              # ZENTRALE Wahrheit: Alle Sprints & Tickets
├── VALIDATION_CHECKLIST.md     # GOLDENE REGEL: Original-Informationsgehalt bewahren
├── DEPENDENCY_FLOW.md          # System-Erklärung und Sprint-Erkennung
├── SPRINT_CHECKLIST.md         # Sprint-Management-Prozess
├── SPRINT_TEMPLATE.md          # Vorlage für neue Sprints
├── README.md                   # Übersicht des Ticket-Systems
├── REED-S001/                  # Sprint 1: Dokumentations-Neustrukturierung
├── REED-S002/                  # Sprint 2: [Nächstes Thema]
└── REED-S###/                  # Weitere thematische Sprints
    ├── REED-S###.md           # Sprint-Übersicht mit Fortschritt
    ├── REED-S###-T00.md       # Meta-Ticket: IMMER befolgen!
    └── REED-S###-T##.md       # Sprint-spezifische Tickets
```

### Kritische Arbeitsregeln

1. **NUR EIN TICKET** gleichzeitig aktiv (status: in_progress)
2. **Original-Informationsgehalt** bewahren - bei Unklarheit IMMER nachfragen
3. **Keine eigenen Interpretationen** ohne explizite Nachfrage
4. **Sprint-Dokument** bei jedem Ticket-Abschluss aktualisieren
5. **Git commits** mit Format: [TICKET-ID] - Beschreibung

### Aktueller Status (Stand: 2025-08-06)
- **Sprint:** REED-S001 - Dokumentations-Neustrukturierung
- **Fortschritt:** 20% (2 von 10 Tickets abgeschlossen)
- **Abgeschlossen:** T01 (Repository Setup), T02 (Analyse)
- **Nächstes Ticket:** T03 (Core Concepts)

### Workflow für neue Session
```bash
# 1. Aktiven Sprint identifizieren (via Sprint-Spalte)
ACTIVE_SPRINT=$(grep "active" workspace/tickets/ticket_log.csv | head -1 | cut -d',' -f6 | sed 's/S/REED-S/')

# Falls kein aktives Ticket, nächstes pending Ticket finden
if [ -z "$ACTIVE_SPRINT" ]; then
    ACTIVE_SPRINT=$(grep "pending" workspace/tickets/ticket_log.csv | head -1 | cut -d',' -f6 | sed 's/S/REED-S/')
fi

# 2. Sprint-Status checken
cat workspace/tickets/$ACTIVE_SPRINT/$ACTIVE_SPRINT.md | grep "Fortschritt"

# 3. Aktives Ticket identifizieren oder nächstes starten
# 4. Meta-Ticket T00 des aktiven Sprints befolgen
```

## Projekt-Übersicht

Das Projekt transformiert 22 Import-Dokumente in strukturierte Implementierungs-Dokumentation.
Alle Original-Dokumente befinden sich in workspace/import/.
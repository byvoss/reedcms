# Context Check für [REED-S001-T05] - Core Architecture - 4-Layer Architecture

## Ziel
Dokumentation der 4-Layer Architektur mit Fokus auf:
- Jede Layer klar definiert
- Verantwortlichkeiten abgegrenzt  
- Anti-Bloat Mechanismen
- Data Flow
- KEINE UCG/EPC Details

## Input-Dokumente zu laden
1. `reedcms_02_architecture.md` - 4-Layer Details
2. `reedcms_glossary.md` - Layer Definitionen
3. `reedcms_qa.md` - Layer Clarifications

## Output
- `workspace/restructured/03-4layer-architecture.md`
- Max. 500 Zeilen
- Fokus NUR auf Layer-System

## Wichtige Konzepte
- CSV (Source of Truth)
- PostgreSQL Main
- PostgreSQL UCG  
- Redis
- Anti-Bloat Design Pattern
- Separation of Concerns
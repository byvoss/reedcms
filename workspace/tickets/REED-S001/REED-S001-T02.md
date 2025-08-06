# [REED-S001-T02] - Analyse und Kategorisierung der Import-Dokumente

## Status: INACTIVE

## Beschreibung
Analysiere alle 22 Dokumente unter workspace/import und erstelle eine klare Kategorisierung für die Neustrukturierung.

## Konkrete Schritte
1. Liste alle 22 Dateien aus workspace/import mit `ls -la`
2. Lese JEDE Datei vollständig ein mit `Read` tool
3. Erstelle für jede Datei eine Zusammenfassung:
   - Hauptthema
   - Unterthemen
   - Anzahl Zeilen
   - Enthaltene Konzepte
4. Identifiziere Überlappungen zwischen Dokumenten
5. Gruppiere nach Themen (nicht nach Dateinamen)
6. Erstelle Mapping-Tabelle

## Akzeptanzkriterien
- [ ] Alle 22 Dokumente aus workspace/import zu 100% eingelesen
- [ ] workspace/analysis/file_summaries.md erstellt mit Zusammenfassung jeder Datei
- [ ] workspace/analysis/theme_groups.md erstellt mit thematischer Gruppierung
- [ ] workspace/analysis/overlap_matrix.md erstellt mit Überlappungen
- [ ] workspace/analysis/restructure_plan.md mit konkretem Mapping
- [ ] Prüfung: Jeder Satz aus workspace/import ist in der neuen Struktur zugeordnet
- [ ] Status in ticket_log.csv auf 'completed' gesetzt

## Regeln
- Dokumentation: DEUTSCH
- Code und Code-Kommentare: ENGLISH
- Commit Message Format: "[REED-S001-T02] - description..."
- KEINE neuen Konzepte hinzufügen
- NUR vorhandene Inhalte analysieren
- Bei Unsicherheit: Original-Formulierung beibehalten

## Input-Dokumente
- workspace/import/*.md (22 Dateien)

## Output-Struktur
### workspace/analysis/file_summaries.md
```
# Datei: reedcms_01_overview.md
- Zeilen: XXX
- Hauptthema: ...
- Unterthemen: ...
- Schlüsselkonzepte: ...

# Datei: reedcms_02_architecture.md
...
```

### workspace/analysis/theme_groups.md
```
# Themengruppe: Core Architecture
- Quellen: reedcms_01_overview.md (Zeilen X-Y), reedcms_02_architecture.md (Zeilen A-B)
- Kernaussagen: ...

# Themengruppe: UCG System
...
```

### workspace/analysis/restructure_plan.md
```
# Neue Struktur

## docs/concepts/01_core_principles.md
- Quelle: reedcms_01_overview.md (Zeilen 10-50)
- Quelle: reedcms_03_standards.md (Zeilen 5-20)
- Inhalt: KISS-Brain, Minimalistic Code, ...

## docs/concepts/02_ucg_system.md
...
```
# [REED-S001-T00] - Meta-Ticket: Arbeitsweise und Qualitätssicherung

## Status: ACTIVE (immer aktiv)

## Beschreibung
Dieses Meta-Ticket definiert die verbindliche Arbeitsweise für alle anderen Tickets.

## Verbindliche Arbeitsschritte für JEDES Ticket

### 1. Ticket-Start
- [ ] Status in ticket_log.csv von 'inactive' auf 'active' setzen
- [ ] Andere Tickets auf 'inactive' setzen (nur 1 aktives Ticket)
- [ ] workspace/analysis/context_check.md erstellen mit:
  - Liste aller relevanten Input-Dateien
  - Bestätigung, dass alle zu 100% im Kontext sind

### 2. Während der Bearbeitung
- [ ] Bei JEDEM Absatz prüfen: Stammt dieser aus workspace/import?
- [ ] Bei JEDER Aussage prüfen: Ist das meine Interpretation oder Original?
- [ ] Bei Unsicherheit: Original-Informationsgehalt verstehen und nachfragen
- [ ] KEINE eigenen Interpretationen ohne explizite Nachfrage
- [ ] Fortschritt in workspace/analysis/progress_[ticket-id].md dokumentieren

### 3. Qualitätsprüfung
- [ ] Zeile-für-Zeile Vergleich: Output vs. Input
- [ ] Checkliste in workspace/analysis/qa_[ticket-id].md:
  ```
  - [ ] Keine neuen Konzepte hinzugefügt
  - [ ] Keine eigenen Interpretationen
  - [ ] Alle Aussagen aus Originaldokumenten
  - [ ] Sprache: Doku Deutsch, Code English
  - [ ] Dokument < 500 Zeilen
  ```

### 4. Ticket-Ende
- [ ] Git add, commit mit Format "[TICKET-ID] - Beschreibung"
- [ ] Git push
- [ ] Status in ticket_log.csv auf 'completed' setzen
- [ ] acceptance_criteria_met auf 'done' setzen
- [ ] Sprint-Dokument REED-S001.md aktualisieren (Status + Metriken)

## Warnsignale für Abweichung
- Verwendung von "könnte", "sollte", "würde" ohne Original-Referenz
- Neue technische Begriffe, die nicht in workspace/import vorkommen
- Code-Beispiele, die nicht aus den Dokumenten stammen
- Länge > 500 Zeilen

## Konsequenz bei Nicht-Einhaltung
- Ticket wird als 'failed' markiert
- Neustart des Tickets erforderlich
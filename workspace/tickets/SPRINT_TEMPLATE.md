# Sprint-Template für neue Sprints

## Verwendung

Wenn ein Sprint abgeschlossen ist und ein neuer Sprint beginnt:

1. Kopiere dieses Template nach `REED-S###/REED-S###.md`
2. Ersetze alle `S###` mit der neuen Sprint-Nummer (z.B. S002)
3. Definiere Sprint-Ziele und Tickets
4. Erstelle Meta-Ticket T00 aus dem Template unten

## Sprint-Dokument Template

```markdown
# [REED-S###] - Sprint: [THEMATISCHER TITEL]

## Sprint-Übersicht
- **Sprint-ID:** REED-S###
- **Sprint-Name:** [Beschreibender Name]
- **Sprint-Start:** [YYYY-MM-DD]
- **Sprint-Ende:** [YYYY-MM-DD] (geschätzt)
- **Sprint-Status:** ACTIVE
- **Sprint-Ziel:** [Hauptziel in einem Satz]

## Sprint-Ziele

### Primärziel
**[Hauptziel ausführlich beschreiben]**

### Detailziele
1. **[Erstes Teilziel]** (T01)
2. **[Zweites Teilziel]** (T02)
3. **[Weitere Teilziele...]** (T##)

## Sprint-Regeln

[Identisch zu REED-S001 - siehe sprint_checklist.md]

## Sprint-Fortschritt

### Ticket-Status-Übersicht
| Ticket | Titel | Status | Fortschritt |
|--------|-------|--------|-------------|
| T00 | Meta-Ticket: Arbeitsweise | active | ongoing |
| T01 | [Titel] | inactive | 0% |
| T02 | [Titel] | inactive | 0% |

### Sprint-Metriken
- **Tickets gesamt:** X (+1 Meta)
- **Tickets abgeschlossen:** 0
- **Tickets in Arbeit:** 0
- **Tickets ausstehend:** X
- **Sprint-Fortschritt:** 0%

## Qualitätssicherung

[Standard-Checklisten wie in REED-S001]

## Lessons Learned (wird am Sprint-Ende ausgefüllt)
- [ ] Was war besonders erfolgreich?
- [ ] Welche Herausforderungen gab es?
- [ ] Was würden wir beim nächsten Sprint anders machen?
- [ ] Welche Prozess-Verbesserungen sind nötig?
```

## Meta-Ticket T00 Template

```markdown
# [REED-S###-T00] - Meta-Ticket: Arbeitsweise und Qualitätssicherung

## Status: ACTIVE (immer aktiv während Sprint)

## Beschreibung
Dieses Meta-Ticket definiert die verbindliche Arbeitsweise für alle Tickets des Sprints REED-S###.

## Verbindliche Arbeitsschritte

[Kopiere Inhalt von REED-S001-T00 und passe Sprint-Referenzen an]

## Sprint-spezifische Ergänzungen
[Hier können sprint-spezifische Regeln oder Anpassungen eingefügt werden]
```

## Ticket-Template für Sprint-Tickets

```markdown
# [REED-S###-T##] - [Ticket-Titel]

## Status: INACTIVE

## Beschreibung
[Klare Beschreibung der Aufgabe]

## Akzeptanzkriterien
- [ ] [Kriterium 1]
- [ ] [Kriterium 2]
- [ ] [Weitere Kriterien...]
- [ ] Status in ticket_log.csv auf 'completed' gesetzt

## Regeln
- Dokumentation: DEUTSCH
- Code und Code-Kommentare: ENGLISH
- [Sprint-spezifische Regeln]

## Input-Referenzen
- [Liste der relevanten Quelldokumente]

## Output
- [Erwartete Ergebnisdateien]
```

## ticket_log.csv Einträge für neuen Sprint

```csv
[REED-S###-T00],Meta-Ticket: Arbeitsweise und Qualitätssicherung,inactive,[START-DATE] 10:00:00,[START-DATE] 10:00:00,S###,Claude,ongoing
[REED-S###-T01],[Ticket-Titel],inactive,[START-DATE] 10:00:00,[START-DATE] 10:00:00,S###,Claude,pending
[REED-S###-T02],[Ticket-Titel],inactive,[START-DATE] 10:00:00,[START-DATE] 10:00:00,S###,Claude,pending
```

## Checkliste für neuen Sprint

- [ ] Sprint-Ordner `REED-S###/` erstellen
- [ ] Sprint-Dokument aus Template erstellen
- [ ] Meta-Ticket T00 erstellen
- [ ] Alle Sprint-Tickets definieren und erstellen
- [ ] ticket_log.csv mit neuen Einträgen erweitern
- [ ] Vorherigen Sprint auf "COMPLETED" setzen
- [ ] Lessons Learned vom vorherigen Sprint dokumentieren
- [ ] Git commit mit `[REED-S###] - Sprint Setup: [Thema]`
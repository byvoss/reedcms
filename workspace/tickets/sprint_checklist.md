# Sprint-Level Checkliste

## Wichtig: Einlesereihenfolge
Siehe DEPENDENCY_FLOW.md für optimale Einlesereihenfolge der Dokumente!

## Sprint-Start Checkliste

### 1. Sprint-Planung
- [ ] Sprint-ID vergeben (Format: REED-S###)
- [ ] Sprint-Ziele klar definiert
- [ ] Alle Tickets für Sprint erstellt
- [ ] Akzeptanzkriterien pro Ticket definiert
- [ ] Zeitschätzung realistisch

### 2. Sprint-Dokumentation
- [ ] Sprint-Ordner workspace/tickets/REED-S### erstellt
- [ ] Sprint-Dokument REED-S###.md im Sprint-Ordner erstellt
- [ ] Alle Sprint-Tickets im Sprint-Ordner abgelegt
- [ ] Alle Tickets in ticket_log.csv eingetragen
- [ ] Meta-Ticket T00 für Sprint aktiviert
- [ ] Validation Checklist aktuell

### 3. Arbeitsumgebung
- [ ] Git Repository initialisiert
- [ ] Alle Import-Dokumente verfügbar
- [ ] Entwicklungsumgebung bereit

## Während des Sprints

### Tägliche Routine
- [ ] Sprint-Status in REED-S###.md prüfen
- [ ] Nur ein Ticket aktiv (in_progress)
- [ ] Meta-Ticket T00 Regeln befolgen
- [ ] Bei Blockern sofort dokumentieren

### Ticket-Bearbeitung
- [ ] Context Check durchführen
- [ ] Original-Informationsgehalt verstehen
- [ ] Bei Unklarheiten nachfragen
- [ ] Progress dokumentieren
- [ ] QA-Checkliste durchgehen

### Sprint-Kommunikation
- [ ] Sprint-Dokument aktuell halten
- [ ] Git commits aussagekräftig
- [ ] Fortschritt transparent machen

## Sprint-Abschluss

### Review
- [ ] Alle Tickets abgeschlossen oder begründet
- [ ] Sprint-Ziele erreicht oder Abweichungen erklärt
- [ ] Lessons Learned dokumentiert
- [ ] Metriken finalisiert

### Übergabe zum nächsten Sprint
- [ ] Alle Dokumente committed und gepusht
- [ ] Sprint-Dokument als "COMPLETED" markiert
- [ ] Neuen Sprint-Ordner erstellen (REED-S###)
- [ ] SPRINT_TEMPLATE.md für neuen Sprint verwenden
- [ ] Neue Tickets in ticket_log.csv eintragen
- [ ] Thematischen Fokus für neuen Sprint definieren
- [ ] Retrospektive durchgeführt

## Sprint-Übergangs-Workflow

### Automatische Sprint-Erkennung
```bash
# Aktuelle Sprint-Nummer finden
CURRENT_SPRINT=$(ls workspace/tickets/ | grep "REED-S" | sort -V | tail -1)
SPRINT_NUM=$(echo $CURRENT_SPRINT | sed 's/REED-S0*//')
NEXT_NUM=$((SPRINT_NUM + 1))
NEXT_SPRINT=$(printf "REED-S%03d" $NEXT_NUM)
```

### Neuen Sprint erstellen
1. `mkdir workspace/tickets/$NEXT_SPRINT`
2. Template kopieren und anpassen
3. Tickets für neues Thema definieren
4. ticket_log.csv erweitern

## Kontinuierliche Verbesserung

### Prozess-Optimierung
- [ ] Was hat gut funktioniert?
- [ ] Was war schwierig?
- [ ] Welche Tools/Prozesse fehlen?
- [ ] Action Items für nächsten Sprint

### Qualitätssicherung
- [ ] Wurden alle Validierungen eingehalten?
- [ ] Ist die Dokumentation verständlich?
- [ ] Sind die Prozesse effizient?
- [ ] Gibt es Automatisierungspotenzial?

---

**Diese Checkliste ist Teil des selbstprüfenden Zyklus und wird basierend auf Erfahrungen kontinuierlich verbessert.**
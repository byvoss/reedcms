# Ticket-System Dependency Flow

## Einlesereihenfolge für optimales Verständnis

### 1. Grundlagen-Ebene (Philosophie & Regeln)
```
validation_checklist.md
    ↓
    └─> GOLDENE REGEL: Original-Informationsgehalt bewahren
    └─> Kontext-Prüfung vor Start
    └─> Inhalts-Treue während Arbeit
    └─> Qualitätskontrolle nach Arbeit
```

### 2. Sprint-Ebene (Organisation & Prozess)
```
sprint_checklist.md
    ↓
    └─> Sprint-Start Prozess
    └─> Tägliche Routine
    └─> Sprint-Abschluss
    └─> Kontinuierliche Verbesserung
```

### 3. Status-Ebene (Aktueller Stand)
```
ticket_log.csv
    ↓
    └─> Globale Übersicht aller Tickets
    └─> Status: inactive/active/completed
    └─> Fortschritt pro Ticket
```

### 4. Sprint-Dokumente (Detailstatus)
```
REED-S###/REED-S###.md (aktueller Sprint aus ticket_log.csv)
    ↓
    └─> Sprint-Ziele
    └─> Fortschrittsmetriken
    └─> Ticket-Übersicht
    └─> Lessons Learned
```

### 5. Meta-Prozess (Arbeitsanweisungen)
```
REED-S###/REED-S###-T00.md (Meta-Ticket des aktuellen Sprints)
    ↓
    └─> Verbindliche Arbeitsschritte
    └─> Ticket-Start bis Ticket-Ende
    └─> Warnsignale für Abweichungen
    └─> Sprint-Update nach Abschluss
```

## Kreislauf-Logik

```mermaid
graph TD
    A[Session Start] --> B[CLAUDE.md lesen]
    B --> C[validation_checklist.md]
    C --> D[ticket_log.csv prüfen]
    D --> E{Aktives Ticket?}
    E -->|Ja| F[Ticket fortsetzen]
    E -->|Nein| G[Nächstes Ticket starten]
    F --> H[Meta-Ticket T00 befolgen]
    G --> H
    H --> I[Arbeit durchführen]
    I --> J[QA + Progress Docs]
    J --> K[Git Commit]
    K --> L[ticket_log.csv update]
    L --> M[Sprint-Doc update]
    M --> N[Nächstes Ticket oder Ende]
```

## Prioritäts-Hierarchie

1. **HÖCHSTE:** validation_checklist.md - Definiert WIE gearbeitet wird
2. **HOCH:** REED-S###/REED-S###-T00.md - Definiert PROZESS für jedes Ticket
3. **MITTEL:** ticket_log.csv - Zeigt WAS zu tun ist und WELCHER Sprint aktiv
4. **KONTEXT:** REED-S###/REED-S###.md - Zeigt WARUM und FORTSCHRITT
5. **REFERENZ:** sprint_checklist.md - Zeigt ORGANISATION

## Selbsterklärungs-Mechanismen

### In CLAUDE.md:
- Verweist auf Ticket-System beim Start
- Zeigt Einlesereihenfolge
- Erklärt Arbeitsregeln

### In validation_checklist.md:
- GOLDENE REGEL prominent
- Checklisten-Format selbsterklärend
- Rote Flaggen als Warnung

### In REED-S###/REED-S###-T00.md:
- Schritt-für-Schritt Anweisungen
- Warnsignale definiert
- Konsequenzen erklärt
- Gilt für ALLE Tickets des Sprints

### In ticket_log.csv:
- Selbsterklärendes CSV-Format
- Status-Spalten eindeutig
- Sprint-Zuordnung zeigt aktuellen/aktive Sprints
- ZENTRALE Wahrheit für Sprint-Identifikation

### In REED-S###/REED-S###.md:
- Sprint-Ziele definiert
- Fortschritt visualisiert
- Metriken automatisch
- Sprint-spezifische Informationen

## Sprint-Erkennungs-Algorithmus

### Wie finde ich den aktuellen Sprint?

```bash
# 1. Aktive Tickets finden
grep "active" ticket_log.csv

# 2. Sprint-ID extrahieren aus Spalte 6 (Sprint)
ACTIVE_SPRINT=$(grep "active" ticket_log.csv | head -1 | cut -d',' -f6 | sed 's/S/REED-S/')

# 3. Falls kein aktives Ticket, nächstes pending Ticket
if [ -z "$ACTIVE_SPRINT" ]; then
    ACTIVE_SPRINT=$(grep "pending" ticket_log.csv | head -1 | cut -d',' -f6 | sed 's/S/REED-S/')
fi

# 4. Sprint-Ordner = $ACTIVE_SPRINT
# 5. Sprint-Dokument = $ACTIVE_SPRINT/$ACTIVE_SPRINT.md
# 6. Meta-Ticket = $ACTIVE_SPRINT/$ACTIVE_SPRINT-T00.md
```

### Sprint-Übergang

Wenn alle Tickets eines Sprints completed sind:
1. Sprint-Dokument mit Lessons Learned abschließen
2. Sprint-Status in REED-S###.md auf "COMPLETED" setzen
3. Neuen Sprint-Ordner erstellen (z.B. REED-S002)
4. SPRINT_TEMPLATE.md in neuen Ordner kopieren und anpassen
5. Neue Tickets für neuen Sprint definieren (thematischer Fokus!)
6. ticket_log.csv mit neuen Sprint-Tickets erweitern
7. Erstes Ticket des neuen Sprints auf "active" setzen
8. Meta-Ticket T00 ist IMMER das erste aktive Ticket

**WICHTIG:** Es muss IMMER mindestens ein Ticket "active" sein!

### Multi-Sprint-Support

```
workspace/tickets/
├── ticket_log.csv          # ALLE Sprints
├── REED-S001/             # Sprint 1: Dokumentation
├── REED-S002/             # Sprint 2: Core Implementation
├── REED-S003/             # Sprint 3: Features
└── REED-S###/             # Weitere thematische Sprints
```

## Fehlerbehandlung

### Häufige Probleme und Lösungen

1. **Kein aktives Ticket gefunden**
   - Prüfe ob Sprint abgeschlossen ist
   - Setze nächstes pending Ticket auf active
   - Meta-Ticket T00 sollte immer active sein

2. **Sprint-Ordner existiert nicht**
   - Prüfe ticket_log.csv für korrekte Sprint-ID
   - Erstelle fehlenden Ordner aus SPRINT_TEMPLATE.md

3. **Mehrere Tickets sind active**
   - NUR EIN Ticket darf active sein (außer Meta-Ticket)
   - Setze alle außer einem auf inactive

4. **Case-Sensitivity-Probleme**
   - Alle .md Steuerdateien sind GROSSGESCHRIEBEN
   - ticket_log.csv ist kleingeschrieben

## Verbesserungsvorschläge

1. **README.md in tickets/**: ✅ Bereits erstellt
2. **DEPENDENCY_FLOW.md**: ✅ Dieses Dokument (jetzt Sprint-generisch)
3. **Sprint-Detection-Script**: Könnte automatisch aktuellen Sprint finden
4. **Sprint-Template**: ✅ SPRINT_TEMPLATE.md erstellt
5. **Fehlerbehandlung**: ✅ Dokumentiert
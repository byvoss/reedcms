# ReedCMS Ticket System

## Struktur

```
workspace/tickets/
├── README.md                    # Diese Datei
├── ticket_log.csv              # Zentrale Ticket-Übersicht
├── validation_checklist.md     # Allgemeine Validierungs-Checkliste
├── sprint_checklist.md         # Sprint-Level Checkliste
│
└── REED-S###/                  # Sprint-Ordner (z.B. REED-S001)
    ├── REED-S###.md           # Sprint-Übersichtsdokument
    ├── REED-S###-T00.md       # Meta-Ticket für den Sprint
    ├── REED-S###-T01.md       # Sprint-Ticket 1
    ├── REED-S###-T02.md       # Sprint-Ticket 2
    └── ...                     # Weitere Sprint-Tickets
```

## Ticket-Namenskonvention

- **Sprint:** `REED-S###` (z.B. REED-S001)
- **Ticket:** `REED-S###-T##` (z.B. REED-S001-T02)
- **Meta-Ticket:** Immer T00 pro Sprint

## Workflow

1. **Sprint-Start:**
   - Sprint-Ordner erstellen: `workspace/tickets/REED-S###/`
   - Sprint-Dokument erstellen: `REED-S###.md`
   - Alle Sprint-Tickets im Ordner ablegen

2. **Ticket-Bearbeitung:**
   - Nur ein Ticket gleichzeitig aktiv
   - Meta-Ticket T00 befolgen
   - Validation Checklist nutzen

3. **Sprint-Tracking:**
   - ticket_log.csv = globale Übersicht
   - REED-S###.md = Sprint-spezifische Details

## Zentrale Dokumente

- **ticket_log.csv:** Alle Tickets aller Sprints
- **validation_checklist.md:** Goldene Regeln für Ticket-Bearbeitung
- **sprint_checklist.md:** Sprint-Management-Prozess

## Einlesereihenfolge für neue Sessions

**WICHTIG:** Siehe DEPENDENCY_FLOW.md für die optimale Reihenfolge!

1. CLAUDE.md (Root-Verzeichnis)
2. validation_checklist.md
3. ticket_log.csv
4. Aktueller Sprint-Ordner
5. Meta-Ticket T00

## Best Practices

1. **Organisation:** Jeder Sprint in eigenem Ordner
2. **Transparenz:** Sprint-Dokument zeigt Live-Fortschritt
3. **Qualität:** Validation Checklist bei jedem Ticket
4. **Nachvollziehbarkeit:** Git-History dokumentiert alles
5. **Selbsterklärung:** System erklärt sich beim Einlesen selbst
# Validierungs-Checkliste für Ticket-Bearbeitung

## VOR dem Start eines Tickets

### 1. Kontext-Prüfung
- [ ] Sind ALLE relevanten workspace/import Dateien im Kontext?
- [ ] Wurde jede Datei mit dem Read-Tool vollständig eingelesen?
- [ ] Gibt es eine Liste der geladenen Dateien?

### 2. Ziel-Klarheit
- [ ] Ist das Output-Format klar definiert?
- [ ] Sind die Akzeptanzkriterien eindeutig?
- [ ] Ist die maximale Zeilenzahl (500) bekannt?

## WÄHREND der Bearbeitung

### 3. Inhalts-Treue
- [ ] Jeder Satz: Woher stammt diese Information?
- [ ] Jede Aussage: Original oder Interpretation?
- [ ] Jedes Konzept: In workspace/import vorhanden?

### 4. Sprach-Regeln
- [ ] Dokumentation: DEUTSCH
- [ ] Code: ENGLISH
- [ ] Code-Kommentare: ENGLISH

## NACH der Bearbeitung

### 5. Qualitätskontrolle
- [ ] Zeilenzahl < 500?
- [ ] Keine neuen Konzepte?
- [ ] Keine eigenen Interpretationen?
- [ ] Alle Ursprungsdokumente referenziert?

### 6. Abschluss
- [ ] Git commit mit korrektem Format?
- [ ] ticket_log.csv aktualisiert?
- [ ] Nächstes Ticket vorbereitet?

## ROTE FLAGGEN

⚠️ **STOPP bei folgenden Situationen:**
- Unsicherheit über Ursprung einer Information
- Versuchung, etwas "besser" zu formulieren
- Überschreitung der 500-Zeilen-Grenze
- Fehlendes Original-Dokument im Kontext

## GOLDENE REGEL

**Im Zweifel: Original-Informationsgehalt bewahren!**
- Konzept dahinter verstehen
- Bei Unklarheiten IMMER nachfragen
- Keine eigenen Interpretationen hinzufügen
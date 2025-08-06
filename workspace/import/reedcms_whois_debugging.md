# ReedCMS Whois Command - Universal Entity Debugging

## Das Killer-Feature für CMS Administration

**"Nie wieder mystery UUIDs oder broken content ohne Kontext!"**

Jeder CMS-Admin kennt das Problem: Error logs voller UUIDs, broken links, performance issues - aber keine Ahnung was dahinter steckt. ReedCMS löst das mit dem `whois` command, der **sofortigen Kontext** zu jeder Entity liefert.

## Grundlegendes Whois

### **Der Basis-Check**
```bash
reed whois 550e8400-e29b-41d4-a716-446655440000
```

**Was Sie sofort sehen:**
- **Entity-Typ**: Ist das ein snippet, user, theme?
- **Semantic Name**: $welcome_hero statt kryptische UUID
- **Status**: Active, draft, broken, unused?
- **Grunddaten**: Wann erstellt, von wem, letzte Änderung

**Praktisch für:** Error-Debugging, Content-Audit, Daily Administration

---

## Parameter-Modi für Spezial-Analysen

### **`--epc` - UCG Chain Analysis**
```bash
reed whois $welcome_hero --epc
```

**Zeigt die komplette UCG-Hierarchie:** Wo hängt diese Entity in der Content-Struktur? Wie ist die Parent-Child-Beziehung? Welche Theme-Files werden verwendet?

**Nutzen:** Verstehen warum Content nicht rendert, Theme-Debugging, Struktur-Analyse

### **`--references` - Cross-Reference Mapping**
```bash
reed whois $old_banner --references
```

**Zeigt alle Verwendungen:** Welche Pages nutzen diese Entity? In welchen Themes wird sie überschrieben? Wie viele User sehen das täglich?

**Nutzen:** Impact-Analyse vor Löschungen, Content-Audit, Usage-Tracking

### **`--performance` - Performance Intelligence**
```bash
reed whois $slow_gallery --performance
```

**Zeigt Performance-Metriken:** Rendering-Zeit, Memory-Usage, Cache-Hit-Rate, User-Impact. Plus Optimierungsvorschläge.

**Nutzen:** Performance-Probleme identifizieren, Bottlenecks finden, Site-Speed optimieren

### **`--health` - Entity Health Check**
```bash
reed whois $contact_form --health
```

**Zeigt Health-Status:** Schema-Validation, Missing-Fields, Security-Issues, SEO-Warnings. Bekommt Health-Score 0-100.

**Nutzen:** Proaktive Wartung, Quality-Assurance, Compliance-Checks

### **`--debug` - Broken Entity Analysis**
```bash
reed whois $broken_widget --debug
```

**Zeigt Debug-Details:** Missing Templates, Registry-Errors, UCG-Integrity-Issues. Plus konkrete Repair-Schritte.

**Nutzen:** Defekte Entities reparieren, Development-Debugging, System-Maintenance

### **`--siblings` - Context Analysis**
```bash
reed whois $main_content --siblings
```

**Zeigt Geschwister-Entities:** Welche anderen Snippets sind auf gleicher Hierarchie-Ebene? Rendering-Reihenfolge, Layout-Context.

**Nutzen:** Content-Anordnung verstehen, Layout-Debugging, Position-Optimierung

### **`--dependencies` - Asset Dependencies**
```bash
reed whois $complex_component --dependencies
```

**Zeigt alle Abhängigkeiten:** Template-Files, CSS-Dependencies, JavaScript-Components, Image-Assets, Plugin-Requirements.

**Nutzen:** Asset-Management, Deployment-Planung, Dependency-Tracking

### **`--history` - Change History**
```bash
reed whois $edited_snippet --history
```

**Zeigt Änderungshistorie:** Wer hat wann was geändert? Content-Diffs, Editor-Tracking, Rollback-Optionen.

**Nutzen:** Content-Auditing, Change-Management, Compliance-Dokumentation

---

## Marketing-Power: Warum das revolutionär ist

### **Für Agenturen**
**Problem:** Client ruft an: "Die Homepage ist kaputt, UUID 550e8400... zeigt Fehler"
**Lösung:** `reed whois 550e8400... --debug` → Sofortige Diagnose mit Repair-Steps

**Verkaufsargument:** "Wir debuggen Ihre Site in Sekunden, nicht Stunden"

### **Für Enterprises**
**Problem:** Performance-Issues, aber niemand weiß welche Komponenten schuld sind
**Lösung:** `reed whois --batch $all_components --performance` → Komplette Performance-Analyse

**Verkaufsargument:** "Proaktives Performance-Monitoring statt reaktive Feuerwehr"

### **Für Content-Teams**
**Problem:** "Kann ich diesen alten Banner löschen oder wird der noch gebraucht?"
**Lösung:** `reed whois $old_banner --references` → Komplette Impact-Analyse

**Verkaufsargument:** "Sicheres Content-Management ohne Versehens-Schäden"

### **Für Developers**
**Problem:** Neue Entwickler verstehen die Site-Struktur nicht
**Lösung:** `reed whois $any_component --epc` → Sofortige Architektur-Klarheit

**Verkaufsargument:** "Neue Team-Members produktiv in Stunden statt Wochen"

---

## Batch-Operations für Power-Users

### **Multiple Entities analysieren**
```bash
reed whois --batch $hero $nav $footer --performance
reed whois --batch content.1.* --health
reed whois --batch $(reed snippet list --unused) --references
```

**Power-Feature:** Hunderte Entities in einem Command analysieren. Perfekt für Site-Audits und Bulk-Operations.

### **Pipeline Integration**
```bash
# Performance-Report für alle Entities
reed whois --batch --all --performance --format toml > performance-report.toml

# Health-Check für CI/CD
reed whois --batch --all --health --format simple | grep "ERROR" && exit 1
```

**DevOps-Integration:** Automated Site-Health-Checks, Performance-Monitoring, Deployment-Validation.

---

## Warum das ein Verkaufsargument ist

### **Konkurrenzvorteil zu WordPress/Drupal**
"WordPress zeigt Ihnen `ID 1234` in Fehlermeldungen. ReedCMS zeigt Ihnen sofort: `$welcome_hero auf /homepage, erstellt von editor@firma.de, rendering-time 245ms, fix: enable image compression`"

### **Professionelles Site-Management** 
"Statt stundenlang nach der Ursache zu suchen, haben Sie in 5 Sekunden komplette Transparenz über jeden Site-Bestandteil"

### **Proaktive Wartung**
"Probleme erkennen bevor Users sie bemerken. Automated Health-Checks warnen Sie vor Content-Issues, Performance-Problemen und Security-Lücken"

### **Team-Effizienz**
"Neue Entwickler verstehen Ihre Site-Architektur sofort. Content-Editoren können sicher ändern ohne Angst etwas zu zerstören"

**Das `whois` Command macht ReedCMS vom normalen CMS zum professionellen Site-Intelligence-Tool.**

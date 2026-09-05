---
title: MVP Scope & Definition of Done (DoD)
type: Concept
description: Strategische Abgrenzung der ersten funktionsfähigen Version (MVP) zur Vermeidung von Feature-Creep.
status: active
created: 2026-09-05T13:05:00Z
timestamp: 2026-09-05T13:06:43Z
---

# MVP Scope & Definition of Done (DoD)

Dieses Dokument definiert den Umfang des **Minimum Viable Product (MVP)** für den "DVD & Data Catalog". Es dient als Leitplanke für die Entwicklung, um sicherzustellen, dass die Kernprobleme (Auffindbarkeit und physische Zuordnung) gelöst werden, ohne sich in unwichtigen Features zu verlieren.

## 1. Strategische Kernentscheidungen

### Scanner: Fast-Hash statt reiner Dateinamen-Scan
Ein reiner Name-/Pfad-Scan ist unzureichend für Backups.
* **Entscheidung:** Der Scanner berechnet ab Tag 1 einen **Fast-Hash** (Header 8KB + Footer 8KB + Dateigröße).
* **Grund:** Ermöglicht die Erkennung von Duplikaten (auch bei Umbenennung) mit minimalem Performance-Verlust auf optischen Medien.

### Duplikat-Handling: Erkennung statt Bereinigung
* **Entscheidung:** Im MVP wird die Existenz eines Duplikats (identischer Fast-Hash) lediglich durch ein **Warn-Icon** in der Suchergebnisliste angezeigt.
* **Abgrenzung:** Ein automatischer "Duplicate Cleaner" oder die Berechnung von Full-SHA-256-Hashes sind *Nice-to-Have* für spätere Versionen.

---

## 2. Scope-Matrix (Priorisierung)

| Bereich | Priorität | MVP-Funktionalität | Abgrenzung / Später |
| :--- | :--- | :--- | :--- |
| **Laufwerk-Scan** | <span style="color:green">**MUST**</span> | Rekursiv, Name/Pfad/Größe/Datum + Volume-UUID. | - |
| **Hashing** | <span style="color:green">**MUST**</span> | Fast-Hash (Header/Footer/Size). | Full-SHA-256 (Später) |
| **Standort** | <span style="color:green">**MUST**</span> | Freitext-Eingabe beim Scan. | Hierarchische Baumstruktur (Später) |
| **Suche** | <span style="color:green">**MUST**</span> | Echtzeit-Suche (Name/Pfad/Notiz). | - |
| **Notizen** | <span style="color:green">**MUST**</span> | Inline-Editor für Dateien & Ordner. | - |
| **Duplikate** | <span style="color:orange">**NICE**</span> | Anzeige eines Warn-Icons in der Liste. | Separates Dashboard (Später) |
| **Medien-Typ** | <span style="color:orange">**NICE**</span> | Unterscheidung BD-R, DVD, USB. | - |
| **Metadaten** | <span style="color:red">**OUT**</span> | Exif, Thumbnail, OCR. | **Explizit ausgeschlossen** (Performance!) |

---

## 3. Die "Definition of Done" (DoD)

Ein Feature oder die App gilt erst als "fertig", wenn folgende Kriterien erfüllt sind:

### A. System & Datenintegrität
* [ ] Electron-App läuft offline ohne externe Abhängigkeiten.
* [ ] DuckDB speichert `media`, `files` und `notes` persistent.
* [ ] IPC-Kommunikation ist asynchron und blockiert die UI nicht.

### B. Scan-Engine
* [ ] Volume-UUID wird erkannt (verhindert Double-Scans).
* [ ] Fast-Hash wird für jede Datei korrekt berechnet.
* [ ] Scan-Fortschritt (Prozent, aktuelle Datei) wird in Echtzeit angezeigt.
* [ ] Lesefehler führen nicht zum Absturz der App.

### C. UI & Suche
* [ ] Suchergebnisse erscheinen in < 50ms (DuckDB FTS).
* [ ] Notizen können direkt im Detailbereich erstellt/geändert werden.
* [ ] Physischer Standort ist bei jedem Suchtreffer prominent sichtbar.
* [ ] `Ctrl+F` / `Cmd+F` fokussiert das Suchfeld.

### D. Performance-Benchmarks
* [ ] Suche bleibt bei > 100.000 Dateien flüssig.
* [ ] RAM-Verbrauch im Leerlauf < 200 MB.

## timeline
- 2026-09-05T13:05:00Z: MVP Scope und DoD basierend auf Spezifikations-Dokument definiert.
EOF
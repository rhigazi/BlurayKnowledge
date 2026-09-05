---
title: Datenmodell: Zeitstempel & Metadaten-Strategie
type: Note
description: Spezifikation der zu erfassenden Zeitstempel für die Datei- und Medienindizierung in DuckDB.
status: active
created: 2026-09-05T06:56:08Z
timestamp: 2026-09-05T06:56:35Z
---

# Datenmodell: Zeitstempel & Metadaten-Strategie

Um eine präzise zeitliche Rekonstruktion von Backups und eine mächtige Suchfunktion zu ermöglichen, muss das System drei spezifische Zeitstempel unterscheiden und in der DuckDB-Datenbank speichern.

## 1. Die drei relevanten Zeitstempel

| Zeitstempel | Technischer Name | Bedeutung | Nutzen für den User |
| :--- | :--- | :--- | :--- |
| **Änderungsdatum** | `modified_at` (mtime) | Wann der *Inhalt* der Datei zuletzt bearbeitet wurde. | Identifikation der aktuellsten Version eines Dokuments. |
| **Erstellungsdatum** | `created_at` (btime) | Wann die Datei auf dem Dateisystem angelegt wurde. | Ermittlung des Ursprungszeitpunkts der Datei. |
| **Scan-Datum** | `scanned_at` | Wann das *Medium* in die Datenbank eingelesen wurde. | Nachverfolgung, wie aktuell der Katalogstand für ein Medium ist. |

## 2. Integration in das DuckDB-Datenmodell

Die Speicherung erfolgt über den Datentyp `TIMESTAMP` (für präzise Zeitpunkte) oder `DATE`.

### Medien-Tabelle (Media Table)
| Spalte | Typ | Beschreibung |
| :--- | :--- | :--- |
| `media_id` | INTEGER | Primärschlüssel |
| `label` | VARCHAR | Name des Mediums (z.B. "BACKUP_2023") |
| `location` | VARCHAR | Physischer Ort (z.B. "Regal 1, Box A") |
| **`scanned_at`** | **TIMESTAMP** | Zeitstempel des Scan-Vorgangs |

### Datei-Tabelle (Files Table)
| Spalte | Typ | Beschreibung |
| :--- | :--- | :--- |
| `file_id` | INTEGER | Primärschlüssel |
| `media_id` | INTEGER | Foreign Key zur Medien-Tabelle |
| `file_name` | VARCHAR | Name der Datei |
| `file_path` | VARCHAR | Relativer Pfad auf dem Medium |
| **`modified_at`** | **TIMESTAMP** | Letztes Änderungsdatum (mtime) |
| **`created_at`** | **TIMESTAMP** | Erstellungsdatum (btime) |

## 3. Anwendungsszenarien für die Suche

Durch diese Granularität werden mächtige Filterfunktionen möglich:

1.  **Zeitraum-Suche (Range Queries):**
    *   *"Finde alle Dateien, die zwischen 2015 und 2019 bearbeitet wurden."*
    *   `SELECT ... WHERE modified_at BETWEEN '2015-01-01' AND '2019-12-31'`
2.  **Versionskontrolle / Dubletten-Check:**
    *   Vergleich von Dateien mit gleichem Namen/Größe über verschiedene Medien hinweg anhand des `modified_at`.
3.  **Chronologische Sortierung:**
    *   Sortierung der Suchergebnisse nach dem tatsächlichen Alter der Dokumente, nicht nach dem Datum des Scans.

## timeline
- 2026-09-05T06:56:08Z: Definition der Zeitstempel-Strategie für die Datenbankstruktur.
EOF
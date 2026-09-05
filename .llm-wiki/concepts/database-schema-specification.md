---
title: Datenstruktur: Relationales ER-Modell für Medienkatalogisierung
type: Concept
description: Detailliertes Entity-Relationship-Modell und Tabellenspezifikation für die DuckDB-Implementierung der App.
status: active
created: 2026-09-05T06:57:52Z
timestamp: 2026-09-05T06:58:40Z
---

# Datenstruktur: Relationales ER-Modell für Medienkatalogisierung

Dieses Konzept definiert das Datenmodell, das auf **DuckDB** implementiert werden soll. Es kombiniert relationale Stabilität mit der notwendigen Flexibilität für manuelle Notizen und ultraschnelle Volltextsuche (FTS).

## 1. Entity-Relationship-Modell (ER-Modell)

Das Modell basiert auf fünf logischen Hauptkomponenten:

```text
┌────────────────────────────────┐
│           1. MEDIA             │  (Physischer Datenträger: DVD, BD-R, USB)
│  - media_id (PK)               │
│  - storage_location            │  (z. B. "Spindel 1, Platz 04")
│  - volume_label                │  (z. B. "BACKUP_2023")
│  - scanned_at                  │  (Zeitpunkt des Einlesens)
└───────────────┬────────────────┘
                │ 1
                │
                │ n
┌───────────────┴────────────────┐
│           2. FILES             │  (Dateien & Ordner)
│  - file_id (PK)                │
│  - media_id (FK)               │
│  - file_path                   │  (Relativer Pfad)
│  - file_name                   │
│  - file_size_bytes             │
│  - modified_at                 │  (Inhaltliche Änderung)
│  - created_at                 │  (Erstellung)
│  - is_directory                │
│  - fast_hash                  │  (Header/Footer/Size Hash)
│  - full_hash                   │  (Vollständiger BLAKE3 Hash)
│  - hash_verified_at           │  (Zeitpunkt der Vollverifikation)
│  - device_id                   │  (Identifikation für Hardlink-Erkennung)
│  - inode                       │  (Dateisystem Inode/FileIndex)
└───────────────┬────────────────┘
                │ 1
                │
                │ 0..1 (Optional)
┌───────────────┴────────────────┐
│           3. NOTES             │  (Manuelle Anreicherung)
│  - note_id (PK)                │
│  - file_id (FK)                │
│  - note_text                   │  (Semantischer Kontext)
│  - updated_at                  │
└────────────────────────────────┘

┌────────────────────────────────┐
│         4. FILES_FTS           │  (Virtueller Volltext-Index)
│  - Indiziert Name, Pfad & Note │
└────────────────────────────────┘

┌────────────────────────────────┐
│     5. DUPLICATE_EXCLUSIONS     │  (Manuelle Entwarnung)
│  - file_id_a (FK)              │
│  - file_id_b (FK)              │
│  - created_at                   │
└────────────────────────────────┘
```

## 2. Tabellenspezifikationen

### Tabelle 1: `media` (Datenträger & Standorte)

| Feld | Datentyp | Beschreibung / Beispiel |
| :--- | :--- | :--- |
| **`media_id`** | `INTEGER` (PK) | Eindeutige ID |
| **`volume_label`** | `VARCHAR` | Name des Mediums (z. B. `DOCS_2024`) |
| **`volume_uuid`** | `VARCHAR` | Eindeutige ID zur Vermeidung von Dubletten beim Scan |
| **`media_type`** | `VARCHAR` | Typ: `'BD-R'`, `'DVD'`, `'USB-HDD'`, etc. |
| **`storage_location`** | `VARCHAR` | Physischer Ort (z. B. `"Mappe A, Platz 12"`) |
| **`total_size_bytes`** | `UBIGINT` | Gesamtgröße des Datenträgers |
| **`scanned_at`** | `TIMESTAMP` | Zeitpunkt der Katalogisierung |

### Tabelle 2: `files` (Automatischer Dateibaum)

| Feld | Datentyp | Beschreibung / Beispiel |
| :--- | :--- | :--- |
| **`file_id`** | `BIGINT` (PK) | Eindeutige ID |
| **`media_id`** | `INTEGER` (FK) | Verweis auf `media` |
| **`file_path`** | `VARCHAR` | Relativer Pfad (z. B. `/Dokumente/Steuer/`) |
| **`file_name`** | `VARCHAR` | Dateiname inkl. Endung |
| **`extension`** | `VARCHAR` | Dateiendung (z. B. `pdf`, `zip`) |
| **`is_directory`** | `BOOLEAN` | `TRUE` für Ordner, `FALSE` für Dateien |
| **`file_size_bytes`** | `UBIGINT` | Größe in Bytes ($0$ bei Ordnern) |
| **`modified_at`** | `TIMESTAMP` | Letztes Änderungsdatum der Datei |
| **`created_at`** | `TIMESTAMP` | Erstellungsdatum der Datei |
| **`fast_hash`** | `VARCHAR` | Kombination aus Header, Footer und Size |
| **`full_hash`** | `VARCHAR` | Vollständiger BLAKE3/XXH3-Hash zur Verifikation |
| **`hash_verified_at`** | `TIMESTAMP` | Letzter Zeitpunkt der Full-Hash Prüfung |
| **`device_id`** | `VARCHAR` | Identifikation des Volumes (z. B. dev:ino) |
| **`inode`** | `BIGINT` | System-Inode zur Hardlink-Erkennung |

### Tabelle 3: `notes` (Manuelle Anreicherung)

| Feld | Datentyp | Beschreibung / Beispiel |
| :--- | :--- | :--- |
| **`note_id`** | `INTEGER` (PK) | Eindeutige ID |
| **`file_id`** | `BIGINT` (FK) | Verweis auf Datei/Ordner in `files` |
| **`note_text`** | `VARCHAR` | Manuelle Beschreibung (z. B. *"Urlaub Toskana"*) |
| **`tags`** | `VARCHAR[]` | Array von Schlagworten |
| **`updated_at`** | `TIMESTAMP` | Letzte Änderung der Notiz |

### Tabelle 4: `duplicate_exclusions` (Manuelle Entwarnung)

| Feld | Datentyp | Beschreibung / Beispiel |
| :--- | :--- | :--- |
| **`file_id_a`** | `BIGINT` (FK) | Erste Datei des Paares |
| **`file_id_b`** | `BIGINT` (FK) | Zweite Datei des Paares |
| **`created_at`** | `TIMESTAMP` | Zeitpunkt der Markierung als "Eindeutig" |

## 3. Such-Strategie (DuckDB FTS)

Um eine performante Suche über alle Ebenen zu gewährleisten, wird ein **Full Text Search (FTS) Index** auf eine kombinierte Sicht angewendet, die folgende Felder abdeckt:
1.  `file_name`
2.  `file_path`
3.  `note_text`

Dies ermöglicht es dem Nutzer, nach Begriffen zu suchen, die weder im Dateinamen noch im Pfad vorkommen, aber in einer Notiz hinterlegt wurden.

## 4. Vorteile des Modells

*   **Effizienz:** 99 % der Daten (Files) werden ohne manuellen Aufwand erfasst.
*   **Entkopplung:** Änderungen am physischen Standort eines Mediums (`media.storage_location`) wirken sich sofort auf alle verknüpften Dateien aus.
*   **Semantische Suche:** Die Verknüpfung von `files` und `notes` erlaubt die "intelligente" Suche nach Kontext (z. B. Suche nach "Toskana" findet den Ordner `/IMG_2020/`).
*   **Integrität:** Durch die Trennung von Fast-Hash und Full-Hash sowie der Exclusions-Tabelle ist das System gegen False Positives und Bit-Rot geschützt.

## timeline
- 2026-09-05T06:57:52Z: Finalisierung des relationalen Datenmodells und der Tabellenspezifikationen.
- 2026-09-05T14:30:00Z: Erweiterung des Modells um Kollisionsmanagement, Lazy Verification und Link-Erkennung.
EOF
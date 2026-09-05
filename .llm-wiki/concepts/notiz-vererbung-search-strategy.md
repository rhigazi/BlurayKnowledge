---
title: Semantische Verknüpfung & Sucharchitektur (Hybride Lösung)
type: Concept
description: Spezifikation der hybriden Notiz-Vererbung mittels target_path und STARTS_WITH für maximale Performance bei minimalem Schreibaufwand.
status: active
created: 2026-09-05T13:45:00Z
timestamp: 2026-09-05T13:25:12Z
---

# Semantische Verknüpfung & Sucharchitektur

Um die Herausforderung der Notiz-Vererbung (Notizen an Ordnern müssen für alle Unterdateien gelten) zu lösen, ohne die Performance oder die Datenintegrität zu opfern, wird ein **hybrider relationaler Ansatz** verwendet.

## 1. Das Problem: Denormalisierung vs. Rekursion

*   **Ansatz A (Denormalisierung):** Notizen werden auf jede Datei kopiert.
    *   *Problem:* Enormer Schreibaufwand bei Änderungen an Ordnern ($O(n)$) und Speicherverschwendung.
*   **Ansatz B (Rekursive Suche):** Notizen hängen nur am Ordner; die Suche muss zur Laufzeit den Pfad-Baum durchlaufen.
    *   *Problem:* Langsame Suche bei Millionen von Dateien durch teure rekursive Joins.

## 2. Die Lösung: Hybride Vererbung via `target_path`

Wir nutzen ein Modell, das die Notiz an ein Ziel bindet (`FILE`, `FOLDER` oder `MEDIA`), aber die Suche über hochperformante String-Operationen in DuckDB abwickelt.

### Datenmodell (Kern-Entitäten)

#### Tabelle: `files`
| Feld | Typ | Beschreibung |
| :--- | :--- | :--- |
| `id` | `VARCHAR` | PK |
| `file_path` | `VARCHAR` | Absoluter Pfad für Prefix-Matching |
| `parent_path` | `VARCHAR` | Direkter Elternordner |
| `fast_hash` | `VARCHAR` | Identifikation für Duplikate |

#### Tabelle: `notes`
| Feld | Typ | Beschreibung |
| :--- | :--- | :--- |
| `target_type` | `VARCHAR` | `'FILE' \| 'FOLDER' \| 'MEDIA'` |
| `target_path` | `VARCHAR` | Pfad-Präfix (z. B. `/Fotos/2023`) für Ordner-Notizen |
| `target_file_id`| `VARCHAR` | Bindung an spezifische Datei |
| `note_text` | `TEXT` | Der eigentliche Inhalt |

## 3. Such-Mechanismus (High-Performance Query)

Die Suche nutzt die Fähigkeit von DuckDB, `STARTS_WITH` Operationen auf Indizes extrem effizient auszuführen. Eine einzige Abfrage kombiniert drei Ebenen:

```sql
SELECT 
    f.*,
    COALESCE(fn.note_text, dn.note_text, mn.note_text) AS effective_note
FROM files f
JOIN media m ON f.media_id = m.id
-- 1. Ebene: Direkte Datei-Notiz
LEFT JOIN notes fn ON fn.target_type = 'FILE' AND fn.target_file_id = f.id
-- 2. Ebene: Vererbte Ordner-Notiz (Prefix Match)
LEFT JOIN notes dn ON dn.target_type = 'FOLDER' 
    AND dn.target_media_id = f.media_id 
    AND f.parent_path STARTS_WITH dn.target_path
-- 3. Ebene: Medien-Notiz
LEFT JOIN notes mn ON mn.target_type = 'MEDIA' AND mn.target_media_id = m.id
WHERE 
    f.file_name ILIKE '%' || $query || '%'
    OR fn.note_text ILIKE '%' || $query || '%'
    OR dn.note_text ILIKE '%' || $query || '%'
    OR mn.note_text ILIKE '%' || $query || '%';
```

## 4. Vorteile der Architektur

1.  **Schreib-Effizienz:** Eine Notiz an einem Ordner erfordert nur **einen** Datenbank-Eintrag ($O(1)$), auch wenn der Ordner Millionen Dateien enthält.
2.  **Lese-Performance:** Durch `STARTS_WITH` auf dem `parent_path` bleibt die Suche in der Millisekunden-Range.
3.  **Datenintegrität:** Bei einem Re-Scan werden nur die `files`-Einträge aktualisiert; die `notes`-Tabelle bleibt unberührt und muss nicht mühsam neu gematcht werden (außer bei Pfadänderungen).

## timeline
- 2026-09-05T13:45:00Z: Hybride Sucharchitektur und Notiz-Vererbung definiert.
EOF
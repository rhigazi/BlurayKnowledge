---
title: Gestuftes Verification-System: Fast-Hash, Candidate Matching & Deep Scan
type: Concept
description: Spezifikation eines 3-stufigen Verifikationsmodells zur fehlerfreien Duplikaterkennung unter Minimierung von I/O-Last.
status: active
created: 2026-09-05T14:45:00Z
timestamp: 2026-09-05T14:45:00Z
---

# Gestuftes Verification-System: Fast-Hash, Candidate Matching & Deep Scan

---

## 1. Die Architektur der gestuften Verifikation

Um trotz hoher Scan-Geschwindigkeit eine Trefferquote von **100% ohne Falsch-Positive** bei der Duplikaterkennung zu garantieren, darf der Fast-Hash niemals als finaler Beweis für Identität verwendet werden.

Wir implementieren ein **3-Stufen-Verifikationsmodell**, das den teuren I/O-Zugriff für den vollständigen Hash (Deep Scan) auf das absolut notwendige Minimum reduziert.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   GESTUFTES VERIFIKATIONS-MODELL                       │
│                                                                        │
│  STUFE 1: Exakte Dateigröße (SQL Aggregation)                          │
│  └── fast_size_bytes  ───────────────────────────► Einzigartig? ───► KEIN Duplikat
│                                                         │
│                                                     >1 Treffer
│                                                         ▼
│  STUFE 2: Fast-Hash (Header + Middle + Footer)                         │
│  └── fast_hash        ───────────────────────────► Abweichung? ──► KEIN Duplikat
│                                                         │
│                                                      Identisch
│                                                         ▼
│  STUFE 3: Deep Scan / On-Demand Full Hash                             │
│  └── full_hash (XXH3-128 / BLAKE3) ──────────────► Identisch? ───► GUARANTEED DUPLICATE
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Auslöser für den "Deep Scan" (Full-Hash Engine)

Der vollständige Hash wird **nicht** ungezielt beim ersten Durchscannen eines Datenträgers für alle Dateien berechnet, sondern getriggert durch zwei definierte Szenarien:

### Szenario A: Automatische Kandidaten-Verifikation (Kandidaten-Matching)

Sobald in DuckDB zwei oder mehr Dateien bezüglich **Dateigröße UND Fast-Hash** identisch sind, schaltet das System sie in den Status `POTENTIAL_DUPLICATE`.

* Die Hintergrund-Engine schickt **nur für diese betroffenen Dateien** eine Lese-Anforderung an den Worker.
* Es werden nur die vollen Hashes der Kandidaten berechnet und in `files.full_hash` eingetragen.

### Szenario B: User-Initiated & On-Action Verification

Führt der Nutzer im UI eine kritische Aktion aus (z. B. *"Duplikate löschen"*, *"Dateien zusammenführen"* oder *"Integritätsprüfung starten"*), verlangt die App vor der Ausführung zwingend einen erfolgreichen **Deep Scan Check** aller beteiligten Dateien.

---

## 3. Datenbank-Integration & Status-Tracking

Zur Unterscheidung zwischen unbestätigten Vermutungen und verifizierten Fakten ergänzen wir das Schema.

### Schema Erweiterung (`schema.sql`)

```sql
-- Erweiterung der files-Tabelle für gestufte Integrität
ALTER TABLE files ADD COLUMN IF NOT EXISTS full_hash VARCHAR DEFAULT NULL;
ALTER TABLE files ADD COLUMN IF NOT EXISTS hash_type VARCHAR DEFAULT NULL; -- 'XXH3_128', 'BLAKE3'
ALTER TABLE files ADD COLUMN IF NOT EXISTS duplicate_state VARCHAR DEFAULT 'UNCHECKED'; 
-- Mögliche Zustände: 'UNCHECKED', 'UNIQUE', 'POTENTIAL_DUPLICATE', 'CONFIRMED_DUPLICATE'

CREATE INDEX IF NOT EXISTS idx_files_duplicate_state ON files(duplicate_state);
CREATE INDEX IF NOT EXISTS idx_files_full_hash ON files(full_hash);
```

---

## 4. SQL-Pipeline für automatische Dubletten-Verifikation

Mit dieser SQL-Abfrage identifiziert der Main Process effizient alle Dateien, für die ein Deep Scan im Hintergrund angestoßen werden muss:

```sql
-- Identifiziere alle Kandidaten, die dieselbe Größe und denselben Fast-Hash haben,
-- deren Full-Hash aber noch NICHT berechnet wurde:

WITH duplicate_candidates AS (
    SELECT 
        fast_hash, 
        file_size_bytes
    FROM files
    WHERE file_size_bytes > 0
    GROUP BY fast_hash, file_size_bytes
    HAVING COUNT(*) > 1
)
SELECT 
    f.id,
    f.media_id,
    f.file_path,
    f.file_size_bytes
FROM files f
JOIN duplicate_candidates c 
  ON f.fast_hash = c.fast_hash 
 AND f.file_size_bytes = c.file_size_bytes
WHERE f.full_hash IS NULL; -- Noch kein Deep Scan erfolgt
```

---

## 5. UI-Visualisierung im Frontend

In der Benutzeroberfläche werden potenzielle und verifizierte Duplikate klar unterschieden, um absolute Transparenz zu schaffen:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        DUPLIKAT-FINDER RESULTATE                       │
│                                                                        │
│  [?] 2x Video_2024.mp4 (4.2 GB)                                       │
│      Status: Verdacht (Fast-Hash identisch)                            │
│      [ Deep Scan ausführen (Full Hash) ]                               │
│                                                                        │
│  [✓] 2x Dokument_Backup.pdf (12.4 MB)                                  │
│      Status: 100% Bestätigt (XXH3-128 Match)                           │
│      [ Zusammenführen / Löschen ]                                     │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Warn-Symbol / Gelb:** Fast-Hash stimmt überein, Full-Hash fehlt noch. Lösch-Aktionen sind blockiert oder erfordern eine explizite Bestätigung mit automatischem Deep-Scan-Anlauf.
2. **Grünes Häkchen / Grün:** `full_hash` wurde berechnet und stimmt überein. Aktionen können gefahrlos ausgeführt werden.

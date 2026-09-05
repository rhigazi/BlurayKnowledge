---
title: Datenmodell & Schema: Unified Design & Metadata Strategy
type: Concept
description: Spezifikation des relationalen Datenmodells unter Nutzung von DuckDB-Stärken (Columnar Storage & Native JSON) für maximale Performance und Flexibilität.
status: active
created: 2026-09-05T15:30:00Z
timestamp: 2026-09-05T15:30:00Z
---

# Datenmodell & Schema: Unified Design & Metadata Strategy

---

## 1. Unified Schema Design (Zusammenführung)

Anstatt verifikations- und hashspezifische Daten in eine separate Relationaltabelle (`file_verification_details`) auszulagern, konsolidieren wir alle physischen, strukturellen und prüfsummenrelevanten Attribute direkt in eine primäre `files`-Tabelle innerhalb von **DuckDB**.

### A. Architekturvorteile des Unified Designs

* **Zero Join Overhead:** DuckDB ist eine spaltenbasierte (columnar) OLAP-Datenbank. Das Zusammenführen vermeidet teure `JOIN`-Operationen bei Such- und Aggregationsabfragen über Millionen von Dateien.
* **Transaktionale Integrität:** Änderungen an Metadaten, Hash-Werten und Verifikationszuständen erfolgen atomar in einem einzigen Zeilen-Update (`UPSERT` / `UPDATE`).
* **Vektorisiertes Scanning:** Spalten wie `duplicate_state` oder `fast_hash` können von der Vektor-Engine direkt im Speicher ohne Tabellentransformationen verarbeitet werden.

### B. Das finale `files`-Tabellenschema (`schema.sql`)

```sql
-- Aktivierung von UUID-Unterstützung falls nötig
CREATE SEQUENCE IF NOT EXISTS seq_files_id;

CREATE TABLE IF NOT EXISTS files (
    -- 1. Eindeutige Identifikatoren & Relationaler Kontext
    id VARCHAR PRIMARY KEY,                  -- UUIDv4 / Hash-basierte ID
    media_id VARCHAR NOT NULL,               -- Foreign Key zum Medium (media.id)
    
    -- 2. Pfad- & Dateisystem-Struktur
    file_name VARCHAR NOT NULL,              -- Dateiname inkl. Erweiterung (z.B. "film.mkv")
    file_path VARCHAR NOT NULL,              -- Vollständiger relativer/absoluter Pfad
    parent_path VARCHAR NOT NULL,           -- Vaterverzeichnis für schnelle Hierarchiesuche
    
    -- 3. Physische Attribute
    file_size_bytes BIGINT NOT NULL,         -- Tatsächliche Dateigröße in Bytes
    created_at TIMESTAMP,                    -- Erstellungsdatum laut Dateisystem
    mtime TIMESTAMP NOT NULL,                 -- Letzte Änderung (Modification Time)
    is_hardlink BOOLEAN DEFAULT FALSE,       -- Marker für Hardlink-Deduplizierung
    
    -- 4. Ingestierte Prüfsummen & Verifikation (Ehemals file_verification_details)
    fast_hash VARCHAR,                       -- Header + Middle + Footer Hash (3x 64KB)
    full_hash VARCHAR DEFAULT NULL,          -- Vollständiger Inhaltshash (XXH3_128 / BLAKE3)
    hash_type VARCHAR DEFAULT NULL,          -- Hash-Algorithmus (z. B. 'XXH3_128', 'BLAKE3')
    hash_verified_at TIMESTAMP DEFAULT NULL,  -- Zeitstempel des letzten Lazy Full-Hash Checks
    
    -- 5. Zustandsverwaltung & Integrität
    duplicate_state VARCHAR DEFAULT 'UNCHECKED', 
    -- Werte: 'UNCHECKED', 'UNIQUE', 'POTENTIAL_DUPLICATE', 'CONFIRMED_DUPLICATE'
    
    lifecycle_status VARCHAR DEFAULT 'ACTIVE',
    -- Werte: 'ACTIVE', 'UNVERIFIED', 'ARCHIVED', 'MISSING'

    -- 6. Flexible Metadaten (Sparse Container)
    metadata JSON DEFAULT NULL               -- Exif/ID3/Codec-Metadaten als JSON
);

-- Performante Indizes für Kernoperationen
CREATE UNIQUE INDEX IF NOT EXISTS idx_files_media_path ON files(media_id, file_path);
CREATE INDEX IF NOT EXISTS idx_files_lookup ON files(file_size_bytes, fast_hash);
CREATE INDEX IF NOT EXISTS idx_files_duplicate_state ON files(duplicate_state);
CREATE INDEX IF NOT EXISTS idx_files_parent_path ON files(parent_path);
```

---

## 2. Metadata Strategy (Sparse-Struktur via Native JSON)

Da je nach Dateityp völlig unterschiedliche Zusatzinformationen existieren (z. B. EXIF für Fotos, ID3 für MP3s, Video-Codecs für MKVs oder Gar keine für Plaintext-Dateien), würde ein klassisches relationales Schema zu Hunderten meist leeren (`NULL`) Spalten führen.

Wir setzen auf eine **Sparse Metadata Strategy**, die native JSON-Unterstützung von DuckDB nutzt.

### A. Struktur des Sparse `metadata`-JSON-Feldes

Nur wenn bei der Ingestion oder beim On-Demand-Parsing Metadaten extrahiert werden, wird das JSON-Feld befüllt.

#### Beispiel 1: Bilddatei (`.jpg`)

```json
{
  "type": "image",
  "width": 3840,
  "height": 2160,
  "camera": "Sony A7IV",
  "iso": 100,
  "focal_length": "35mm",
  "taken_at": "2024-08-15T14:22:10Z"
}
```

#### Beispiel 2: Audiodatei (`.flac`)

```json
{
  "type": "audio",
  "artist": "Daft Punk",
  "album": "Random Access Memories",
  "duration_seconds": 368,
  "bitrate_kbps": 920,
  "sample_rate": 44100
}
```

#### Beispiel 3: Standard-Datei (`.bin` / `.iso`)

```sql
metadata = NULL -- Verbraucht exakt 0 Bytes zusätzlichen Speicher
```

### B. Abfrage-Performance auf JSON-Spalten in DuckDB

DuckDB kann JSON-Pfade mit Pfeil-Operatoren (`->` und `->>`) extrem schnell und inkrementell abfragen. Es erlaubt sogar das Erstellen von virtuellen/Indizes auf JSON-Feldern:

```sql
-- Beispielsuche: Finde alle 4K-Bilder mit ISO <= 200 quer über alle Medien
SELECT 
    file_name, 
    file_path,
    metadata->>'camera' AS camera_model,
    CAST(metadata->>'iso' AS INTEGER) AS iso_value
FROM files
WHERE metadata->>'type' = 'image'
  AND CAST(metadata->>'width' AS INTEGER) >= 3840
  AND CAST(metadata->>'iso' AS INTEGER) <= 200;
```

### C. Speicher- & Zukunftsfähigkeit

* **Keine Migrationen bei neuen Metadaten-Feldern:** Kommen neue Felder hinzu (z. B. PDF-Seitenzahlen oder Geolocation-Daten), muss das SQL-Schema nicht angepasst werden (`ALTER TABLE` entfällt).
* **Sparse Column Compression:** DuckDB komprimiert `NULL`-Werte und JSON-Strings im spaltenbasierten Speicherformat highly efficient. Für Dateien ohne Extrametadaten entsteht kein Speicherplatz-Overhead.

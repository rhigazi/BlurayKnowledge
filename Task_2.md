# Task 2: DuckDB Schema & Datenbank Manager

## Ziel
Implementiere das konsolidierte DuckDB Datenbankschema und die Datenbankschicht im Main Process. Alle Tabellen, Indizes und Sequences werden via Schema SQL definiert und beim App Start initialisiert.

## Schritte

### 2.1 Schema SQL Datei
Schreibe `src/main/db/schema.sql` mit den Tabellen aus der Unified Schema Spezifikation:

**Tabelle `media`** (Datentraeger & Standorte):
```sql
CREATE TABLE IF NOT EXISTS media (
    media_id INTEGER PRIMARY KEY,
    volume_label VARCHAR NOT NULL,
    volume_uuid VARCHAR,
    media_type VARCHAR,          -- 'BD-R', 'DVD', 'USB-HDD'
    storage_location VARCHAR,    -- physischer Ort
    total_size_bytes UBIGINT,
    lifecycle_status VARCHAR DEFAULT 'ACTIVE',  -- ACTIVE, UNVERIFIED, ARCHIVED, MISSING
    scanned_at TIMESTAMP
);
```

**Tabelle `files`** (konsolidierte Datei Tabelle mit allen Attributen):
```sql
CREATE TABLE IF NOT EXISTS files (
    id VARCHAR PRIMARY KEY,
    media_id INTEGER NOT NULL REFERENCES media(media_id),
    file_name VARCHAR NOT NULL,
    file_path VARCHAR NOT NULL,
    parent_path VARCHAR NOT NULL,
    extension VARCHAR,
    is_directory BOOLEAN DEFAULT FALSE,
    file_size_bytes BIGINT NOT NULL,
    created_at TIMESTAMP,
    mtime TIMESTAMP NOT NULL,
    is_hardlink BOOLEAN DEFAULT FALSE,
    device_id VARCHAR,
    inode BIGINT,
    fast_hash VARCHAR,
    full_hash VARCHAR DEFAULT NULL,
    hash_type VARCHAR DEFAULT NULL,
    hash_verified_at TIMESTAMP DEFAULT NULL,
    duplicate_state VARCHAR DEFAULT 'UNCHECKED',  -- UNCHECKED, UNIQUE, POTENTIAL_DUPLICATE, CONFIRMED_DUPLICATE
    lifecycle_status VARCHAR DEFAULT 'ACTIVE',     -- ACTIVE, UNVERIFIED, ARCHIVED, MISSING
    metadata JSON DEFAULT NULL
);
```

**Tabelle `notes`** (manuelle Anreicherung):
```sql
CREATE TABLE IF NOT EXISTS notes (
    note_id INTEGER PRIMARY KEY,
    target_type VARCHAR NOT NULL,   -- FILE, FOLDER, MEDIA
    target_path VARCHAR,
    target_file_id VARCHAR,
    target_media_id INTEGER,
    note_text TEXT NOT NULL,
    tags VARCHAR[],
    updated_at TIMESTAMP
);
```

**Tabelle `duplicate_exclusions`** (manuelle Entwarnung):
```sql
CREATE TABLE IF NOT EXISTS duplicate_exclusions (
    file_id_a BIGINT NOT NULL,
    file_id_b BIGINT NOT NULL,
    created_at TIMESTAMP,
    PRIMARY KEY (file_id_a, file_id_b)
);
```

**Indizes fuer Kernoperationen:**
```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_files_media_path ON files(media_id, file_path);
CREATE INDEX IF NOT EXISTS idx_files_lookup ON files(file_size_bytes, fast_hash);
CREATE INDEX IF NOT EXISTS idx_files_duplicate_state ON files(duplicate_state);
CREATE INDEX IF NOT EXISTS idx_files_parent_path ON files(parent_path);
CREATE INDEX IF NOT EXISTS idx_files_full_hash ON files(full_hash);
CREATE INDEX IF NOT EXISTS idx_notes_target ON notes(target_type, target_path);
CREATE INDEX IF NOT EXISTS idx_media_volume_uuid ON media(volume_uuid);
```

### 2.2 Database Manager Klasse
Schreibe `src/main/db/database.ts`:

- Klasse `DatabaseManager` die eine DuckDB Instanz verwaltet
- `getInstance()` Singleton Methode
- `initialize()`: Liest `schema.sql`, fuehrt es aus, erzeugt die Tabellen
- `getConnection()`: Gibt die native DuckDB Verbindung zurueck
- `close()`: Sauberes Schliessen der Datenbank
- Datenbank Datei: `catalog.duckdb` im Benutzer Daten Verzeichnis (via `app.getPath('userData')`)

### 2.3 In-Memory Mode fuer Tests
- Option `:memory:` Datenbank fuer Unit Tests (ohne Datei I/O)
- Wird ueber Konstruktor Parameter oder Environment Variable gesteuert

### 2.4 Schema Versionierung
- Tabelle `_schema_version` mit `version INTEGER` und `migrated_at TIMESTAMP`
- Bei App Start: Pruefe Version, wende Migrationen nur an wenn veraetert
- Version 1: Das oben definierte Schema

### 2.5 Prepared Statement Factory
- `src/main/db/statements.ts` mit wiederverwendbaren Prepared Statements
- `insertFile`, `updateFileHash`, `findDuplicateCandidates`, `searchFiles`, etc.
- Jedes Statement wird einmal vorbereitet (`connection.prepare()`) und wiederverwendet

## Akzeptanzkriterien
- [ ] `DatabaseManager.initialize()` erzeugt alle 4 Tabellen + Indizes korrekt
- [ ] Ein INSERT in `files` mit allen Feldern funktioniert und kann zurueckgelesen werden
- [ ] JSON Spalte `metadata` kann geschrieben und via DuckDB JSON Operatoren abgefragt werden
- [ ] `:memory:` Mode funktioniert ohne Dateisystem Zugriff
- [ ] Schema Versionierung verhindert Doppelt Migrationen
- [ ] Prepared Statements sind typsicher und wiederverwendbar

## Abgrenzung
- Noch keine Geschaeftslogik (Scan, Hash)
- Keine IPC Handler fuer den Renderer
- DuckDB wird initialisiert, die App kann starten ohne zu crashen
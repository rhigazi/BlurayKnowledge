# Task 8: Notes, Search & Notiz Vererbung

## Ziel
Implementiere das Notizen System mit hierarchischer Vererbung (FILE > FOLDER > MEDIA) und die hybride Volltextsuche ueber DuckDB FTS und STARTS_WITH Prefix Matching.

## Schritte

### 8.1 Notes CRUD Operations
Schreibe `src/main/services/notesService.ts`:

- `createNote(targetType, targetPath, targetFileId, targetMediaId, noteText, tags)`: INSERT in `notes`
- `updateNote(noteId, noteText, tags)`: UPDATE note_text, tags, updated_at
- `deleteNote(noteId)`: DELETE FROM notes WHERE note_id = ?
- `getNotesForFile(fileId)`: Alle Notizen fuer eine Datei, inklusive geerbter
- `getNotesForFolder(folderPath, mediaId)`: Ordnernotizen via Prefix Match

### 8.2 Notiz Vererbung SQL
Implementiere die hybride Vererbung mit `STARTS_WITH` und `COALESCE` Priorisierung:

```sql
SELECT f.*,
       COALESCE(
           fn.note_text,      -- 1. Prioritaet: Datei-Notiz
           dn.note_text,      -- 2. Prioritaet: Ordner-Notiz
           mn.note_text       -- 3. Prioritaet: Medien-Notiz
       ) AS effective_note,
       COALESCE(
           fn.tags,
           dn.tags,
           mn.tags
       ) AS effective_tags
FROM files f
LEFT JOIN notes fn ON fn.target_type = 'FILE' AND fn.target_file_id = f.id
LEFT JOIN notes dn ON dn.target_type = 'FOLDER'
    AND dn.target_media_id = f.media_id
    AND f.parent_path STARTS_WITH dn.target_path
LEFT JOIN notes mn ON mn.target_type = 'MEDIA' AND mn.target_media_id = f.media_id
WHERE f.media_id = ?
  AND (? IS NULL OR f.file_name ILIKE '%' || ? || '%');
```

**Prioritaetsreihenfolge:** FILE (hoechste) > FOLDER > MEDIA (niedrigste)

### 8.3 FTS Index Setup
Schreibe `src/main/db/fts-setup.sql`:

```sql
-- Erstelle FTS Virtuelle Tabelle ueber die relevanten Felder
INSTALL fts;
LOAD fts;

CREATE OR REPLACE VIEW files_search_view AS
SELECT
    f.id,
    f.file_name,
    f.file_path,
    f.parent_path,
    f.media_id,
    f.extension,
    f.file_size_bytes,
    f.fast_hash,
    f.duplicate_state,
    COALESCE(fn.note_text, dn.note_text, mn.note_text) AS note_text,
    m.volume_label,
    m.storage_location,
    m.media_type
FROM files f
JOIN media m ON f.media_id = m.media_id
LEFT JOIN notes fn ON fn.target_type = 'FILE' AND fn.target_file_id = f.id
LEFT JOIN notes dn ON dn.target_type = 'FOLDER'
    AND dn.target_media_id = f.media_id
    AND f.parent_path STARTS_WITH dn.target_path
LEFT JOIN notes mn ON mn.target_type = 'MEDIA' AND mn.target_media_id = m.media_id;

-- Erstelle FTS Index auf der Sicht
PRAGMA create_fts_index('files_search_view', 'id', 'file_name', 'file_path', 'note_text');
```

### 8.4 Search Service
Schreibe `src/main/services/searchService.ts`:

```typescript
export class SearchService {
  constructor(private db: DatabaseManager) {}

  async search(query: string, filters?: SearchFilters): Promise<SearchResult[]> {
    // Nutze DuckDB FTS fuer die Volltextsuche
    const ftsQuery = `
      SELECT f.*,
             score(?) as relevance
      FROM files_fts(?)
      JOIN files f ON f.id = files_fts.id
      WHERE files_fts MATCH ?
      ORDER BY relevance DESC
      LIMIT ?;
    `;

    // Fallback fuer einfache ILIKE Suche (wenn FTS nicht installiert)
    const fallbackQuery = `
      SELECT f.*, 0.0 as relevance
      FROM files f
      WHERE f.file_name ILIKE '%' || ? || '%'
         OR f.file_path ILIKE '%' || ? || '%'
         OR f.parent_path ILIKE '%' || ? || '%'
      ORDER BY f.file_size_bytes DESC
      LIMIT ?;
    `;

    // Wende Filter an
    if (filters?.mediaId) { /* AND f.media_id = ? */ }
    if (filters?.extension) { /* AND f.extension = ? */ }
    if (filters?.minSize) { /* AND f.file_size_bytes >= ? */ }
    if (filters?.maxSize) { /* AND f.file_size_bytes <= ? */ }
    if (filters?.duplicateState) { /* AND f.duplicate_state = ? */ }
    if (filters?.dateFrom) { /* AND f.mtime >= ? */ }

    const results = await this.db.query(ftsQuery, [query, query, query, 100]);
    return results.map(this.toSearchResult);
  }

  async searchByFolder(folderPath: string, mediaId: number): Promise<FileEntry[]> {
    // Nutze parent_path fuer schnelle Ordner Navigation
    return this.db.query(
      `SELECT * FROM files WHERE parent_path = ? AND media_id = ? ORDER BY is_directory DESC, file_name`,
      [folderPath, mediaId]
    );
  }
}
```

### 8.5 Fuzzy Search (Optional-Fallback)
- Nutze DuckDB `levenshtein()` Funktion fuer Tippfehler Toleranz
- Aktiviert nur wenn FTS Suche keine Ergebnisse liefert
- Beispiel: `WHERE levenshtein(f.file_name, ?) <= 2`
- Grenze: Max 2 Zeichen Abstand, sonst zu teuer

### 8.6 IPC Handler Integration
- `dbHandler.ts`: Rufe `SearchService.search()` auf
- `noteHandler.ts`: Rufe `NotesService` CRUD Methoden auf
- Liefere Ergebnisse als getypte Objekte an den Renderer

### 8.7 Optimierung: Cached Search
- `searchCache.ts`: Simple In-Memory Cache mit 5 Minuten TTL
- `Map<string, { results: SearchResult[], cachedAt: number }>`
- Bei identischer Query innerhalb TTL: Gib gecachte Ergebnisse zurueck
- Invalidierung: Nach Notes Update, nach Scan Neueintrag

## Akzeptanzkriterien
- [ ] Notes koennen fuer Dateien, Ordner und Medien angelegt werden
- [ ] Folder Notes werden an alle Unterdateien vererbt (STARTS_WITH)
- [ ] FILE Notizen ueberschreiben FOLDER Notizen (Prioritaet korrekt)
- [ ] DuckDB FTS Index ist installiert und liefert Ergebnisse
- [ ] Suche nach Dateiname, Pfad UND Notiztext funktioniert
- [ ] ILIKE Fallback funktioniert wenn FTS nicht verfuegbar
- [ ] Filter (Media, Extension, Groesse, Datum) werden korrekt angewendet
- [ ] Suche bleibt unter 50ms bei 100.000+ Dateien
- [ ] Suche mit Tippfehlern findet trotzdem Ergebnisse (Fuzzy/Levenshtein)

## Abgrenzung
- UI Search View kommt in Task 11
- Keine Volltext-Suche von Dateiinhalten (nur Dateiname + Pfad + Notiz)
- Cache wird beim App Start geleert
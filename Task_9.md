# Task 9: Media Lifecycle, Delta-Scan & Duplikat Exclusions

## Ziel
Implementiere den Medien Lebenszyklus (ACTIVE/UNVERIFIED/ARCHIVED/MISSING), den schnellen Delta-Scan fuer wiederkehrende Medien und das False-Positive Handling via Duplikat Exclusions.

## Schritte

### 9.1 Media Lifecycle Status
Schreibe `src/main/services/mediaService.ts`:

- `createMedia(volumeLabel, volumeUuid, mediaType, storageLocation)`: Legt neuen Medien Eintrag an
- `updateMedia(mediaId, updates)`: Aendert storage_location, media_type, etc.
- `archiveMedia(mediaId)`: Setzt lifecycle_status = 'ARCHIVED', Daten bleiben erhalten
- `markMissing(mediaId)`: Setzt lifecycle_status = 'MISSING'
- `reactivate(mediaId)`: Setzt lifecycle_status = 'ACTIVE'

**Automatische Status Uebergaenge:**
- Nach Scan: `lifecycle_status = 'ACTIVE'`, `scanned_at = NOW()`
- Wenn > 30 Tage seit Scan (nur schreibbare Medien): `UNVERIFIED`
- Nach Quick-Verify (Volume UUID Match): `ACTIVE`, updated `scanned_at`
- Wenn Volume beim Scan nicht gefunden: User kann `MISSING` setzen

### 9.2 Delta-Scan Mechanismus
Schreibe `src/main/services/deltaScan.ts`:

```typescript
export class DeltaScanEngine {
  async runDeltaScan(mediaId: number, rootPath: string): Promise<DeltaScanResult> {
    const db = DatabaseManager.getInstance();

    // 1. Erstelle temporaere Staging Tabelle
    await db.exec('CREATE TEMP TABLE IF NOT EXISTS _delta_stage (LIKE files);');

    // 2. Fuelle Stage mit aktuellen Dateisystem Daten
    //    (Traversierung OHNE Hash Berechnung fuer Speed)
    //    Erfasst nur: path, name, size, mtime
    const stageEntries = await this.traverseForDelta(rootPath);
    await this.bulkInsertStage(db, stageEntries);

    // 3. Metadaten Vergleich
    const changes = await db.query(`
      -- Neue / Geanderte Dateien
      SELECT s.file_path, s.file_name, s.file_size_bytes, s.mtime, 'NEW' as change_type
      FROM _delta_stage s
      LEFT JOIN files f ON f.media_id = ? AND f.file_path = s.file_path
      WHERE f.id IS NULL
        OR f.file_size_bytes != s.file_size_bytes
        OR f.mtime != s.mtime

      UNION ALL

      -- Geloschte Dateien
      SELECT f.file_path, f.file_name, f.file_size_bytes, f.mtime, 'DELETED' as change_type
      FROM files f
      LEFT JOIN _delta_stage s ON s.file_path = f.file_path
      WHERE f.media_id = ? AND s.file_path IS NULL
    `, [mediaId, mediaId]);

    // 4. Wende Aenderungen an (innerhalb einer Transaktion)
    const deletedCount = changes.filter(c => c.change_type === 'DELETED').length;
    const newOrChangedCount = changes.filter(c => c.change_type === 'NEW').length;

    if (changes.length > 0) {
      await db.exec('BEGIN TRANSACTION;');
      // Loesche nicht mehr existente Dateien
      await db.exec(`DELETE FROM files WHERE media_id = ? AND file_path IN (
        SELECT file_path FROM _delta_stage WHERE change_type = 'DELETED'
      )`, [mediaId]);
      // Markiere UNCHECKED fuer neue/geanderte (Hash muss neu berechnet)
      await db.exec(`UPDATE files SET duplicate_state = 'UNCHECKED', fast_hash = NULL
        WHERE media_id = ? AND file_path IN (...)`, [mediaId]);
      await db.exec('COMMIT;');
    }

    await db.exec('DROP TABLE IF EXISTS _delta_stage;');
    return { deletedCount, newOrChangedCount, unchanged: stageEntries.length - changes.length };
  }
}
```

### 9.3 Delta Scan Optimierung
- OHNE Fast-Hash Berechnung fuer unveraenderte Dateien (nur Metadaten Vergleich)
- Wenn `file_size` UND `mtime` identisch: Datei gilt als unveraendert
- Hash wird NUR fuer neue oder geanderte Dateien berechnet (spaeter via Fast-Hash Task)
- Typische Laufzeit: < 1 Sekunde fuer 10.000 Dateien (reine Metadaten)

### 9.4 Quick-Verify via Volume UUID
- Bevor Scan startet: `SELECT media_id FROM media WHERE volume_uuid = ?`
- Wenn gefunden: `UPDATE media SET scanned_at = NOW(), lifecycle_status = 'ACTIVE'`
- Nur Lebenszyklus aktualisieren, kein erneuter Scan noetig
- Bei `media_type` IMMUTABLE (DVD/BD-R): Keine weiteren Checks, sofort Quick-Verify

### 9.5 Duplikat Exclusions (False-Positive)
Schreibe `src/main/services/exclusionService.ts`:

```typescript
export class ExclusionService {
  // User sagt: "Diese zwei Dateien sind KEINE Duplikate, auch wenn der Hash matcht"
  async markAsFalsePositive(fileIdA: string, fileIdB: string): Promise<void> {
    const db = DatabaseManager.getInstance();
    await db.exec(
      `INSERT INTO duplicate_exclusions (file_id_a, file_id_b, created_at)
       VALUES (?, ?, NOW())`, [fileIdA, fileIdB]
    );
  }

  // Pruefe ob ein Paar ausgeschlossen ist
  async isExcluded(fileIdA: string, fileIdB: string): Promise<boolean> {
    const result = await DatabaseManager.getInstance().query(
      `SELECT 1 FROM duplicate_exclusions
       WHERE (file_id_a = ? AND file_id_b = ?)
          OR (file_id_a = ? AND file_id_b = ?)`,
      [fileIdA, fileIdB, fileIdB, fileIdA]
    );
    return result.length > 0;
  }

  // Entferne Ausschluss
  async removeExclusion(fileIdA: string, fileIdB: string): Promise<void> {
    await DatabaseManager.getInstance().exec(
      `DELETE FROM duplicate_exclusions
       WHERE (file_id_a = ? AND file_id_b = ?)
          OR (file_id_a = ? AND file_id_b = ?)`,
      [fileIdA, fileIdB, fileIdB, fileIdA]
    );
  }
}
```

### 9.6 SQL: Exclusions in Duplikat Query integrieren
```sql
-- Finde Duplikat Kandidaten, aber schliesse manuelle Exclusions aus
WITH pot_dups AS (
    SELECT fast_hash, file_size_bytes, COUNT(*) as cnt
    FROM files
    WHERE fast_hash IS NOT NULL AND duplicate_state != 'UNIQUE'
    GROUP BY fast_hash, file_size_bytes
    HAVING COUNT(*) > 1
)
SELECT f.*
FROM files f
JOIN pot_dups p ON f.fast_hash = p.fast_hash AND f.file_size_bytes = p.file_size_bytes
WHERE NOT EXISTS (
    SELECT 1 FROM duplicate_exclusions de
    WHERE (de.file_id_a = f.id OR de.file_id_b = f.id)
)
ORDER BY f.file_size_bytes DESC;
```

## Akzeptanzkriterien
- [ ] Media Eintraege koennen erstellt, archiviert und als MISSING markiert werden
- [ ] Delta-Scan erkennt neue, geaenderte und geloeschte Dateien ohne Hash
- [ ] Unveraenderte Dateien werden beim Delta-Scan uebersprungen
- [ ] Quick-Verify aktualisiert `scanned_at` bei Volume UUID Match
- [ ] IMMUTABLE Medien (DVD/BD-R) erhalten keine Veraltungs Warnung
- [ ] Duplikat Exclusions verhindern False-Positives in der Duplikat Liste
- [ ] Exclusions sind bidirektional (file_id_a + file_id_b unabhaengig von Reihenfolge)
- [ ] Nach Scan oder Delta-Scan wird `scanned_at` aktualisiert

## Abgrenzung
- UI fuer Media Management kommt in Task 14
- UI fuer Duplikat Management kommt in Task 15
- Auto-Pause bei OS Activity kommt als separater Task
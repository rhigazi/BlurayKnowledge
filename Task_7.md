# Task 7: 3-Stufen Duplikat Verifikation & Full-Hash Engine

## Ziel
Implementiere das gestufte Verifikationssystem zur Duplikaterkennung: Stufe 1 (Groessenabgleich) -> Stufe 2 (Fast-Hash Matching) -> Stufe 3 (Full-Hash Deep Scan). Kein Fast-Hash wird als finaler Beweis verwendet.

## Schritte

### 7.1 Duplikat Kandidaten SQL (Stufen 1 und 2)
Schreibe `src/main/services/duplicateDetector.ts`:

**Stufe 1: Exakte Dateigroesse**
```sql
SELECT file_size_bytes, COUNT(*) as cnt
FROM files
WHERE file_size_bytes > 0 AND is_directory = FALSE
GROUP BY file_size_bytes
HAVING COUNT(*) > 1;
```

**Stufe 2: Fast-Hash Matching (innerhalb gleicher Groesse)**
```sql
WITH size_dups AS (
    SELECT file_size_bytes FROM files
    WHERE file_size_bytes > 0 AND is_directory = FALSE
    GROUP BY file_size_bytes HAVING COUNT(*) > 1
)
SELECT f.id, f.media_id, f.file_path, f.file_name,
       f.file_size_bytes, f.fast_hash, f.duplicate_state
FROM files f
JOIN size_dups s ON f.file_size_bytes = s.file_size_bytes
WHERE f.fast_hash IS NOT NULL
ORDER BY f.file_size_bytes DESC, f.fast_hash;
```

**Markiere POTENTIAL_DUPLICATE:**
```sql
UPDATE files SET duplicate_state = 'POTENTIAL_DUPLICATE'
WHERE (fast_hash, file_size_bytes) IN (
    SELECT fast_hash, file_size_bytes FROM files
    WHERE fast_hash IS NOT NULL
    GROUP BY fast_hash, file_size_bytes
    HAVING COUNT(*) > 1
)
AND duplicate_state = 'UNCHECKED';
```

**Markiere UNIQUE:**
```sql
UPDATE files SET duplicate_state = 'UNIQUE'
WHERE id NOT IN (
    SELECT id FROM files
    WHERE (fast_hash, file_size_bytes) IN (
        SELECT fast_hash, file_size_bytes FROM files
        GROUP BY fast_hash, file_size_bytes
        HAVING COUNT(*) > 1
    )
)
AND duplicate_state = 'UNCHECKED';
```

### 7.2 Full-Hash Berechnung (Stufe 3)
Schreibe `src/main/services/fullHash.ts`:

```typescript
import { createHash } from 'node:crypto';
import * as fs from 'node:fs';

export async function computeFullHash(filePath: string): Promise<string> {
  return new Promise((resolve, reject) => {
    const hash = createHash('sha256');  // oder BLAKE3 wenn verfuegbar
    const stream = fs.createReadStream(filePath);
    stream.on('data', (chunk) => hash.update(chunk));
    stream.on('end', () => resolve(hash.digest('hex')));
    stream.on('error', reject);
  });
}

export async function computeFullHashSync(filePath: string): Promise<string> {
  // Alternative: Sync Streaming mit festem Buffer fuer groessere Kontrolle
  const hash = createHash('sha256');
  const fd = fs.openSync(filePath, 'r');
  const buffer = Buffer.alloc(65536); // 64KB Chunks
  let bytesRead = 0;

  try {
    while ((bytesRead = fs.readSync(fd, buffer, 0, buffer.length, null)) > 0) {
      hash.update(buffer.subarray(0, bytesRead));
    }
  } finally {
    fs.closeSync(fd);
  }

  return hash.digest('hex');
}
```

### 7.3 Deep Scan Engine
Schreibe `src/main/services/deepScanEngine.ts`:

```typescript
import { computeFullHash } from './fullHash';
import { DatabaseManager } from '../db/database';

export class DeepScanEngine {
  private queue: DeepScanJob[] = [];
  private running = false;
  private currentJob: DeepScanJob | null = null;

  // Fuelle Queue mit POTENTIAL_DUPLICATE Kandidaten
  async enqueueCandidates(): Promise<number> {
    const db = DatabaseManager.getInstance().getConnection();
    const rows = await this.findCandidatesWithoutFullHash(db);
    for (const row of rows) {
      this.queue.push({
        fileId: row.id,
        filePath: row.file_path,
        mediaId: row.media_id,
      });
    }
    return rows.length;
  }

  // Verifiziere ein spezifisches Paar (fuer User Aktion)
  async verifyPair(fileIdA: string, fileIdB: string): Promise<boolean> {
    const db = DatabaseManager.getInstance();
    const fileA = await db.getFile(fileIdA);
    const fileB = await db.getFile(fileIdB);
    if (!fileA || !fileB) return false;

    const hashA = await computeFullHash(fileA.file_path);
    const hashB = await computeFullHash(fileB.file_path);
    const match = hashA === hashB;

    // Update DB
    await db.exec(`UPDATE files SET full_hash = ?, hash_type = 'SHA256',
                    hash_verified_at = NOW(), duplicate_state = ?
                    WHERE id IN (?, ?)`, [hashA, match ? 'CONFIRMED_DUPLICATE' : 'UNIQUE', fileIdA, fileIdB]);
    return match;
  }
}
```

### 7.4 Full-Hash Matching SQL
```sql
-- Finde BESTAETIGTE Duplikate (vollstaendiger Hash stimmt ueberein)
SELECT full_hash, COUNT(*) as cnt,
       array_agg(id) as file_ids,
       SUM(file_size_bytes) as wasted_bytes
FROM files
WHERE full_hash IS NOT NULL AND duplicate_state = 'CONFIRMED_DUPLICATE'
GROUP BY full_hash
HAVING COUNT(*) > 1
ORDER BY wasted_bytes DESC;
```

### 7.5 UI Status Unterscheidung
Das System unterscheidet fuer die UI Anzeige:

| Status | Bedeutung | DB Wert | UI Farbe |
|--------|-----------|---------|----------|
| UNCHECKED | Noch nicht geprueft | `UNCHECKED` | Grau |
| UNIQUE | Kein Duplikat bekannt | `UNIQUE` | Gruen |
| POTENTIAL | Fast-Hash Match, kein Full-Hash | `POTENTIAL_DUPLICATE` | Gelb |
| CONFIRMED | Full-Hash bestaetigt | `CONFIRMED_DUPLICATE` | Rot |

### 7.6 Background Deep Scan Queue
- Nach dem Initial-Scan: Automatisch `enqueueCandidates()` aufrufen
- Verarbeite die Queue im Idle (wenn kein anderer Task I/O beansprucht)
- Prioritaet: Niedriger als User Aktionen (P2 im Orchestrator Modell)
- Pro Takt: Max 50 Dateien deep-scannen, dann 500ms pausieren (Fairness)

## Akzeptanzkriterien
- [ ] SQL identifiziert Duplikat Kandidaten via Groesse + Fast-Hash Korrekt
- [ ] `computeFullHash()` erzeugt korrekten SHA256/XXH3 Hash
- [ ] `verifyPair()` berechnet Full-Hash und aktualisiert DB Status
- [ ] POTENTIAL_DUPLICATE Status wird korrekt gesetzt
- [ ] Nach Deep Scan werden Kandidaten zu CONFIRMED_DUPLICATE oder UNIQUE
- [ ] Background Queue laeuft im Idle und stoppt bei User Aktivitaet
- [ ] UI Farben (Grau/Gruen/Gelb/Rot) sind korrekt zugeordnet

## Abgrenzung
- UI Komponente fuer Duplikat Dashboard kommt in Task 15
- Duplikat Exclusions (False-Positive) kommt in Task 9
- Full-Hash wird NUR bei Kandidaten berechnet, nicht bei allen Dateien
# Task 4: Scan Engine Worker Thread mit Chunk Checkpointing

## Ziel
Implementiere die Scan Engine als dedizierten Node.js Worker Thread mit chunkbasiertem Checkpointing. Der Worker traversiert rekursiv das Dateisystem, produziert Batches und kommuniziert via `postMessage` mit dem Main Process.

## Schritte

### 4.1 Worker Thread Einstieg
Schreibe `src/main/services/scanWorker.ts`:

- Erzeuge `node:worker_threads.Worker` mit der Datei `src/main/services/scanWorker.js`
- Der Worker wird via `new Worker(join(__dirname, 'scanWorker.js'))` gestartet
- API: `startScan(mediaId, rootPath)`, `cancelScan()`
- Empfaengt `BATCH_ACK` vom Main Process, sendet `BATCH_DATA`, `SCAN_COMPLETE`, `SCAN_ERROR`, `SCAN_PROGRESS`

### 4.2 Der Worker Code
Schreibe `src/main/services/scanWorker.js` (plain JS wegen Worker Einschraenkungen, oder TypeScript via ts-loader/ESBuild):

```typescript
// src/main/services/scanWorker.ts (compiled)
import { parentPort } from 'node:worker_threads';
import * as fs from 'node:fs/promises';
import * as path from 'node:path';

class ScanWorker {
  private buffer: FileEntry[] = [];
  private readonly BATCH_SIZE = 1000;
  private isWaitingForAck = false;
  private cancelled = false;
  private scannedCount = 0;
  private checkpointPath: string | null = null;
  private readonly mediaId: number;

  constructor(mediaId: number) { this.mediaId = mediaId; }

  async traverseDirectory(dirPath: string): Promise<void> {
    if (this.cancelled) return;

    const entries = await fs.readdir(dirPath, { withFileTypes: true });
    for (const entry of entries) {
      if (this.cancelled) return;
      // Symlink Schutz
      if (entry.isSymbolicLink()) continue;

      const fullPath = path.join(dirPath, entry.name);
      if (entry.isDirectory()) {
        await this.traverseDirectory(fullPath);
      } else if (entry.isFile()) {
        await this.processFile(fullPath, entry);
      }
    }
  }

  private async processFile(fullPath: string, entry: fs.Dirent): Promise<void> {
    const stat = await fs.stat(fullPath);
    const fileEntry: FileEntry = {
      id: crypto.randomUUID(),
      mediaId: this.mediaId,
      fileName: entry.name,
      filePath: fullPath,
      parentPath: path.dirname(fullPath),
      fileSizeBytes: stat.size,
      mtime: stat.mtime.toISOString(),
      createdAt: stat.birthtime.toISOString(),
      isDirectory: entry.isDirectory(),
      deviceId: `${stat.dev}`,
      inode: stat.ino,
    };
    this.buffer.push(fileEntry);
    this.scannedCount++;

    // Checkpoint setzen fuer Resume
    this.checkpointPath = fullPath;

    if (this.buffer.length >= this.BATCH_SIZE) {
      await this.flush();
    }

    // Progress alle 100 Dateien melden
    if (this.scannedCount % 100 === 0) {
      parentPort?.postMessage({
        type: 'SCAN_PROGRESS',
        scannedCount: this.scannedCount,
        currentFile: fullPath,
      });
    }
  }

  private async flush(): Promise<void> {
    while (this.isWaitingForAck) {
      await new Promise(r => setTimeout(r, 5));
    }
    this.isWaitingForAck = true;
    parentPort?.postMessage({
      type: 'BATCH_DATA',
      payload: [...this.buffer],
      checkpointPath: this.checkpointPath,
    });
    this.buffer = [];
  }
}
```

### 4.3 Main Process Controller
Schreibe `src/main/services/scanController.ts`:

- `ScanController` Klasse die den Worker managed
- `startScan(mediaId, rootPath, profile)`: Erzeugt Worker, lauscht auf Messages
- `cancelScan()`: Sendet `CANCEL` Signal an Worker
- Haelt Referenz auf `DatabaseManager` fuer Batch Ingestion
- Implementiert `handleBatchData(batch)`: Fuehrt `BEGIN TRANSACTION`, INSERT Batches, `COMMIT`, sendet `BATCH_ACK`
- Resume Support: Prueft ob `checkpoint_path` in DB existiert, startet dort

### 4.4 Chunk Checkpointing in der DB
Beim Initialisieren des Scans:

```sql
INSERT INTO media (media_id, volume_label, scanned_at, lifecycle_status)
VALUES (?, ?, NOW(), 'SCANNING');
```

Bei jedem erfolgreichen Batch Commit:

```sql
UPDATE media SET scanned_at = NOW(), total_size_bytes = total_size_bytes + ?
WHERE media_id = ?;
```

Wenn Scan abbricht bleibt der letzte checkpointierte Chunk erhalten.

### 4.5 Resume Mechanismus
- Vor dem Start: Pruefe ob `media` Eintrag mit `lifecycle_status = 'SCANNING'` existiert
- Frage User: "Scan fortsetzen ab Checkpoint oder neu starten?"
- Bei Fortsetzen: Starte Worker mit `checkpointPath` als Startverzeichnis
- Ueberspringe bereits indizierte Pfade via EXISTS SQL Subquery

## Akzeptanzkriterien
- [ ] Worker Thread traversiert rekursiv ein Verzeichnis
- [ ] Batches von 1000 Dateien werden korrekt an Main Process gesendet
- [ ] Nach jedem Batch Commit wird `BATCH_ACK` zurueckgesendet
- [ ] Bei `cancelScan()` hoert der Worker auf zu scannen
- [ ] Nach einem Abbruch ist der letzte erfolgreiche Chunk in der DB erhalten
- [ ] Resume Scan ueberspringt bereits indizierte Dateien
- [ ] Symlinks werden uebersprungen (keine Endlosschleifen)

## Abgrenzung
- Fast-Hash Berechnung kommt in Task 5
- I/O Throttling kommt in Task 6
- Kein UI fuer Scan Fortschritt (nur Worker sendet Events)
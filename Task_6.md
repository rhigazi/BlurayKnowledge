# Task 6: Backpressure Flow Control & Progress Reporting

## Ziel
Vervollstaendige die ACK-basierte Flow Control zwischen Worker Thread und Main Process. Implementiere das progressiv gedrosselte Progress Event System fuer die UI Aktualisierung.

## Schritte

### 6.1 ACK Signal Schleife (Main Process)
Schreibe `src/main/services/flowController.ts`:

```typescript
export class FlowController {
  private pendingAck = false;
  private readonly maxPendingBatches: number;

  constructor(maxPending = 1) { this.maxPendingBatches = maxPending; }

  get canSendBatch(): boolean {
    return !this.pendingAck;  // Nur einen Batch in Flight erlauben
  }

  markBatchSent(): void { this.pendingAck = true; }

  markBatchComplete(): void { this.pendingAck = false; }

  async waitForSlot(): Promise<void> {
    while (!this.canSendBatch) {
      await new Promise(r => setTimeout(r, 5));
    }
  }
}
```

**Verbindung mit Scan Ingestion Service:**
- `ScanIngestionService` erhaelt `FlowController`
- Bei `BATCH_DATA` Empfang: `markBatchSent()` -> Daten in DuckDB schreiben -> `markBatchComplete()` -> `BATCH_ACK` senden
- Haelt den Worker am Laufen waehrend der DB Write laeuft

### 6.2 Worker Side ACK Handling
Erweitere `ScanWorker` um:
- `parentPort.on('message', (msg) => { if (msg.type === 'BATCH_ACK') { this.isWaitingForAck = false; } })`
- Extra `handleCancel()` Methode: Setzt `cancelled = true`, stoppt nach aktuellem Batch
- Timeout Mechanismus: Wenn ACK > 30 Sekunden ausbleibt, logge Warnung und setze `isWaitingForAck = false`

### 6.3 Progress Throttler
Schreibe `src/main/services/progressThrottler.ts`:

```typescript
export class ProgressThrottler {
  private lastEmit = 0;
  private readonly minIntervalMs: number;

  constructor(minIntervalMs = 100) {  // max 10 FPS
    this.minIntervalMs = minIntervalMs;
  }

  shouldEmit(): boolean {
    const now = Date.now();
    if (now - this.lastEmit >= this.minIntervalMs) {
      this.lastEmit = now;
      return true;
    }
    return false;
  }
}
```

### 6.4 Progress Event Aggregation
Erweitere `ScanController`:

```typescript
class ScanController {
  private progressThrottler = new ProgressThrottler(100); // 10 FPS

  private setupWorkerEvents(worker: Worker): void {
    worker.on('message', (msg) => {
      switch (msg.type) {
        case 'BATCH_DATA':
          this.handleBatchData(msg.payload, msg.checkpointPath);
          break;
        case 'SCAN_PROGRESS':
          if (this.progressThrottler.shouldEmit()) {
            this.sendProgressToUI({
              scannedCount: msg.scannedCount,
              currentFile: msg.currentFile,
              speed: this.calculateSpeed(),
              estimatedRemaining: this.estimateRemaining(),
            });
          }
          break;
        case 'SCAN_ERROR':
          this.handleScanError(msg.error);
          break;
        case 'SCAN_COMPLETE':
          this.finalizeScan();
          break;
      }
    });
  }
}
```

### 6.5 Speed & ETA Berechnung
- Tracke `scannedCount` und `elapsedMs` ueber ein gleitendes Fenster (letzte 10 Sekunden)
- `calculateSpeed()`: Dateien pro Sekunde im Fenster
- `estimateRemaining()`: Wenn totalFileCount bekannt ist, `(total - scanned) / speed`

### 6.6 Progress an UI senden
- Via IPC Kanal `scan:progress` an den Renderer
- `webContents.send('scan:progress', { scannedCount, currentFile, speed, eta, profile })`
- UI empfängt via `window.api.onScanProgress(callback)`

### 6.7 Integritaet: Transaktionssicherheit
- Jeder `BATCH_DATA` wird in einer eigenen DuckDB Transaktion verarbeitet:
  ```sql
  BEGIN TRANSACTION;
  INSERT INTO files (...) VALUES (...);  -- fuer jede Datei im Batch
  COMMIT;
  ```
- Nur nach erfolgreichem COMMIT wird `BATCH_ACK` gesendet
- Bei Fehler: `ROLLBACK`, logge Fehler, sende `SCAN_ERROR` an UI
- Der Worker wartet nicht ewig: Timeout nach 10 Sekunden -> Abbruch

### 6.8 Speicher Monitor
- Bevor `BATCH_DATA` gesendet wird: Pruefe `process.memoryUsage().heapUsed`
- Wenn > 500 MB: Setze BATCH_SIZE auf 500 (dynamische Reduktion)
- Wenn > 800 MB: Sende `MEMORY_WARNING` an UI, reduziere BATCH_SIZE auf 200

## Akzeptanzkriterien
- [ ] Worker sendet `BATCH_DATA` und wartet auf `BATCH_ACK` vor naechstem Batch
- [ ] Main Process schreibt Batch in DuckDB und sendet `BATCH_ACK`
- [ ] Bei DuckDB Fehler wird `ROLLBACK` ausgefuehrt und `SCAN_ERROR` gesendet
- [ ] Progress erreicht UI mit max 10 FPS
- [ ] Speed und ETA werden korrekt berechnet
- [ ] Speicherverbrauch bleibt unter 500 MB bei schnellen Scans
- [ ] Bei 30s Timeout auf ACK wird der Scan abgebrochen (kein Deadlock)
- [ ] RAM Spitze bleibt bei <= 1 Batch im Flight (kein unbounded Queuing)

## Abgrenzung
- UI Komponente fuer Progress Display kommt in Task 10
- Resume/Retry Logik kommt spaeter (Task 13)
- Auto-Pause bei OS Activity kommt als separater Task
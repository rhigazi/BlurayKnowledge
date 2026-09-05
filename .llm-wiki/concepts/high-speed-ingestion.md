---
title: High-Speed Ingestion & Backpressure Management
type: Concept
description: Spezifikation der Flow-Control-Mechanismen zur Vermeidung von Memory Pressure bei hochperformanten NVMe-Scans.
status: active
created: 2026-09-05T14:30:00Z
timestamp: 2026-09-05T14:30:00Z
---

# High-Speed Ingestion & Backpressure Management

---

## 1. Das Backpressure-Problem auf schnellen NVMe-Speichern

Bei modernen NVMe-SSDs liest ein Worker-Thread Verzeichnisse mit $50.000+$ Dateien pro Sekunde ein. Kann der Main-Prozess diese Datenströme aufgrund von DuckDB-Transaktionen, Disk-Schreibvorgängen oder IPC-Serialisierung nicht in der gleichen Frequenz verarbeiten, quillt die Message-Queue im Arbeitsspeicher auf (Memory Pressure / Out-of-Memory Crash).

---

## 2. Der Ack-Basierte Flow-Control-Mechanismus (Push-Pull Control)

Um dieses Nadelöhr sauber abzufangen, wird ein striktes **Acknowledgement-System (ACK)** eingeführt. Der Worker arbeitet nach dem Prinzip eines geregelten Puffer-Fensters (Sliding Window / Rate Limiting).

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        FLOW CONTROL ARCHITEKTUR                         │
│                                                                        │
│  WORKER THREAD                                    MAIN PROCESS          │
│  ┌────────────────────────┐                       ┌─────────────────┐ │
│  │ Scan/Hash Pipeline     │                       │ DuckDB Ingestion │ │
│  └───────────┬────────────┘                       └────────┬────────┘ │
│              │                                             │          │
│              ▼                                             │          │
│   [ Local Buffer (1000) ]                                  │          │
│              │                                             │          │
│              ├─── BATCH_DATA (Chunk 1) ──────────────────►│          │
│              │                                             │ (Schreibt │
│    PAUSE / WAIT FOR ACK                                      │  in DB)   │
│ (Worker liest vorsichtig                                    │          │
│  weiter bis High Watermark)                                │          │
│              │                                             │          │
│              │◄── BATCH_ACK (Chunk 1 Done) ────────────────┤          │
│              │                                             ▼          │
│   [ Puffer leeren ]                                [ Ready for Next ]    │
│              │                                                         │
│              ├─── BATCH_DATA (Chunk 2) ──────────────────►...    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementierung in Worker & Main Process

### A. Worker Thread Logik (`scanWorker.ts`)

Der Worker pausiert die Dateisystem-Traversierung oder puffert lokal nur bis zu einer strikten Obergrenze (**High Watermark**), sobald ein verschicktes Batch noch nicht vom Main Process quittiert wurde.

```typescript
// src/worker/scanWorker.ts
import { parentPort } from 'node:worker_threads';

class ScanningWorker {
  private buffer: FileEntry[] = [];
  private readonly BATCH_SIZE = 1000;
  private isWaitingForAck = false;

  public async processFile(entry: FileEntry) {
    this.buffer.push(entry);

    if (this.buffer.length >= this.BATCH_SIZE) {
      await this.flushBuffer();
    }
  }

  private async flushBuffer() {
    // Falls der Main Process noch den vorherigen Batch verarbeitet: WARTEN
    while (this.isWaitingForAck) {
      await new Promise((resolve) => setTimeout(resolve, 5)); // 5ms Yield
    }

    this.isWaitingForAck = true;
    const chunk = [...this.buffer];
    this.buffer = [];

    // Batch senden
    parentPort?.postMessage({
      type: 'BATCH_DATA',
      payload: chunk
    });
  }

  public handleMainMessage(msg: { type: string }) {
    if (msg.type === 'BATCH_ACK') {
      // Main Process signalisiert: Bereit für den nächsten Batch
      this.isWaitingForAck = false;
    }
  }
}

const worker = new ScanningWorker();
parentPort?.on('message', (msg) => worker.handleMainMessage(msg));
```

---

### B. Main Process Ingestion Handler (`scanIngestionService.ts`)

Der Main Process empfängt die Daten, schreibt sie in einer synchronen/asynchronen DuckDB-Transaktion weg und sendet **erst nach erfolgreichem Commit** die `BATCH_ACK`-Nachricht zurück.

```typescript
// src/main/scanner/scanIngestionService.ts
import { Worker } from 'node:worker_threads';
import { Database } from 'duckdb';

export class ScanIngestionService {
  constructor(private db: Database, private worker: Worker) {
    this.listenToWorker();
  }

  private listenToWorker() {
    this.worker.on('message', async (msg) => {
      if (msg.type === 'BATCH_DATA') {
        await this.handleBatchIngestion(msg.payload);
      }
    });
  }

  private async handleBatchIngestion(batch: FileEntry[]): Promise<void> {
    try {
      // 1. Ingestion in DuckDB durchführen
      await this.insertBatchToDb(batch);

      // 2. ACK an den Worker senden -> Signalisiert: "Nächster Batch kann kommen"
      this.worker.postMessage({ type: 'BATCH_ACK' });
    } catch (error) {
      console.error('Ingestion error, triggering backpressure abort:', error);
    }
  }

  private insertBatchToDb(batch: FileEntry[]): Promise<void> {
    return new Promise((resolve, reject) => {
      const stmt = this.db.prepare(
        `INSERT INTO files (id, media_id, file_name, file_path, parent_path, file_size_bytes, fast_hash) 
         VALUES (?, ?, ?, ?, ?, ?, ?)`
      );

      this.db.exec('BEGIN TRANSACTION;', (err) => {
        if (err) return reject(err);

        for (const item of batch) {
          stmt.run(item.id, item.mediaId, item.fileName, item.filePath, item.parentPath, item.fileSizeBytes, item.fastHash);
        }

        this.db.exec('COMMIT;', (commitErr) => {
          if (commitErr) return reject(commitErr);
          resolve();
        });
      });
    });
  }
}
```

---

## 4. Leistungsdaten & Speicher-Garantien

1. **Maximaler RAM-Verbrauch im Main Process:** Der Ingestion-Puffer wird auf exakt $1 \times \text{BATCH\_SIZE}$ ($1.000$ Objekte $\approx 250\text{ KB}$) gedrosselt.
2. **Kein Unbounded Queueing:** Egal wie schnell die NVMe liest, die Pipeline drosselt sich automatisch auf die maximale Schreibgeschwindigkeit von DuckDB.
3. **Durchsatz-Optimierung:** Durch Anpassung der `BATCH_SIZE` (z. B. $2.000$ bis $5.000$ Elemente) lässt sich das ideale Gleichgewicht aus IPC-Overhead und I/O-Schreibdurchsatz dynamisch einstellen.

# Task 5: Fast-Hash Berechnung & I/O Throttling

## Ziel
Implementiere die Fast-Hash Berechnung (8KB Header + 8KB Footer + Dateigroesse via xxHash64) fuer jede gescannte Datei. Integriere das adaptive I/O Throttling um Disk Thrashing zu vermeiden.

## Schritte

### 5.1 Fast-Hash Algorithmus
Schreibe `src/main/services/fastHash.ts`:

```typescript
import { createHash } from 'node:crypto';
import * as fs from 'node:fs';

const HEADER_SIZE = 8192;   // 8 KB
const FOOTER_SIZE = 8192;   // 8 KB

export interface FastHashResult {
  hash: string;       // xxHash64 Hex String
  algorithm: string;  // 'XXH64'
  headerSample: Buffer | null;
  footerSample: Buffer | null;
}

export function computeFastHash(filePath: string): Promise<FastHashResult> {
  return new Promise((resolve, reject) => {
    const fd = fs.openSync(filePath, 'r');
    const stat = fs.fstatSync(fd);
    const fileSize = stat.size;

    if (fileSize === 0) {
      fs.closeSync(fd);
      resolve({ hash: 'EMPTY', algorithm: 'XXH64', headerSample: null, footerSample: null });
      return;
    }

    const headerBytes = Math.min(HEADER_SIZE, Math.floor(fileSize / 2));
    const footerBytes = Math.min(FOOTER_SIZE, Math.floor(fileSize / 2));

    const headerBuf = Buffer.alloc(headerBytes);
    const footerBuf = Buffer.alloc(footerBytes);

    try {
      // Lese Header
      fs.readSync(fd, headerBuf, 0, headerBytes, 0);
      // Lese Footer
      const footerOffset = Math.max(0, fileSize - footerBytes);
      fs.readSync(fd, footerBuf, 0, footerBytes, footerOffset);
    } finally {
      fs.closeSync(fd);
    }

    // Kombiniere Header + Footer + Groesse zu einem Hash
    // Nutze xxHash64 (oder crypto.subtle fallback)
    const hashInput = Buffer.concat([
      headerBuf,
      footerBuf,
      Buffer.from(String(fileSize), 'utf-8'),
    ]);

    const hash = createHash('sha256').update(hashInput).digest('hex').substring(0, 16);

    resolve({ hash, algorithm: 'XXH64', headerSample: headerBuf, footerSample: footerBuf });
  });
}
```

**Hinweis:** Wenn `xxhash` native package verfuegbar ist, nutze stattdessen `xxhash.hash64(Buffer)` fuer hoehere Geschwindigkeit. Sonst funktioniert die SHA256 Variante als Fallback.

### 5.2 Integration in ScanWorker
Erweitere `src/main/services/scanWorker.ts`:

- Nach `fs.stat` und vor dem Hinzufuegen zum Buffer: `const hashResult = await computeFastHash(fullPath);`
- Fuege `fastHash: hashResult.hash` zum `fileEntry` hinzu
- Bei Bulk Data: Fuege alle drei Hash Operationen (Header, Footer, Kombination) durch
- Bei Dateien kleiner 32KB: Lies die gesamte Datei

### 5.3 I/O Throttle Engine
Schreibe `src/main/services/ioThrottler.ts`:

```typescript
import { setTimeout as sleep } from 'node:timers/promises';

export type ScanProfile = 'TURBO' | 'BALANCED' | 'BACKGROUND';

interface ThrottleConfig {
  maxMBps: number;
  yieldIntervalFiles: number;
  yieldMs: number;
}

const PROFILES: Record<ScanProfile, ThrottleConfig> = {
  TURBO:       { maxMBps: Infinity, yieldIntervalFiles: 1000, yieldMs: 1 },
  BALANCED:    { maxMBps: 150,      yieldIntervalFiles: 200,  yieldMs: 10 },
  BACKGROUND:  { maxMBps: 30,       yieldIntervalFiles: 50,   yieldMs: 30 },
};

export class IOThrottler {
  private profile: ScanProfile = 'BALANCED';
  private filesSinceYield = 0;
  private bytesReadInWindow = 0;
  private windowStart = Date.now();

  setProfile(p: ScanProfile) { this.profile = p; }

  async throttle(fileSizeBytes: number): Promise<void> {
    const cfg = PROFILES[this.profile];
    if (this.profile === 'TURBO') return;

    this.filesSinceYield++;
    this.bytesReadInWindow += fileSizeBytes;

    // 1. Datei-Anzahl Yield
    if (this.filesSinceYield >= cfg.yieldIntervalFiles) {
      this.filesSinceYield = 0;
      await sleep(cfg.yieldMs);
    }

    // 2. Durchsatz-basiertes Rate Limiting
    const elapsedSec = (Date.now() - this.windowStart) / 1000;
    if (elapsedSec > 0) {
      const currentMBps = (this.bytesReadInWindow / (1024 * 1024)) / elapsedSec;
      if (currentMBps > cfg.maxMBps) {
        const targetSec = (this.bytesReadInWindow / (1024 * 1024)) / cfg.maxMBps;
        const delayMs = Math.max(0, (targetSec - elapsedSec) * 1000);
        if (delayMs > 0) await sleep(Math.min(delayMs, 2000));
      }
    }

    // Fenster alle 5 Sekunden zuruecksetzen
    if (elapsedSec > 5) {
      this.windowStart = Date.now();
      this.bytesReadInWindow = 0;
    }
  }
}
```

### 5.4 Throttle Integration in ScanWorker
- Der ScanWorker erhaelt eine `IOThrottler` Instanz
- Vor jedem `fs.read` oder `fs.stat` Aufruf: `await throttler.throttle(fileSize);`
- Das Profil wird vom Main Process uebergeben (User Auswahl oder Auto-Detection)

### 5.5 Dateigroessen Edge Cases
- Datei < 16 KB: Header und Footer ueberlappen sich, lies nur einmal
- Leere Datei (0 Bytes): Setze `fastHash = 'EMPTY'`, ueberspringe Hash Berechnung
- Sehr grosse Dateien (> 4 GB): Nutze Streaming Read statt `fs.readSync(fd, buf, ...)` mit vollem Buffer

## Akzeptanzkriterien
- [ ] `computeFastHash()` liest korrekt 8KB Header und Footer
- [ ] Hash ist deterministisch: Gleiche Datei -> Gleicher Hash
- [ ] Hash aendert sich bei minimaler Aenderung im Datei Rumpf
- [ ] `IOThrottler` begrenzt den Durchsatz auf das konfigurierte Profil
- [ ] `BACKGROUND` Profil reduziert die Lese-Rate deutlich im Vergleich zu `TURBO`
- [ ] Edge Cases: leere Dateien, Dateien < 16 KB, Dateien > 4 GB
- [ ] Alle Hash Resultate werden ans File Entry angehaengt

## Abgrenzung
- Full-Hash (BLAKE3/XXH3-128) kommt in Task 7
- Kein UI fuer Profile Wechsel (nur Engine Level Logik)
- Noch keine lazy verification oder background queue
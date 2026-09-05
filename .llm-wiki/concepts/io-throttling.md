---
title: I/O-Throttling & Resource Governance Strategy
type: Concept
description: Strategie zur Vermeidung von Disk Thrashing und zur Sicherstellung einer stabilen Systemantwort während intensiver Scan-Vorgänge.
status: active
created: 2026-09-05T15:00:00Z
timestamp: 2026-09-05T15:00:00Z
---

# I/O-Throttling & Resource Governance Strategy

---

## 1. Das Problem des ungedrosselten I/O-Zugriffs

Wenn der Scan Worker ohne Durchsatzgrenzen Verzeichnisse traversiert oder Hashes berechnet, tritt das sogenannte **Disk Thrashing** auf. Bei mechanischen Festplatten (HDDs) führt das permanente Umherspringen der Lese-Köpfe zu extreme Latenzen für das Gesamtsystem; bei SSDs blockieren hunderte parallele Queue-Requests die Bus-Bandbreite. Das System wird unbedienbar.

---

## 2. Die Lösung: Adaptives & Einstellbares I/O-Throttling

Wir integrieren eine dynamische Drosselung auf **zwei Ebenen**:

1. **Concurrency Control (Thread Priority & Parallelität):** Reduktion von parallelen File-Descriptors.
2. **Read-Rate Limiting (Tokens/Delay Engine):** Erzwungene Mikro-Pausen zwischen E/O-Operationen basierend auf Systemauslastung und Nutzerprofil.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                      I/O THROTTLING CONTROL LOOP                       │
│                                                                        │
│  [ SCANNER WORKER PIPELINE ]                                           │
│          │                                                             │
│          ▼                                                             │
│  ┌───────────────────────────┐                                         │
│  │ File Read /              │                                         │
│  │ Directory Walk           │                                         │
│  └───────────┬───────────────┘                                         │
│              │                                                         │
│              ▼                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ I/O Throttle Engine                                              │  │
│  │  - Modus: LOW_PRIORITY / BALANCED / MAX_SPEED                    │  │
│  │  - Berechnet benötigten Delay (ms) basierend auf Dateigröße      │  │
│  │  - Erstellt `await sleep(calculatedDelay)`                       │  │
│  └───────┬──────────────────────────────────────────────────────────┘  │
│          │                                                         │
│          ▼                                                         │
│  Proceed with Read / Hash Calculation                                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementierung der Throttle Engine im Worker

Die Throttle-Engine berechnet dynamisch eine Verzögerung per Token-Bucket/Delay-Modell, bevor ein Read-Stream geöffnet oder eine Datei abgefragt wird.

```typescript
// src/worker/ioThrottler.ts
import { setTimeout as sleep } from 'node:timers/promises';

export type ScanProfile = 'TURBO' | 'BALANCED' | 'BACKGROUND';

interface ThrottleConfig {
  maxMBps: number;          // Maximale Durchsatzrate in MB/s
  yieldIntervalFiles: number; // Nach wie vielen Dateien eine erzwungene Pause eingelegt wird
  yieldMs: number;            // Dauer der erzwungenen Pause in ms
}

const PROFILES: Record<ScanProfile, ThrottleConfig> = {
  TURBO: { maxMBps: Infinity, yieldIntervalFiles: 1000, yieldMs: 1 },
  BALANCED: { maxMBps: 150, yieldIntervalFiles: 200, yieldMs: 10 },
  BACKGROUND: { maxMBps: 30, yieldIntervalFiles: 50, yieldMs: 30 },
};

export class IOThrottler {
  private currentProfile: ScanProfile = 'BALANCED';
  private processedFileCount = 0;
  private bytesReadInWindow = 0;
  private windowStart = Date.now();

  constructor(profile: ScanProfile = 'BALANCED') {
    this.currentProfile = profile;
  }

  public setProfile(profile: ScanProfile): void {
    this.currentProfile = profile;
  }

  /**
   * Wird vor jeder E/O-intensiven Operation (z.B. Hash-Berechnung oder File-Read) aufgerufen.
   */
  public async throttleRead(fileSizeBytes: number): Promise<void> {
    const config = PROFILES[this.currentProfile];
    
    if (this.currentProfile === 'TURBO') return;

    this.processedFileCount++;
    this.bytesReadInWindow += fileSizeBytes;

    // 1. Datei-Anzahl basierter Yield (Gibt dem OS-Scheduler Zeit für andere Apps)
    if (this.processedFileCount >= config.yieldIntervalFiles) {
      this.processedFileCount = 0;
      await sleep(config.yieldMs);
    }

    // 2. Durchsatz-basiertes Rate-Limiting (Bandbreiten-Deckel)
    const now = Date.now();
    const elapsedSeconds = (now - this.windowStart) / 1000;

    if (elapsedSeconds > 0) {
      const currentMBps = (this.bytesReadInWindow / (1024 * 1024)) / elapsedSeconds;
      
      if (currentMBps > config.maxMBps) {
        // Berechne nötige Wartezeit, um den Ziel-Durchsatz einzuhalten
        const targetTimeSec = (this.bytesReadInWindow / (1024 * 1024)) / config.maxMBps;
        const delayMs = (targetTimeSec - elapsedSeconds) * 1000;
        
        if (delayMs > 0) {
          await sleep(Math.min(delayMs, 2000)); // Max. 2s am Stück schlafen
        }
      }
    }

    // Zeitfenster alle 5 Sekunden zurücksetzen
    if (elapsedSeconds > 5) {
      this.windowStart = Date.now();
      this.bytesReadInWindow = 0;
    }
  }
}
```

---

## 4. UI-Steuerung & Auto-Detection Modus

Damit der Nutzer die volle Kontrolle hat (oder das System automatisch adaptiert), wird eine Steuerung im UI integriert:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        SCANNER LEISTUNGSPROFIL                         │
│                                                                        │
│  Scan-Geschwindigkeit:                                               │
│                                                                        │
│  ( ) Hintergrund (30 MB/s)  ──► Minimaler Einfluss, stört nie              │
│  (•) Ausgewogen    (150 MB/s) ──► Empfohlen für normales Arbeiten       │
│  ( ) Turbo         (Max Speed) ──► Volle Leistung (System wird träge) │
│                                                                        │
│  [x] Automatisch drosseln, wenn Nutzer aktiv am PC arbeitet            │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Auto-Pause bei User-Activity:** Wenn der Main Process Inaktivität/Aktivität der Maus oder Tastatur via OS-Events registriert, schaltet die Throttle-Engine automatisch zwischen `BACKGROUND` (Nutzer tippt/arbeitet) und `TURBO` (Nutzer ist abwesend) um.
2. **Low-Priority Process Flag:** Auf Betriebssystemebene wird dem Worker-Prozess/Thread eine niedrige E/O- und CPU-Priorität zugewiesen (`os.setPriority` / `nice` / `THREAD_MODE_BACKGROUND_BEGIN`).

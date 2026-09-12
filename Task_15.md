# Task 15: Lebenszyklus Automatismen & System Integration

## Ziel
Implementiere die automatischen Lebenszyklus Uebergaenge, die Auto-Pause bei OS Activity, und die System Integration (Tray Icon, Auto-Start, globale Shortcuts).

## Schritte

### 15.1 Automatische Veraltungserkennung
Schreibe `src/main/services/lifecycleScheduler.ts`:

```typescript
export class LifecycleScheduler {
  private checkInterval: Timer | null = null;

  start(): void {
    // Pruefe alle 6 Stunden auf veraltete Medien
    this.checkInterval = setInterval(() => this.checkStaleness(), 6 * 60 * 60 * 1000);
  }

  async checkStaleness(): Promise<void> {
    const db = DatabaseManager.getInstance();
    // Setze UNVERIFIED fuer schreibbare Medien aelter als 30 Tage
    await db.exec(`
      UPDATE media SET lifecycle_status = 'UNVERIFIED'
      WHERE lifecycle_status = 'ACTIVE'
        AND media_type IN ('USB-HDD', 'USB-Stick')
        AND scanned_at < NOW() - INTERVAL '30 days'
    `);
  }
}
```

### 15.2 Auto-Pause bei OS Activity
Schreibe `src/main/services/activityMonitor.ts`:

```typescript
import { powerMonitor } from 'electron';

export class ActivityMonitor {
  private lastActivity = Date.now();
  private readonly IDLE_TIMEOUT_MS = 60_000; // 1 Minute Inaktivitaet = Idle
  private callback: ((isActive: boolean) => void) | null = null;

  constructor() {
    powerMonitor.on('resume', () => this.onUserActive());
  }

  onUserActive(): void {
    this.lastActivity = Date.now();
    this.callback?.(true); // User ist aktiv
  }

  startMonitoring(callback: (isActive: boolean) => void): void {
    this.callback = callback;
    // Starte Check Intervall
    setInterval(() => {
      const isActive = (Date.now() - this.lastActivity) < this.IDLE_TIMEOUT_MS;
      callback(isActive);
    }, 10_000); // Alle 10 Sekunden checken
  }
}
```

**Integration mit I/O-Throttler:**
- Wenn User aktiv: Setze I/O Profil auf `BACKGROUND` (30 MB/s)
- Wenn User inaktiv fuer > 1 Minute: Setze auf `BALANCED` oder `TURBO`
- Bei Maus/Tastatur Event im Renderer: Sende `user:active` Signal an Main Process

### 15.3 Tray Icon Integration
Schreibe `src/main/services/trayManager.ts`:

- System Tray Icon erstellen (kleines Icon der App)
- Kontext Menu: `[Oeffnen]`, `[Scan starten]`, `[Trennen]`
- Click auf Tray: Fenster fokussieren/oeffnen
- Tooltip: Aktuelle Katalog Statistik

### 15.4 Globale Shortcuts
Schreibe `src/main/services/globalShortcuts.ts`:

```typescript
import { globalShortcut } from 'electron';

export function registerGlobalShortcuts(mainWindow: BrowserWindow): void {
  // Ctrl+F / Cmd+F: Suche fokussieren (wird im Renderer gehandhabt)
  // F5: Katalog aktualisieren
  // Ctrl+Shift+S: Screenshot / Schnellzugriff

  globalShortcut.register('F5', () => {
    mainWindow.webContents.send('sys:refresh');
  });
}
```

### 15.5 Scanning Status im Fenster Titel
- Waehrend eines Scans: Setze Fenster Titel auf `"BlurayKnowledge - Scanne (42.531 Dateien)"`
- Nach Scan: Zurueck zu `"BlurayKnowledge"`
- Via `mainWindow.setTitle()` oder `webContents.setWindowTitle()`

### 15.6 Auto-Start Option
- Einstellung: App beim Systemstart laden (optional)
- via `app.setLoginItemSettings({ openAtLogin: true })`
- Standard: Aus (User muss aktiv zustimmen)

### 15.7 System Statistik Aktualisierung
Erweitere `sysHandler.ts`:

```typescript
ipcMain.handle('sys:stats', async () => {
  const db = DatabaseManager.getInstance();
  const [mediaStats, fileStats, dupStats] = await Promise.all([
    db.query(`SELECT COUNT(*) as count FROM media`),
    db.query(`SELECT COUNT(*) as count, SUM(file_size_bytes) as totalSize FROM files WHERE is_directory = FALSE`),
    db.query(`SELECT COUNT(*) as count FROM files WHERE duplicate_state IN ('POTENTIAL_DUPLICATE', 'CONFIRMED_DUPLICATE')`),
  ]);
  return {
    mediaCount: mediaStats[0].count,
    fileCount: fileStats[0].count,
    totalSizeBytes: Number(fileStats[0].totalSize) || 0,
    duplicateCandidates: dupStats[0].count,
    lastScanDate: /* letztes scanned_at aus media */,
  };
});
```

### 15.8 Datenbank Backup Mechanismus
- Periodisches Backup der `catalog.duckdb` Datei (alle 24 Stunden)
- Backup Pfad: `userData/bluray-knowledge/backups/`
- Max 7 Backups behalten (rotierende Loeschung)
- Manuelles Backup via `[Backup erstellen]` in den Einstellungen

### 15.9 Fehlerbericht und Logging
- Logger Service: `src/main/services/logger.ts`
- Log Level: DEBUG, INFO, WARN, ERROR
- Log Datei: `userData/bluray-knowledge/logs/app.log`
- Rotation: Max 10 MB pro Datei, max 3 Dateien
- Bei schweren Fehlern: Zeige Dialog "App muss neu gestartet werden"

## Akzeptanzkriterien
- [ ] Veraltete Medien werden automatisch nach 30 Tagen markiert
- [ ] Auto-Pause schaltet auf BACKGROUND bei User Aktivitaet
- [ ] Bei Idle > 1 Minute wird auf TURBO oder BALANCED geschaltet
- [ ] Tray Icon zeigt App Status und Kontext Menu
- [ ] F5 aktualisiert die Ansicht
- [ ] Fenster Titel zeigt Scan Status waerend aktiven Scans
- [ ] Auto-Start kann in Einstellungen aktiviert werden
- [ ] System Statistik wird korrekt aus der DB aggregiert
- [ ] Datenbank Backups werden automatisch erstellt (max 7)
- [ ] Logger schreibt in Datei mit Rotation

## Abgrenzung
- Kein Multi-Worker Threading (Task 15 im Roadmap, optional)
- Keine Netzwerk Synchronisation (Offline First Prinzip)
- Auto-Pause verwendet einfaches Timeout Modell (kein OS Idle API)
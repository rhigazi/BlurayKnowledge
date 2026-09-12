# Task 17: Einstellungen, Export & Electron Packaging

## Ziel
Baue die Einstellungen Ansicht, implementiere Datenbank Export/Import Funktionen und konfiguriere den Production Build via electron-builder fuer Windows, macOS und Linux.

## Schritte

### 17.1 SettingsView Komponente
Schreibe `src/renderer/views/SettingsView.tsx`:

```tsx
// Kategorisierte Einstellungen:
// +------------------------------------------+
// | ⚙️ Einstellungen                          |
// |                                           |
// | Allgemein                                 |
// |   [ ] App beim Systemstart laden          |
// |   [ ] Minimize to tray statt schliessen   |
// |   Sprache: [Deutsch]                      |
// |                                           |
// | Scan Standard                            |
// |   Standard Profil: [Ausgewogen]            |
// |   [x] Deep Scan nach Scan automatisch     |
// |   [ ] Auto-Pause bei Aktivitaet           |
// |                                           |
// | Katalog                                   |
// |   Datenbank Pfad: /home/user/...          |
// |   [Backup jetzt erstellen]                |
// |   [Katalog exportieren]                   |
// |   [Katalog importieren]                   |
// |                                           |
// | Info                                      |
// |   Version: 0.1.0                          |
// |   Datenbank Groesse: 45.2 MB              |
// |   Anzahl Medien: 8                        |
// |   Anzahl Dateien: 42.531                  |
// |   [Datenbank zuruecksetzen] (Destruktiv)   |
// +------------------------------------------+
```

### 17.2 Export Funktion
Schreibe `src/main/services/exportService.ts`:

```typescript
export class ExportService {
  async exportToJSON(outputPath: string): Promise<void> {
    const db = DatabaseManager.getInstance();
    // Exportiere media, files, notes, duplicate_exclusions als JSON
    const media = await db.query('SELECT * FROM media');
    const files = await db.query('SELECT * FROM files');
    const notes = await db.query('SELECT * FROM notes');
    const exclusions = await db.query('SELECT * FROM duplicate_exclusions');

    const exportObj = {
      exportDate: new Date().toISOString(),
      version: '0.1.0',
      data: { media, files, notes, exclusions },
    };

    await fs.writeFile(outputPath, JSON.stringify(exportObj, null, 2), 'utf-8');
  }

  async importFromJSON(inputPath: string): Promise<ImportResult> {
    // Lese JSON Datei
    // Validiere Struktur und Version
    // Frage User: "Existierende Daten ersetzen oder erweitern?"
    // Option Replace: Loesche alle Tabellen, importiere neue Daten
    // Option Append: Fuege hinzu, ignoriere Duplikate (ON CONFLICT DO NOTHING)
    // Gebe Zusammenfassung: "8 Medien, 42.531 Dateien importiert"
  }
}
```

### 17.3 CSV Export (Optional)
- `exportToCSV(outputDir)`:
  - `media.csv`, `files.csv`, `notes.csv`
  - DuckDB `.csv` Export Funktion nutzen: `COPY (SELECT * FROM media) TO 'media.csv'`
  - Nutzer kann die CSV in Excel oeffnen

### 17.4 Electron Builder Konfiguration
Schreibe `electron-builder.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/electron-userland/electron-builder/master/packages/app-builder-lib/scheme.json",
  "appId": "com.blurayknowledge.app",
  "productName": "BlurayKnowledge",
  "directories": {
    "buildResources": "build",
    "output": "release"
  },
  "files": [
    "dist/**/*",
    "node_modules/duckdb/**/*",
    "node_modules/xxhash/**/*",
    "package.json"
  ],
  "extraResources": [
    {
      "from": "node_modules/duckdb/build",
      "to": "duckdb-native",
      "filter": ["**/*.node"]
    }
  ],
  "win": {
    "target": ["nsis"],
    "icon": "build/icon.ico"
  },
  "mac": {
    "target": ["dmg"],
    "icon": "build/icon.icns",
    "category": "public.app-category.utilities"
  },
  "linux": {
    "target": ["AppImage", "deb"],
    "icon": "build/icon.png",
    "category": "Utility"
  },
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true
  }
}
```

### 17.5 Native Binary Handling (DuckDB)
- DuckDB `.node` Binary muss im Package enthalten sein
- `electron-builder` extraResources kopiert die native Library
- In `main.ts`: Pruefe auf `process.resourcesPath` fuer den Package Pfad
- Fallback: `app.isPackaged ? path.join(process.resourcesPath, 'duckdb-native') : localPath`

### 17.6 Build Scripts in package.json
```json
{
  "scripts": {
    "dev": "concurrently \"vite\" \"tsc -p tsconfig.main.json --watch\" \"electron .\"",
    "build:renderer": "vite build",
    "build:main": "tsc -p tsconfig.main.json",
    "build": "npm run build:renderer && npm run build:main",
    "start": "electron dist/main/index.js",
    "package:win": "npm run build && electron-builder --win",
    "package:mac": "npm run build && electron-builder --mac",
    "package:linux": "npm run build && electron-builder --linux"
  }
}
```

### 17.7 Icons und Build Assets
- `build/icon.png` (512x512, App Icon)
- `build/icon.ico` (Windows, aus PNG generiert)
- `build/icon.icns` (macOS, aus PNG generiert)
- Alle Assets werden lokal erstellt (kein CDN)

### 17.8 Datenbank Reset (Destruktiv)
- Warnung: "Alle gespeicherten Daten gehen verloren. Medien muessen neu gescannt werden."
- Loesche `catalog.duckdb` Datei
- Erstelle neue Datenbank mit Schema
- Logge Aktion im Logger

### 17.9 Release Prozess
- `npm run package:linux` erzeugt `.AppImage` and `.deb`
- `npm run package:mac` erzeugt `.dmg`
- `npm run package:win` erzeugt `.exe` via NSIS Installer
- Output in `release/` Verzeichnis

## Akzeptanzkriterien
- [ ] Settings View zeigt alle Kategorien korrekt an
- [ ] Export nach JSON exportiert alle 4 Tabellen
- [ ] JSON Import funktioniert (Replace + Append Optionen)
- [ ] CSV Export produziert korrekte Dateien
- [ ] `electron-builder` build erzeugt Installer fuer die Zielplattform
- [ ] DuckDB native Binary wird korrekt ins Package eingebunden
- [ ] App startet aus dem Package heraus (getestet)
- [ ] Datenbank Reset loescht alle Daten und erstellt neues Schema
- [ ] Settings werden persistent gespeichert (via `electron-store` oder einfachem JSON in userData)

## Abgrenzung
- Keine Netzwerk Updates (Auto-Updater)
- Kein Multi-Sprache Support (nur Deutsch initial)
- Kein i18n Framework
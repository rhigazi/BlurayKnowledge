# Task 1: Projektgrundlage & Build Infrastruktur

## Ziel
Erzeuge das leere Projektgerüst für eine autarke Electron SPA mit React, TypeScript, Vite und DuckDB. Alle Assets werden lokal gebündelt, keine externen CDN Abhängigkeiten.

## Schritte

### 1.1 Projekt initialisieren
- Lege das Verzeichnis `dvd-catalog-app/` an.
- Führe `npm init` aus oder schreibe `package.json` mit folgenden Feldern:
  - `name`: `bluray-knowledge`
  - `version`: `0.1.0`
  - `type`: `module`
  - `main`: `dist/main/index.js`
  - `scripts`: `dev`, `build`, `start`, `package`
- Setze `"engines"` auf Node `>=20`.

### 1.2 Abhängigkeiten installieren
- **Runtime:** `electron`, `duckdb` (node binding), `react`, `react-dom`, `react-router-dom`
- **Dev:** `vite`, `@vitejs/plugin-react`, `electron-builder`, `typescript`, `@types/react`, `@types/react-dom`, `vitest`
- Achte auf DuckDB native Binary Bundling via `electron-builder` (Konfiguration in späterem Task).

### 1.3 Verzeichnisstruktur anlegen
Erzeuge das Skelett nach der Architekturvorgabe:

```
dvd-catalog-app/
├── src/
│   ├── main/           # Electron Main Process (Node.js Hintergrund)
│   │   ├── ipc/        # IPC Handler Kanal
│   │   ├── db/         # DuckDB Schicht
│   │   └── services/   # scan, hash, metadata services
│   ├── preload/        # contextBridge Schnittstelle
│   └── renderer/       # React SPA
│       ├── views/      # Seitenansichten
│       ├── components/ # Wiederverwendbare UI Komponenten
│       ├── hooks/      # Custom React Hooks
│       └── assets/     # Lokale Icons, Fonts, CSS
├── package.json
├── tsconfig.json
├── vite.config.ts
└── electron-builder.json
```

### 1.4 TypeScript Konfiguration
- `tsconfig.json` mit `strict: true`, Pfad-Aliase (`@main/*`, `@renderer/*`, `@shared/*`)
- Getrennte Configs für main und renderer falls nötig.

### 1.5 Vite Config für Renderer
- `vite.config.ts`: `@vitejs/plugin-react`, Build Output `dist/renderer/`
- Keine externen CDN Resourcen im HTML Template.

### 1.6 Electron Hauptprozess Grundgerüst
- `src/main/main.ts`: Fenster erstellen, `index.html` aus `dist/renderer/` laden
- `src/preload/index.ts`: `contextBridge.exposeInMainWorld('api', {...})` mit erster Grundstruktur
- Dev Server Unterstützung für Hot Reload im Renderer.

### 1.7 Renderer Grundgerüst
- `src/renderer/index.html`: Minimales HTML mit `#root`
- `src/renderer/main.tsx`: React Root mit `BrowserRouter` (oder `MemoryRouter` für Electron)
- `src/renderer/App.tsx`: Leeres Layout mit Platzhaltern

### 1.8 Build Scripts
- `dev`: Startet Vite Dev Server + Electron parallel
- `build`: Baut renderer via Vite, kompiliert main/preload via tsc
- `start`: Startet Electron mit gebauten Assets

## Akzeptanzkriterien
- [ ] `npm run dev` startet Electron Fenster mit React Oberfläche
- [ ] DuckDB kann im Main Process importiert werden (`.node` Binary gefunden)
- [ ] Preload Bridge ist definiert und im Renderer via `window.api` erreichbar
- [ ] TypeScript Compiler läuft ohne Fehler durch
- [ ] Keine externen Netzwerk Requests beim Laden der App

## Abgrenzung
- Noch keine Geschäftslogik (Scan, Hash, DB Schema)
- Noch keine UI Komponenten außer Platzhalter
- DuckDB wird nur importiert, noch nicht verwendet
---
title: Architektur: Autarke Electron SPA mit DuckDB
type: Concept
description: Spezifikation für eine 100% offlinefähige Single Page Application auf Electron-Basis zur Medienkatalogisierung.
status: active
created: 2026-09-05T06:48:51Z
timestamp: 2026-09-05T06:49:45Z
---

# Architektur: Autarke Electron SPA mit DuckDB

## Überblick
Diese Architektur beschreibt eine **100 % autarke Single Page Application (SPA)** auf Electron-Basis. Das Ziel ist eine Anwendung, die ohne jegliche Internetverbindung (Zero External Dependencies) funktioniert und alle Daten lokal verwaltet.

## Kernprinzipien

*   **Zero External Dependencies:** Keine Cloud-Dienste, keine externen APIs, keine CDN-Calls (auch keine Google Fonts).
*   **Single Page Application (SPA):** Das Frontend nutzt eine einzige `index.html` und steuert die Ansichten dynamisch über lokales Routing.
*   **Self-Contained Storage:** Verwendung von **DuckDB** als eingebettete, lokale Datenbank in einer einzigen Datei (`catalog.duckdb`).

## Systemarchitektur

Das System ist in drei Hauptschichten unterteilt:

1.  **Renderer Process (Frontend):** Eine React-basierte SPA, die komplett gebündelt (bundled) ausgeführt wird.
2.  **Preload Script (Bridge):** Nutzt `contextBridge`, um eine sichere Schnittstelle zwischen dem Frontend und dem Node.js-Hintergrund zu schaffen.
3.  **Main Process (Backend):** Verwaltet den Lebenszyklus der App, das Dateisystem-Scanning und die DuckDB-Datenbank-Engine.

## Projektstruktur (Empfehlung)

```text
dvd-catalog-app/
├── src/
│   ├── main/                       # Node.js Main Process (Hintergrund-Logik)
│   │   ├── ipc/                    # Kommunikationskanäle (scanHandler, dbHandler)
│   │   ├── db/                     # DuckDB Schicht (duckdb.ts, schema.ts)
│   │   └── services/               # diskScanner.ts, metadataExtractor.ts
│   ├── preload/                    # contextBridge-Schnittstelle
│   └── renderer/                   # React SPA (Views, Components, Hooks)
├── package.json
├── vite.config.ts
└── electron-builder.json
```

## Datenfluss (Beispiel: Suche)

1.  **UI:** Nutzer gibt Suchbegriff in der `CatalogView.tsx` ein.
2.  **Frontend:** Ruft `window.api.searchCatalog(query)` auf.
3.  **Preload:** Leitet den Aufruf via `ipcRenderer.invoke('db:search', query)` an den Main Process weiter.
4.  **Main:** Der `dbHandler` führt eine SQL-Abfrage gegen die lokale **DuckDB** aus.
5.  **Rückgabe:** Die Ergebnisse fließen über den IPC-Kanal zurück zur UI.

## Technische Anforderungen für die Autarkie

*   **Asset Bundling:** Icons (z.B. Lucide) und Fonts müssen im Build-Schritt lokal integriert werden.
*   **Binary Bundling:** Das native DuckDB-Kompilat (`duckdb.node`) muss via `electron-builder` in das Paket integriert werden.
*   **Metadata Extraction:** Nutzung von reinen JS/Node-Bibliotheken (z.B. `music-metadata`), um externe Abhängigkeiten zu vermeiden.

## timeline
- 2026-09-05T06:48:51Z: Architekturdefinition und Strukturfestlegung nach technischer Analyse.
EOF
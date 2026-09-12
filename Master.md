# BlurayKnowledge - Masterplan

> Autarke Electron SPA zur Katalogisierung von Offline Medien (DVD, BD-R, USB).
> 100% offline, DuckDB embedded, React Frontend, TypeScript.
> Abgeleitet aus den Wiki Seiten in `.llm-wiki/`.

---

## Bauphasen

| Phase | Tasks | Abhaengigkeit | Meilenstein |
|-------|-------|---------------|-------------|
| **A Fundament** | 1, 2, 3 | Keine | App startet mit leerem DuckDB Schema |
| **B Scan Engine** | 4, 5, 6 | Phase A | Worker scannt Verzeichnisse in DB |
| **C Verifikation** | 7 | Phase B | Duplikaterkennung arbeitet |
| **D Features** | 8, 9 | Phase A | Suche, Notizen, Lifecycle |
| **E Frontend Kern** | 10, 11 | Phase A + 8 | Hauptansicht mit Suche + Detail |
| **F Frontend Scan** | 12, 13, 14 | Phase B + E | Assistent, Medien, Duplikate |
| **G Integration** | 15 | Phase C + D + F | Automatismen, System |
| **H Qualitaet** | 16 | Alle vorher | Tests bestehen |
| **I Auslieferung** | 17, 18 | Phase H | Package und Abnahme |

---

## Task 1: Projektgrundlage & Build Infrastruktur (Mittel, 3 Subtasks)

### 1.1 Abhaengigkeiten und package.json
- `npm init`, fuelle `package.json` mit name, version, type, scripts, engines
- Installiere Runtime: `electron`, `duckdb`, `react`, `react-dom`, `react-router-dom`
- Installiere Dev: `vite`, `@vitejs/plugin-react`, `electron-builder`, `typescript`, `@types/react`, `@types/react-dom`, `vitest`
- **Output:** `package.json`, `node_modules/`

### 1.2 TypeScript + Vite + Verzeichnisstruktur
- `tsconfig.json` mit strict mode und Pfad Aliase (`@main`, `@renderer`, `@shared`)
- `vite.config.ts` mit React Plugin, Build Output `dist/renderer/`
- Lege Verzeichnisstruktur an: `src/main/`, `src/preload/`, `src/renderer/`, `tests/`
- **Output:** `tsconfig.json`, `vite.config.ts`, Ordnerstruktur

### 1.3 Electron Grundgeruest + Renderer Basis
- `src/main/main.ts`: Erstelle BrowserWindow, lade index.html
- `src/preload/index.ts`: contextBridge.exposeInMainWorld('api', {})
- `src/renderer/index.html`: Minimal HTML mit #root
- `src/renderer/main.tsx`: ReactDOM.createRoot
- `src/renderer/App.tsx`: Leeres Layout Platzhalter
- `build.sh` oder `concurrently` scripts fuer `dev` Mode
- **Output:** App startet mit Electron Fenster

---

## Task 2: DuckDB Schema & Datenbank Manager (Mittel, 3 Subtasks)

### 2.1 Schema SQL Definition
- `src/main/db/schema.sql`: Alle 4 Tabellen (`media`, `files`, `notes`, `duplicate_exclusions`)
- Alle Indizes (`idx_files_lookup`, `idx_files_media_path`, `idx_files_duplicate_state`, etc.)
- `files` Tabelle mit JSON Sparse Metadata Spalte
- **Output:** Vollstaendiges Schema SQL

### 2.2 DatabaseManager Klasse
- `src/main/db/database.ts`: Singleton, `initialize()` fuehrt schema.sql aus
- `getConnection()`, `close()`, `query()`, `exec()`
- `:memory:` Mode fuer Tests
- Datenbank Datei in `app.getPath('userData')`
- **Output:** DuckDB wird beim App Start initialisiert

### 2.3 Schema Versionierung + Prepared Statements
- Tabelle `_schema_version` mit Version Tracking
- Bei App Start: Pruefe Version, migriere nur bei Aenderung
- `src/main/db/statements.ts`: Prepared Statements Factory fuer alle Kernoperationen
- **Output:** Schema Migration + wiederverwendbare Queries

---

## Task 3: IPC Kommunikationsschicht & Preload Bridge (Mittel, 3 Subtasks)

### 3.1 Kanal Konstanten + Shared Types
- `src/shared/ipc-channels.ts`: Alle Kanal Namen als const (scan, db, note, media, dup, sys)
- `src/shared/api-types.ts`: `ElectronAPI`, `ScanProgress`, `FileEntry`, `MediaEntry`, `SearchResult`, etc.
- **Output:** Gemeinsames Typsystem fuer main + renderer

### 3.2 Preload Script
- `src/preload/index.ts`: `contextBridge.exposeInMainWorld('api', {...})`
- Jede Methode mapped auf `ipcRenderer.invoke(channel, ...args)`
- Progress Events via `ipcRenderer.on` (mit cleanup Funktion)
- **Output:** `window.api` ist im Renderer definiert

### 3.3 Main Process IPC Handler (Stubs)
- `src/main/ipc/index.ts`: `registerAllHandlers(db)` registriert alle Kanäle
- Domain Handler: `scanHandler.ts`, `dbHandler.ts`, `noteHandler.ts`, `mediaHandler.ts`, `sysHandler.ts`
- `progressEmitter.ts`: Event Bus fuer Progress Events
- Handler werfen `NOT_IMPLEMENTED` oder minimale Antwort
- **Output:** IPC Kanal Struktur steht, Renderer kann API aufrufen

---

## Task 4: Scan Engine Worker Thread mit Chunk Checkpointing (Gross, 4 Subtasks)

### 4.1 Worker Entry Point + Lebenszyklus
- `src/main/services/scanWorker.ts`: Export `startScan(mediaId, rootPath)`, `cancelScan()`, `SCAN_COMPLETE`
- Kommunikation via `parentPort.postMessage` / `parentPort.on('message')`
- Nachrichten Typen: `BATCH_DATA`, `BATCH_ACK`, `SCAN_PROGRESS`, `SCAN_ERROR`, `SCAN_COMPLETE`, `CANCEL`
- **Output:** Worker kann gestartet und gestoppt werden

### 4.2 Rekursive Directory Traversierung mit Batch Puffer
- `readdir` mit `withFileTypes`, `isSymbolicLink()` Schutz
- `fs.stat` fuer jede Datei: extrahiere `size`, `mtime`, `birthtime`, `dev`, `ino`
- Baue `FileEntry` Objekte, sammle in Buffer bis `BATCH_SIZE = 1000`
- Bei Buffer voll: `flush()` zu Main Process, warte auf ACK
- **Output:** Worker traversiert und produziert Batches

### 4.3 ScanController (Main Process)
- `src/main/services/scanController.ts`: `ScanController` Klasse
- `startScan()`: Erzeugt Worker, registriert Message Handler
- `handleBatchData()`: `BEGIN TRANSACTION`, `INSERT` aller Dateien, `COMMIT`, sende `BATCH_ACK`
- `cancelScan()`: Sende CANCEL, setze `media.lifecycle_status = 'CANCELLED'`
- **Output:** Daten fliessen vom Worker in DuckDB

### 4.4 Resume Mechanismus
- Pruefe bei Scan Start: `SELECT lifecycle_status FROM media` (ist `SCANNING`?)
- Wenn ja: Frage User "Fortsetzen oder neu starten?"
- Bei Fortsetzen: Starte Worker mit `checkpointPath`, ueberspringe via `WHERE file_path NOT IN (...)`
- Bei Abbruch: `media.total_size_bytes`, `media.scanned_at` bleiben vom letzten Chunk erhalten
- **Output:** Abgebrochene Scans koennen fortgesetzt werden

---

## Task 5: Fast-Hash Berechnung & I/O Throttling (Mittel, 3 Subtasks)

### 5.1 FastHash Computation
- `src/main/services/fastHash.ts`: `computeFastHash(filePath)` -> `{ hash, algorithm }`
- Lese 8KB Header + 8KB Footer + Dateigroesse, kombiniere in xxHash64 (oder SHA256 Fallback)
- Edge Cases: Leere Dateien (`'EMPTY'`), Dateien < 16KB (Header/Footer ueberlappend), > 4GB (Streaming)
- **Output:** Deterministicher Fast-Hash fuer jede Datei

### 5.2 IOThrottler
- `src/main/services/ioThrottler.ts`: `IOThrottler` Klasse mit Profilen `TURBO`, `BALANCED`, `BACKGROUND`
- Token Bucket Modell: Datei Anzahl Yield + Durchsatzbasiertes Rate Limiting
- Fenster alle 5 Sekunden zuruecksetzen
- **Output:** Drosselt I/O Zugriffe nach Profil

### 5.3 Integration in ScanWorker
- Rufe `computeFastHash()` nach `fs.stat` auf, haenge `fastHash` ans FileEntry
- Rufe `ioThrottler.throttle(fileSize)` vor jedem `fs.read` auf
- Profil wird vom Main Process ueber `startScan(mediaId, rootPath, profile)` uebergeben
- **Output:** ScanWorker hash und drosselt korrekt

---

## Task 6: Backpressure Flow Control & Progress Reporting (Mittel, 3 Subtasks)

### 6.1 FlowController (ACK Signal)
- `src/main/services/flowController.ts`: `FlowController` mit Slot Management
- `canSendBatch`, `markBatchSent()`, `markBatchComplete()`, `waitForSlot()`
- Nur 1 Batch in Flight (maxPendingBatches = 1)
- Timeout: Nach 30s ohne ACK -> Abbruch
- **Output:** Kein unbounded Queuing, Worker wartet auf ACK

### 6.2 ProgressThrottler + Speed/ETA
- `src/main/services/progressThrottler.ts`: Max 10 FPS (100ms Interval)
- Speed Berechnung: Gleitendes Fenster (letzte 10s) -> Dateien/sec
- ETA: `(totalScannedEstimate - scannedCount) / speed`
- **Output:** Geglaettete Progress Events an UI

### 6.3 Memory Monitor + Dynamische Batch Size
- Ueberwache `process.memoryUsage().heapUsed`
- Bei > 500MB: Reduziere BATCH_SIZE auf 500
- Bei > 800MB: Sende `MEMORY_WARNING`, reduziere auf 200
- Integration in `ScanController.handleBatchData()`
- **Output:** Kein Out-of-Memory bei schnellen NVMe Scans

---

## Task 7: 3-Stufen Duplikat Verifikation & Full-Hash Engine (Gross, 4 Subtasks)

### 7.1 Duplikat Kandidaten SQL (Stufen 1 + 2)
- `src/main/services/duplicateDetector.ts`:
  - Stufe 1: `GROUP BY file_size_bytes HAVING COUNT(*) > 1`
  - Stufe 2: `GROUP BY fast_hash, file_size_bytes HAVING COUNT(*) > 1`
  - Markiere `POTENTIAL_DUPLICATE` / `UNIQUE` via UPDATE Statements
- **Output:** SQL identifiziert Kandidaten korrekt

### 7.2 FullHash Computation (Stufe 3)
- `src/main/services/fullHash.ts`: `computeFullHash(filePath)` -> SHA256 Hex String
- Streaming Read mit 64KB Chunks, kein Overly Memory
- `verifyPair(fileIdA, fileIdB)`: Berechne beide, update DB auf `CONFIRMED_DUPLICATE` oder `UNIQUE`
- **Output:** Vollstaendiger Hash fuer Deep Scan

### 7.3 DeepScanEngine (Background Queue)
- `src/main/services/deepScanEngine.ts`: Queue fuer POTENTIAL_DUPLICATE Kandidaten
- `enqueueCandidates()`: Lade alle Kandidaten ohne Full-Hash
- Idle Processing: Max 50 Dateien pro Takt, dann 500ms Pause (Fairness)
- Prioritaet: P2 (niedriger als User Aktionen)
- **Output:** Hintergrund Deep Scan arbeitet automatisch

### 7.4 Status Management + DB Integration
- Vollstaendige Status Transition Chain:
  `UNCHECKED -> (scan) -> UNIQUE` oder `UNCHECKED -> (match) -> POTENTIAL -> (deep scan) -> CONFIRMED/UNIQUE`
- SQL: `SELECT wasted_bytes ... GROUP BY full_hash HAVING COUNT(*) > 1`
- UI Farben: Grau (`UNCHECKED`), Gruen (`UNIQUE`), Gelb (`POTENTIAL`), Rot (`CONFIRMED`)
- **Output:** Status wird bei jedem DB Zugriff korrekt angezeigt

---

## Task 8: Notes, Search & Notiz Vererbung (Gross, 4 Subtasks)

### 8.1 Notes CRUD
- `src/main/services/notesService.ts`:
  - `createNote(targetType, targetPath, targetFileId, targetMediaId, noteText, tags)`
  - `updateNote(noteId, noteText, tags)`, `deleteNote(noteId)`
  - `getNotesForFile(fileId)`: Alle Notizen + geerbte
  - `getNotesForFolder(folderPath, mediaId)`: Prefix Match
- **Output:** Notes koennen angelegt, gelesen, geaendert, geloescht werden

### 8.2 Note Inheritance SQL
- SQL: `COALESCE(fn.note_text, dn.note_text, mn.note_text) AS effective_note`
- Prioritaet: `FILE` (hoechste) > `FOLDER` > `MEDIA` (niedrigste)
- Ordner Vererbung via `f.parent_path STARTS_WITH dn.target_path`
- **Output:** Notizen werden hierarchisch korrekt vererbt

### 8.3 FTS Index + SearchService
- `src/main/db/fts-setup.sql`: `INSTALL fts; LOAD fts; CREATE VIEW files_search_view; PRAGMA create_fts_index`
- `src/main/services/searchService.ts`:
  - `search(query, filters)`: FTS Query fuer file_name, file_path, note_text
  - Fallback: `ILIKE` wenn FTS nicht verfuegbar
  - Filter: mediaId, extension, minSize, maxSize, duplicateState, dateFrom, dateTo
  - `searchByFolder(folderPath, mediaId)`: Ordner Navigation via parent_path
- **Output:** Suche unter 50ms ueber alle Felder

### 8.4 Fuzzy Search + Cache + IPC
- `levenshtein()` Fallback bei leeren FTS Ergebnissen (max 2 Toleranz)
- `searchCache.ts`: In-Memory Cache mit 5 Minuten TTL
- IPC Handler: `dbHandler.ts` -> `SearchService.search()`, `noteHandler.ts` -> `NotesService`
- **Output:** Gecachte Suche mit Tippfehler Toleranz

---

## Task 9: Media Lifecycle, Delta-Scan & Duplikat Exclusions (Mittel, 3 Subtasks)

### 9.1 MediaService + Lifecycle Transitions
- `src/main/services/mediaService.ts`: CRUD + `archiveMedia()`, `markMissing()`, `reactivate()`
- Automatische Uebergaenge: nach Scan -> `ACTIVE`, nach 30 Tagen -> `UNVERIFIED`
- Status: `ACTIVE`, `UNVERIFIED`, `ARCHIVED`, `MISSING`
- Quick-Verify: `SELECT media_id FROM media WHERE volume_uuid = ?`, update scanned_at
- **Output:** Medien Lebenszyklus ist nachvollziehbar

### 9.2 DeltaScanEngine
- `src/main/services/deltaScan.ts`: `runDeltaScan(mediaId, rootPath)` -> `DeltaScanResult`
- Temporare Staging Tabelle `_delta_stage`, fuelle mit aktuellen Metadaten
- SQL Vergleich: `LEFT JOIN ... WHERE f.id IS NULL OR f.size != s.size OR f.mtime != s.mtime`
- Loesche nicht mehr existente Dateien, markiere neue als `UNCHECKED`
- **Output:** Delta Scan erkennt Aenderungen ohne Hash

### 9.3 ExclusionService
- `src/main/services/exclusionService.ts`: `markAsFalsePositive(fileIdA, fileIdB)`
- Bidirektional: `WHERE (a= ? AND b= ?) OR (a= ? AND b= ?)`
- Integration in Duplikat SQL: `NOT EXISTS (SELECT 1 FROM duplicate_exclusions WHERE ...)`
- `removeExclusion()`, `isExcluded()`
- **Output:** False-Positives werden korrekt ausgeblendet

---

## Task 10: React UI Grundgeruest & Layout (Mittel, 3 Subtasks)

### 10.1 App Shell + Router
- `src/renderer/App.tsx`: `MemoryRouter` mit 5 Routes (/, /scan, /media, /duplicates, /settings)
- `src/renderer/components/Layout.tsx`: Dreiteiliges Layout (Sidebar + Header + Content + Footer)
- **Output:** Navigierbares Grundgeruest

### 10.2 Sidebar + GlobalSearchBar + StatusBar
- `Sidebar.tsx`: 4 Nav Punkte mit Icons, aktive Route Hervorhebung
- `GlobalSearchBar.tsx`: Debounced Search (300ms), Results Dropdown, `Ctrl+F` Fokus
- `StatusBar.tsx`: System Statistik (Medien, Dateien, Speicher), Refresh alle 30s
- **Output:** Kern UI Komponenten funktionsfaehig

### 10.3 View Skeletons + Routing
- 5 View Dateien: `SearchView.tsx`, `ScanView.tsx`, `MediaView.tsx`, `DuplicatesView.tsx`, `SettingsView.tsx`
- Jede mit Ueberschrift, Platzhalter Inhalt, Typ Definitionen
- `src/renderer/types.ts`: Alle UI Typen
- **Output:** 5 Navigationsziele sichtbar

---

## Task 11: Katalog Suche & Detail Ansicht (Gross, 4 Subtasks)

### 11.1 SearchView Layout + SearchHighlight
- `src/renderer/views/SearchView.tsx`: Zweispaltig (45/55), teilt globale Query
- `src/renderer/components/SearchHighlight.tsx`: Regex basiertes Highlighting
- **Output:** Suchansicht mit Highlighting

### 11.2 ResultList
- `src/renderer/components/ResultList.tsx`: Scrollbare Ergebnisliste
- Pro Eintrag: Icon, Dateiname (fett), Pfad (klein), Groesse, Duplikat Status Farben
- Virtuelles Scrolling via IntersectionObserver (nur sichtbare rendern)
- Klick selektiert fuer Detail Panel
- **Output:** Ergebnisliste mit virtuellen Scrollen

### 11.3 DetailPanel (Datei Info + Medien Standort)
- `src/renderer/components/DetailPanel.tsx`:
  - Abschnitt 1: Dateiname, Pfad, Groesse, Erstellungsdatum, Aenderungsdatum
  - Abschnitt 2: Medien Standort prominent (Volume Label, Typ, Storage Location), `[Oeffnen]` Button
  - Abschnitt 3: Duplikat Info (Status, Liste, Deep Scan Button, Exclusion Button)
- **Output:** Vollstaendige Detailansicht

### 11.4 NoteEditor + Breadcrumb + FilterBar + Keyboard Nav
- `NoteEditor.tsx`: Inline Textarea, Auto-Save nach 2s Inaktivitaet, zeigt geerbte Notiz mit Quelle
- `BreadcrumbNav.tsx`: Klickbare Pfadsegmente, letztes Segment ist aktuelle Datei
- `FilterBar.tsx`: Zusammenklappbar, Media/Extension/Groesse/Datum/Duplikat Status Filter
- Keyboard: Pfeiltasten, Enter, Escape, Tab
- **Output:** Interaktive Hauptansicht

---

## Task 12: Scan Center View (3-Schritt Assistent) (Gross, 4 Subtasks)

### 12.1 ScanView + StepIndicator + StepMedia
- `src/renderer/views/ScanView.tsx`: Step state (1/2/3), sichtbarer Step Indicator
- `ScanStepMedia.tsx`: Option A (bestehendes Medium, Volume UUID Check), Option B (neues Medium Formular)
- Quick-Verify Button bei bekanntem Medium
- **Output:** Schritt 1 funktioniert

### 12.2 ScanStepProfile
- `ScanStepProfile.tsx`: Radio Buttons (Background/Ausgewogen/Turbo), Laufwerk Pfad (Text + Browse)
- `enableDeepScan` Checkbox
- Pflichtfeld Validierung, `[Scan starten]` Button
- **Output:** Schritt 2 konfiguriert den Scan

### 12.3 ScanProgressView
- `ScanProgressView.tsx`: Progress Circle, Dateienanzahl, Speed (Dateien/sec), ETA, Progress Bar
- Aktuelle Datei Anzeige, `[Abbrechen]` Button mit Bestaetigung
- Verbindung via `window.api.startScan()` + `onScanProgress()`
- **Output:** Echtzeit Scan Fortschritt

### 12.4 Scan Summary + Volume UUID + Error Handling
- Nach Scan: Zusammenfassung (Dateien, Duplikate, Medien), Links zu Suche/Duplikaten
- Volume UUID Erkennung: Plattformspezifisch (Windows `stat.dev`, Linux `mountinfo`, Fallback UUID)
- I/O Fehler: Warnung, Scan laeuft weiter. Schwere Fehler: Abbruch + Dialog
- **Output:** Scan UX vollstaendig

---

## Task 13: Medien & Standort Verwaltung (Mittel, 3 Subtasks)

### 13.1 MediaTable
- `src/renderer/components/MediaTable.tsx`: Spalten (Label, Typ, Standort, Status, Dateien, Groesse, Scan Datum)
- Sortierung per Klick auf Spaltenkopf
- Status farblich: Gruen (ACTIVE), Gelb (UNVERIFIED), Grau (ARCHIVED), Rot (MISSING)
- **Output:** Uebersicht aller Medien

### 13.2 MediaDetailDialog
- `src/renderer/components/MediaDetailDialog.tsx`: Detailansicht + Edit Formular
- Aktionen: Standort aendern, Status setzen, Medium loeschen (mit Bestaetigung)
- Loeschung: `DELETE FROM files WHERE media_id = ?`, dann `DELETE FROM media`
- **Output:** Medien koennen bearbeitet werden

### 13.3 Bulk Standort Update + Create Media Form
- `CreateMediaForm.tsx`: Label, Typ, Standort, erzeugt `INSERT INTO media`
- Bulk Update: Mehrere Medien selektieren -> Standort aendern Dialog
- Nach Erstellung: Direktlink zum Scan
- **Output:** Massenaktionen und Neuanlage

---

## Task 14: Duplikate & Speicheranalyse (Gross, 4 Subtasks)

### 14.1 DuplicatesView Layout + Stats
- `src/renderer/views/DuplicatesView.tsx`: 3 Kacheln oben (Kandidaten, Verschwendung, Bestaetigte)
- Statistik laden via `window.api.getSystemStats()`
- **Output:** Dashboard Kacheln mit Werten

### 14.2 DuplicateGroupList
- `DuplicateGroupList.tsx`: Gruppen nach Hash, mit Dateiliste, Anzahl, Groesse
- Status Badges (Gelb = POTENTIAL, Rot = CONFIRMED)
- Aktionen pro Gruppe: Deep Scan, Bestaetigen, Exclusion, Loeschen (nur CONFIRMED)
- **Output:** Gruppierte Duplikat Ansicht

### 14.3 Bulk Deep Scan + Batch Exclusion
- `[Alle Kandidaten deep-scannen]` Button: Zeige Progress "Verifiziere 45 von 128"
- Batch Exclusion: Alle Dateien einer Gruppe als eindeutig markieren
- Nach Abschluss: Automatische Aktualisierung der Ansicht
- **Output:** Bulk Aktionen funktionieren

### 14.4 Storage Visualisierung + Sicherheitsdialoge
- Top 10 groesste Duplikat Gruppen: Horizontale Balken
- `[Loeschen]` Button: 2-stufiger Sicherheitsdialog (CONFIRMED erforderlich, Bestaetigung)
- Loeschung loescht DB Eintrag, nicht physische Datei
- Sortierung/Filter: Nach Groesse/Anzahl/Datum, Nur POTENTIAL/CONFIRMED
- **Output:** Visuelle Analyse mit Sicherheit

---

## Task 15: Lebenszyklus Automatismen & System Integration (Mittel, 3 Subtasks)

### 15.1 LifecycleScheduler + ActivityMonitor
- `src/main/services/lifecycleScheduler.ts`: Alle 6 Stunden, setze `UNVERIFIED` fuer > 30 Tage alte Medien
- `src/main/services/activityMonitor.ts`: Nutze `powerMonitor`, setze Callback bei Aktivitaet/Idle
- Integration: User aktiv -> I/O Profil `BACKGROUND`, Idle > 1 Min -> `TURBO`/`BALANCED`
- **Output:** Automatische Status und Drosselung

### 15.2 Tray + Shortcuts + Window Title
- `trayManager.ts`: Tray Icon mit Kontext Menu (Oeffnen, Scan, Trennen), Click fokussiert Fenster
- `globalShortcuts.ts`: F5 refresh, Fenster Titel zeigt Scan Status
- **Output:** System Integration

### 15.3 Backup + Logger
- Datenbank Backup: Kopiere `catalog.duckdb` nach `userData/backups/`, taeglich, max 7 Rotation
- `logger.ts`: Log Level (DEBUG, INFO, WARN, ERROR), Datei Rotation (10MB, max 3 Dateien)
- **Output:** Daten sind gesichert, Fehler werden geloggt

---

## Task 16: Test Infrastruktur & Integrationstests (Gross, 4 Subtasks)

### 16.1 Test Setup + Fixtures
- `vitest.config.ts`: Environment node, Coverage v8, Thresholds (branches 70, functions 80, lines 75)
- `playwright.config.ts`: Electron E2E Konfiguration
- `tests/fixtures/mockFileSystem.ts`: Temporare Dateien in `os.tmpdir()`, definierte Struktur
- **Output:** Test Laufumgebung

### 16.2 Unit Tests
- `tests/unit/fastHash.test.ts`: Identische Dateien, 1 Byte Aenderung, leere Dateien, kleine Dateien
- `tests/unit/ioThrottler.test.ts`: TURBO kein Delay, BACKGROUND > 3000ms bei 100MB
- `tests/unit/duplicateDetection.test.ts`: DuckDB :memory: mit Testdaten, POTENTIAL und UNIQUE Markierung
- **Output:** Alle Unit Tests gruen

### 16.3 Integration Tests
- `tests/integration/scanPipeline.test.ts`: Scan eines Verzeichnisses, Count Matching, Cancellation, Backpressure, Read Errors
- `tests/integration/noteInheritance.test.ts`: Ordner Notiz vererbt, FILE Prioritaet ueber FOLDER
- **Output:** Kritische Pfade getestet

### 16.4 E2E Tests + Coverage
- `tests/e2e/app.test.ts`: Playwright startet Electron, prueft Navigation, Suche, Footer Statistik
- Coverage Report: `vitest --coverage`, min 70% Abdeckung
- **Output:** E2E Workflow getestet

---

## Task 17: Einstellungen, Export & Electron Packaging (Mittel, 3 Subtasks)

### 17.1 SettingsView UI
- `src/renderer/views/SettingsView.tsx`: Kategorien (Allgemein, Scan, Katalog, Info)
- Toggle fuer Auto-Start, Minimize to Tray, Auto-Pause
- Standard Profil Auswahl, Deep Scan Default
- Einstellungen persistieren via einfacher JSON Datei in `userData`
- **Output:** Settings UI + Persistenz

### 17.2 Export/Import Service
- `exportService.ts`: `exportToJSON(path)` (alle 4 Tabellen), `exportToCSV(dir)` (DuckDB COPY)
- `importService.ts`: `importFromJSON(path)` mit Replace oder Append Option
- Import Validierung (Version, Struktur), Zusammenfassung nach Import
- **Output:** Katalog Export/Import

### 17.3 electron-builder + Build Scripts
- `electron-builder.json`: App ID, Files, extraResources (DuckDB .node), Platform Targets
- Native Binary Handling: `app.isPackaged ? resourcesPath : localPath`
- Build Scripts: `package:win`, `package:mac`, `package:linux`
- Icons: `build/icon.png`, `build/icon.ico`, `build/icon.icns`
- **Output:** Installierbares Package

---

## Task 18: Optimierung, Edge Cases & Abschlussarbeit (Mittel, 3 Subtasks)

### 18.1 Performance Tuning
- DuckDB Pragmas: `threads=2`, `memory_limit='512MB'`, `temp_directory`
- Virtualisierung der Ergebnisliste bei > 500 Eintraegen (IntersectionObserver)
- Lazy Loading: Detail Daten nur bei Klick laden (notes, full_hash, metadata)
- **Output:** Suche < 50ms, RAM < 200MB idle

### 18.2 Edge Cases
- Dateisystem: Unicode Pfade, Sonderzeichen, > 255 Zeichen, tiefe Verzeichnisse (> 50), leere Verzeichnisse
- Speicher: > 1 Mio Dateien getestet, DB > 10GB Kompression
- Scan: Medium entfernt (ENOENT), Schreibschutz, Quick-Verify bei bekanntem Medium
- **Output:** Robuster Produktionscode

### 18.3 DoD Validierung + README
- Pruefe alle MVP DoD Punkte (Task 8 Checkboxen im MVP Scope)
- Benchmark Suite: `tests/benchmark/searchPerformance.test.ts` (100k Dateien < 50ms)
- `tsc --noEmit --strict` = 0 Fehler, ESLint = 0 Warnings
- `README.md`: Projekt Beschreibung, Features, Technologie, Schnellstart
- **Output:** Abnahmebereite App

---

## Aufgaben Uebersicht

```
Phase A: Fundament
  ├── Task 1: Projektgrundlage & Build Infrastruktur (3)
  ├── Task 2: DuckDB Schema & Datenbank Manager (3)
  └── Task 3: IPC Kommunikationsschicht & Preload Bridge (3)

Phase B: Scan Engine
  ├── Task 4: Scan Engine Worker Thread mit Chunk Checkpointing (4)
  ├── Task 5: Fast-Hash Berechnung & I/O Throttling (3)
  └── Task 6: Backpressure Flow Control & Progress Reporting (3)

Phase C: Verifikation
  └── Task 7: 3-Stufen Duplikat Verifikation & Full-Hash Engine (4)

Phase D: Features
  ├── Task 8: Notes, Search & Notiz Vererbung (4)
  └── Task 9: Media Lifecycle, Delta-Scan & Duplikat Exclusions (3)

Phase E: Frontend Kern
  ├── Task 10: React UI Grundgeruest & Layout (3)
  └── Task 11: Katalog Suche & Detail Ansicht (4)

Phase F: Frontend Scan
  ├── Task 12: Scan Center View (3-Schritt Assistent) (4)
  ├── Task 13: Medien & Standort Verwaltung (3)
  └── Task 14: Duplikate & Speicheranalyse (4)

Phase G: Integration
  └── Task 15: Lebenszyklus Automatismen & System Integration (3)

Phase H: Qualitaet
  └── Task 16: Test Infrastruktur & Integrationstests (4)

Phase I: Auslieferung
  ├── Task 17: Einstellungen, Export & Electron Packaging (3)
  └── Task 18: Optimierung, Edge Cases & Abschlussarbeit (3)

Total: 18 Tasks, 61 Subtasks
```

## Parallele Entwicklungszweige

```
Zeit →
├── Phase A ──────────────────────────────────────────────────────────►
├── Phase B ───────────────────────►  (nach A)
│   └── Phase C ────────────────►  (nach B)
├── Phase D ────────────────────►  (kann parallel zu B+C starten, nach A)
├── Phase E ───────────────────────────►  (nach A+D)
│   └── Phase F ──────────────────────────►  (nach B+E)
├── Phase G ─────────────────────────────────────►  (nach C+D+F)
├── Phase H ──────────────────────────────────────────────────►  (kann parallel zu G starten)
└── Phase I ───────────────────────────────────────────────────────────►  (nach H)
```

Empfehlung: Phase B + D parallel entwickeln (setzen beide auf A auf, unabhaengig voneinander). Phase E kann mit Phase B parallel beginnen (Backend + Frontend parallel). Phase H (Tests) kann ab Phase B parallel mitschreiben.
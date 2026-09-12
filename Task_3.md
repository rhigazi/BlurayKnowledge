# Task 3: IPC Kommunikationsschicht & Preload Bridge

## Ziel
Baue die sichere Kommunikationsschicht zwischen Renderer (React) und Main Process (Node.js) über Electron `contextBridge` und `ipcMain/ipcRenderer`. Definiere alle Kanalnamen als Konstanten und implementiere die Handler Grundstruktur für jede Domäne.

## Schritte

### 3.1 IPC Kanal Konstanten
Schreibe `src/shared/ipc-channels.ts` (wird von main und preload verwendet):

```typescript
export const IPC_CHANNELS = {
  // Scan Domain
  SCAN_START: 'scan:start',
  SCAN_CANCEL: 'scan:cancel',
  SCAN_PROGRESS: 'scan:progress',
  SCAN_STATUS: 'scan:status',

  // Database Domain
  DB_SEARCH: 'db:search',
  DB_GET_FILE: 'db:get-file',
  DB_GET_MEDIA: 'db:get-media',
  DB_LIST_MEDIA: 'db:list-media',

  // Notes Domain
  NOTE_CREATE: 'note:create',
  NOTE_UPDATE: 'note:update',
  NOTE_DELETE: 'note:delete',
  NOTE_GET_BY_FILE: 'note:get-by-file',

  // Media Domain
  MEDIA_CREATE: 'media:create',
  MEDIA_UPDATE: 'media:update',
  MEDIA_DELETE: 'media:delete',

  // Duplicate Domain
  DUP_CANDIDATES: 'dup:candidates',
  DUP_EXCLUDE: 'dup:exclude',
  DUP_FULL_HASH: 'dup:full-hash',

  // System Domain
  SYS_STATS: 'sys:stats',
  SYS_OPEN_FILE: 'sys:open-file',
} as const;
```

### 3.2 Preload Script
Schreibe `src/preload/index.ts`:

- `contextBridge.exposeInMainWorld('api', {...})`
- Jeder IPC Kanal wird als Methode exposed: `window.api.searchCatalog(query)`, `window.api.startScan(mediaId)`, etc.
- Alle Aufrufe nutzen `ipcRenderer.invoke(channel, ...args)` für request/response
- Empfange `ipcRenderer.on` Events für Push Nachrichten wie `scan:progress`

### 3.3 TypeScript Typen fuer die API
Schreibe `src/shared/api-types.ts`:

```typescript
export interface ElectronAPI {
  // Scan
  startScan(mediaId: number, profile: ScanProfile): Promise<ScanResult>;
  cancelScan(): Promise<void>;
  onScanProgress(callback: (progress: ScanProgress) => void): () => void;

  // Database
  searchCatalog(query: string, filters?: SearchFilters): Promise<SearchResult[]>;
  getFile(fileId: string): Promise<FileEntry | null>;
  listMedia(): Promise<MediaEntry[]>;

  // Notes
  createNote(note: CreateNoteRequest): Promise<NoteEntry>;
  updateNote(noteId: number, text: string): Promise<NoteEntry>;
  deleteNote(noteId: number): Promise<void>;
  getNotesForFile(fileId: string): Promise<NoteEntry[]>;

  // Media
  createMedia(media: CreateMediaRequest): Promise<MediaEntry>;
  updateMedia(mediaId: number, updates: Partial<MediaEntry>): Promise<MediaEntry>;

  // Duplicates
  getDuplicateCandidates(): Promise<DuplicateCandidate[]>;
  excludePair(fileIdA: string, fileIdB: string): Promise<void>;

  // System
  getSystemStats(): Promise<SystemStats>;
}

// Typen exportieren
export type ScanProfile = 'TURBO' | 'BALANCED' | 'BACKGROUND';
export interface ScanProgress { ... }
export interface FileEntry { ... }
// etc.
```

### 3.4 Main Process IPC Handler (Grundstruktur)
Schreibe `src/main/ipc/index.ts`:

- Registriere alle `ipcMain.handle(channel, handler)` in einer `registerAllHandlers(db: DatabaseManager)` Funktion
- Delegiere an domaenspezifische Handler Dateien:
  - `src/main/ipc/scanHandler.ts` (Platzhalter, wirft `NOT_IMPLEMENTED`)
  - `src/main/ipc/dbHandler.ts` (fuehrt SQL Abfragen aus)
  - `src/main/ipc/noteHandler.ts`
  - `src/main/ipc/mediaHandler.ts`
  - `src/main/ipc/sysHandler.ts` (gibt System Stats)

### 3.5 Progress Event System
- `src/main/ipc/progressEmitter.ts`: Zentrale Event Bus Klasse
- Erlaubt dass Scan Engine Progress Events feuert ohne direkte IPC Kenntnis
- `ProgressEmitter.on('scan', callback)` / `emit('scan', data)`
- Der Scanner kann `progressEmitter.emit('scan', { ... })` aufrufen
- Der Main Process leitet an `webContents.send('scan:progress', data)` weiter

### 3.6 Type Safety
- Alle IPC Parameter und Rueckgabewerte sind getypt
- Nutze `ipcMain.handle` mit generischem Channel Mapping
- Preload API Typen werden in einer `.d.ts` deklariert fuer Auto-Vervollstaendigung im Renderer

## Akzeptanzkriterien
- [ ] `window.api` ist im Renderer definiert mit allen Methoden
- [ ] `ipcMain.handle('db:search', ...)` kann aufgerufen werden und gibt Ergebnis zurueck
- [ ] Progress Events erreichen den Renderer via `window.api.onScanProgress()`
- [ ] TypeScript Typen sind konsistent zwischen main, preload und renderer
- [ ] `registerAllHandlers` wird beim App Start aufgerufen
- [ ] `scan:start` wirft eine sinnvolle Fehlermeldung (noch nicht implementiert)

## Abgrenzung
- Handler sind Platzhalter oder minimal (dbHandler hat echte SQL, scanHandler erst spaeter)
- Keine Geschaeftslogik in den Handlern
- Der Renderer kann die API aufrufen, bekommt aber nur Dummy Daten oder Fehler zurueck
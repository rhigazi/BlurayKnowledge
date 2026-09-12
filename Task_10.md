# Task 10: React UI Grundgeruest & Layout

## Ziel
Baue das fundamentale UI Layout der Electron SPA mit React: Sidebar Navigation, globalen Header mit Schnellsuche, Footer mit Statuszeile und die vier Hauptansichten als Router Ziele.

## Schritte

### 10.1 React Router & Grundstruktur
Schreibe `src/renderer/App.tsx`:

```tsx
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { SearchView } from './views/SearchView';
import { ScanView } from './views/ScanView';
import { MediaView } from './views/MediaView';
import { DuplicatesView } from './views/DuplicatesView';
import { SettingsView } from './views/SettingsView';

export function App() {
  return (
    <MemoryRouter>
      <Layout>
        <Routes>
          <Route path="/" element={<SearchView />} />
          <Route path="/scan" element={<ScanView />} />
          <Route path="/media" element={<MediaView />} />
          <Route path="/duplicates" element={<DuplicatesView />} />
          <Route path="/settings" element={<SettingsView />} />
        </Routes>
      </Layout>
    </MemoryRouter>
  );
}
```

### 10.2 Layout Komponente
Schreibe `src/renderer/components/Layout.tsx`:

```tsx
// Dreiteiliges Layout:
// +-----------------------+
// | Header | Schnellsuche  |
// +--------+--------------+
// | Side   | Main Content |
// | bar    | (Outlet)     |
// +--------+--------------+
// | Footer | Status       |
// +-----------------------+
```

- **Sidebar (links, 240px breit):** Navigation Icons/Links zu den 4 Hauptansichten
- **Header (oben):** Globales Suchfeld (immer sichtbar), `Ctrl+F` / `Cmd+F` Shortcut
- **Content (mitte):** Geroutete View Komponente
- **Footer (unten):** Katalog Statistik (Anzahl Medien, Dateien, Speicher)

### 10.3 Sidebar Navigation
Schreibe `src/renderer/components/Sidebar.tsx`:

- Vier Navigationspunkte als Icons + Label:
  - 🔍 Suche (Pfad `/`)
  - 💿 Scannen (Pfad `/scan`)
  - 📦 Medien (Pfad `/media`)
  - 🔗 Duplikate (Pfad `/duplicates`)
- Aktive Route wird hervorgehoben
- Schlankes Design, keine Sub-Navigation

### 10.4 Global Search Header
Schreibe `src/renderer/components/GlobalSearchBar.tsx`:

```tsx
export function GlobalSearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isSearching, setIsSearching] = useState(false);

  // Debounced Search (300ms)
  useEffect(() => {
    if (query.length < 2) { setResults([]); return; }
    const timer = setTimeout(async () => {
      setIsSearching(true);
      const res = await window.api.searchCatalog(query);
      setResults(res);
      setIsSearching(false);
    }, 300);
    return () => clearTimeout(timer);
  }, [query]);

  return ( /* Input Feld + Results Dropdown */ );
}
```

- Immer sichtbar, unabhaengig von der aktuellen View
- `Ctrl+F` / `Cmd+F` setzt Fokus auf das Suchfeld
- Zeigt erste 10 Ergebnisse als Dropdown

### 10.5 Footer Status Bar
Schreibe `src/renderer/components/StatusBar.tsx`:

```tsx
export function StatusBar() {
  const [stats, setStats] = useState<SystemStats>({ mediaCount: 0, fileCount: 0, totalSizeBytes: 0 });

  useEffect(() => {
    const load = async () => {
      const s = await window.api.getSystemStats();
      setStats(s);
    };
    load();
    const interval = setInterval(load, 30000); // Refresh alle 30s
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="status-bar">
      <span>Medien: {stats.mediaCount}</span>
      <span>Dateien: {stats.fileCount.toLocaleString()}</span>
      <span>Speicher: {formatBytes(stats.totalSizeBytes)}</span>
      <span>Stand: {stats.lastScanDate || 'Noch kein Scan'}</span>
    </div>
  );
}
```

### 10.6 View Platzhalter
Schreibe jede der fuenf Views als minimales Skelett:

- `SearchView.tsx`: "Suche" Ueberschrift, zentrale Suchmaske, Ergebnisbereich (leer)
- `ScanView.tsx`: "Scannen" Ueberschrift, Scan Start Bereich (Platzhalter)
- `MediaView.tsx`: "Medien" Ueberschrift, Medien Tabelle (Platzhalter)
- `DuplicatesView.tsx`: "Duplikate" Ueberschrift, Platzhalter
- `SettingsView.tsx`: "Einstellungen" Ueberschrift, Platzhalter

### 10.7 Lokales CSS / Styling
- Keine externen CSS Frameworks (Zero External Dependencies)
- Eigene CSS Module oder einfaches CSS in `src/renderer/assets/styles.css`
- Dark Theme als Default (passend fuer Archiv/Medien Anwendung)
- Responsive nur soweit noetig fuer verschiedene Fenstergroessen

### 10.8 TypeScript Typen fuer View Props
- `src/renderer/types.ts`: Alle Typen die in Views benoetigt werden
- `SearchResult`, `MediaEntry`, `FileEntry`, `DuplicateCandidate`, `ScanProgress`
- Uebernahme aus `src/shared/api-types.ts`

## Akzeptanzkriterien
- [ ] App zeigt das 3-teilige Layout (Sidebar, Content, Footer) korrekt an
- [ ] Navigation wechselt zwischen den 5 Views
- [ ] Globale Schnellsuche ist in jeder View verfuegbar
- [ ] `Ctrl+F` / `Cmd+F` fokussiert das Suchfeld
- [ ] Footer zeigt System Statistik an
- [ ] Keine externen CSS/JS Abhaengigkeiten geladen
- [ ] Sidebar zeigt aktive Route hervorgehoben an
- [ ] Dark Theme sieht lesbar aus

## Abgrenzung
- Views sind Platzhalter ohne echte Funktionalitaet
- Suche zeigt noch keine Ergebnisse (kein IPC Call in Task 10)
- Keine komplexen UI Komponenten (Tabellen, Forms, Dialogs)
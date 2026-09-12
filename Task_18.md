# Task 18: Optimierung, Edge Cases & Abschlussarbeit

## Ziel
Verfeinere die App fuer den Production Einsatz: Performance Optimierung, Edge Case Behandlung, Code Sauberkeit, und abschliessende Validierung gegen die Definition of Done.

## Schritte

### 18.1 Performance Optimierung

**DuckDB Tuning:**
```sql
-- Setze DuckDB Optimierung Parameter beim DB Start
PRAGMA threads = 2;             -- Max 2 Threads (nicht mit UI konkurrieren)
PRAGMA memory_limit = '512MB';  -- Max 512 MB RAM fuer DuckDB
PRAGMA temp_directory = '/tmp/duckdb_tmp';  -- Temp Verzeichnis
PRAGMA enable_profiling = 'json';
```

**Virtualisierung der Ergebnisliste:**
- Bei > 500 Suchergebnissen: Virtuelles Scrolling aktivieren
- Nutze `react-window` oder einfaches IntersectionObserver Laden
- Nur sichtbare Elemente werden gerendert

**Lazy Loading von Detail Daten:**
- Ergebnisliste laedt nur: id, file_name, file_path, file_size_bytes, duplicate_state
- Detail Daten (notes, full_hash, metadata) nur bei Klick laden
- Vermeidet teure JOINs in der initialen Suche

### 18.2 Edge Cases

**Dateisystem Grenzfaelle:**
- Pfade laenger als 255 Zeichen: Verkuerze in der UI, speichere vollstaendig in DB
- Unicode Dateinamen (Chinesisch, Kyrillisch, Emojis): Teste korrekte Speicherung
- Dateien mit Sonderzeichen im Pfad (`\`, `:`, `*`, `?`): escape in Queries
- Sehr tiefe Verzeichnisstrukturen (> 100 Ebenen): Setze max Tiefe auf 50, logge Warnung
- Leere Verzeichnisse: Als `is_directory = TRUE` mit size=0 speichern

**Speicher Grenzfaelle:**
- Mehr als 1 Million Dateien im Katalog: Teste Suchzeit < 100ms
- Datenbank > 10 GB: Teste DuckDB Kompression und Performance
- RAM Verbrauch > 1 GB: Drossle Background Tasks, zeige Warnung

**Scan Grenzfaelle:**
- Medium wird waerend Scan entfernt: Erkennung via mehrfacher ENOENT -> Abbruch
- Schreibschutz aktiviert: Scan laeuft, aber Deep Scan kann nix schreiben -> Warnung
- Bereits gescanntes Medium erneut einlegen: Quick-Verify Angebot (nicht erneut scannen)

### 18.3 Code Sauberkeit

**Typisierung pruefen:**
- `tsc --noEmit --strict` durchlaufen ohne Fehler
- Alle `any` Typen entfernen oder durch korrekte Typen ersetzen
- Keine `eslint-disable` Kommentare ohne Begruendung

**Linting:**
- ESLint Konfiguration (`eslint.config.js`)
- `npm run lint` durchlaufen ohne Fehler
- Konsistente Code Formatierung (Prettier oder einfache Konvention)

**Dead Code Entfernen:**
- Unbenutzte Importe entfernen
- Unbenutzte Komponenten/Funktionen entfernen
- TODO Kommentare aufloesen oder in Tasks umwandeln

### 18.4 Definition of Done Validierung

Pruefe jeden Punkt der MVP DoD:

**A. System & Datenintegritaet**
- [ ] Electron App laeuft offline ohne externe Abhaengigkeiten
- [ ] DuckDB speichert media, files und notes persistent
- [ ] IPC Kommunikation ist asynchron und blockiert die UI nicht

**B. Scan Engine**
- [ ] Volume UUID wird erkannt (verhindert Double-Scans)
- [ ] Fast-Hash wird fuer jede Datei korrekt berechnet
- [ ] Scan Fortschritt (Prozent, aktuelle Datei) wird in Echtzeit angezeigt
- [ ] Lesefehler fuehren nicht zum Absturz der App

**C. UI & Suche**
- [ ] Suchergebnisse erscheinen in < 50ms (DuckDB FTS)
- [ ] Notizen koennen direkt im Detail Bereich erstellt/geaendert werden
- [ ] Physischer Standort ist bei jedem Suchtreffer prominent sichtbar
- [ ] Ctrl+F / Cmd+F fokussiert das Suchfeld

**D. Performance Benchmarks**
- [ ] Suche bleibt bei > 100.000 Dateien fluesig
- [ ] RAM Verbrauch im Leerlauf < 200 MB

### 18.5 Benchmark Test Suite
Schreibe `tests/benchmark/searchPerformance.test.ts`:

```typescript
describe('Search Performance', () => {
  it('should search 100k files under 50ms', async () => {
    // 1. Fuelle DB mit 100.000 generischen Datei Eintraegen
    // 2. Suche nach einem spezifischen Begriff
    // 3. Erwartung: < 50ms Antwortzeit
  });

  it('should keep RAM under 200MB idle', async () => {
    // 1. Warte 5 Sekunden nach App Start
    // 2. Pruefe process.memoryUsage().heapUsed
    // 3. Erwartung: < 200 MB
  });

  it('should handle concurrent queries', async () => {
    // 1. Sende 10 Suchanfragen parallel
    // 2. Erwartung: Alle 10 werden innerhalb von 500ms beantwortet
  });
});
```

### 18.6 README Dokumentation
Schreibe `README.md` fuer das Repository:

```markdown
# BlurayKnowledge

Autarke Electron SPA zur Katalogisierung von Offline-Medien (DVD, BD-R, USB).

## Funktionen
- **Scanning:** Rekursiver Dateisystem Scan mit Fast-Hash Duplikaterkennung
- **Suche:** Volltextsuche ueber Dateinamen, Pfade und Notizen
- **Notizen:** Hierarchische Vererbung (Ordner -> Dateien)
- **Duplikaterkennung:** 3-Stufen Verifikation (Size -> Fast-Hash -> Full-Hash)
- **Medienverwaltung:** Physische Standorte, Lebenszyklus Management
- **100% Offline:** Keine Internetverbindung erforderlich

## Technologie
- Electron, React, TypeScript, Vite
- DuckDB (eingebettete spaltenbasierte Datenbank)
- Worker Threads fuer Hintergrund Scans

## Schnellstart
1. `npm install`
2. `npm run build`
3. `npm start`

## Dokumentation
Siehe `docs/specs/` fuer die Architektur Entscheidungen.
```

### 18.7 Abschluss Check
- [ ] Alle Tasks von 1 bis 17 sind abgeschlossen
- [ ] Alle Tests bestehen (`npm test`)
- [ ] Build erzeugt fehlerfrei (`npm run build`)
- [ ] App startet aus dem installierten Package
- [ ] Scan eines Testmediums funktioniert Ende-zu-Ende
- [ ] Suche unter 50ms bei 100k Dateien
- [ ] RAM unter 200MB im Leerlauf
- [ ] Linting ohne Fehler

## Akzeptanzkriterien
- [ ] DuckDB Performance Parameter sind optimiert
- [ ] Ergebnisliste virtualisiert bei grossen Datenmengen
- [ ] Detail Daten werden lazy geladen
- [ ] Unicode Dateinamen und Sonderzeichen funktionieren
- [ ] Scan Abbruch bei entferntem Medium erkannt
- [ ] `tsc --noEmit --strict` hat 0 Fehler
- [ ] Alle MVP DoD Punkte sind abgehakt
- [ ] Benchmark Tests bestehen (Suche < 50ms, RAM < 200MB)
- [ ] README ist vollstaendig
- [ ] App kann von Null gebaut und ausgefuehrt werden

## Abgrenzung
- Multi-Worker Threading (Task 15 in Roadmap) ist P3 und optional
- Netzwerkfunktionen sind explizit NICHT im Scope
- Auto-Updater fuer neue Versionen kommt spaeter
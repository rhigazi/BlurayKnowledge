# Task 16: Test Infrastruktur & Integrationstests

## Ziel
Richte die Test Pyramide ein: Unit Tests (Vitest), Integration Tests (Vitest + DuckDB :memory: + Worker Threads), und E2E Tests (Playwright for Electron). Schreibe Tests fuer die kritischsten Komponenten.

## Schritte

### 16.1 Test Setup
- `vitest.config.ts` fuer Unit und Integration Tests
- Playwright Setup: `playwright.config.ts` fuer Electron E2E Tests
- Test Fixtures in `tests/fixtures/` (Mock Dateisystem Strukturen, Beispiel Daten)

### 16.2 Vitest Konfiguration
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',     // Kein jsdom (Electron Main Process)
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
    testTimeout: 10000,
  },
  resolve: {
    alias: {
      '@main': '/src/main',
      '@shared': '/src/shared',
    },
  },
});
```

### 16.3 MemFS: Mock Dateisystem
Schreibe `tests/fixtures/mockFileSystem.ts`:

```typescript
// Baue eine deterministische Verzeichnis Struktur im Speicher auf:
// /mock-media/
//   Dokumente/
//     Rechnung_2023.pdf (12 KB)
//     Rechnung_2024.pdf (12 KB) (Duplikat Kandidat)
//     Urlaubsplanung.xlsx (45 KB)
//   Fotos/
//     IMG_001.jpg (2.4 MB)
//     IMG_002.jpg (2.4 MB) (Duplikat Kandidat)
//   backup.zip (500 MB) (leere Datei als Platzhalter)
```

- Erstelle temporaere Dateien auf der Festplatte (in `os.tmpdir()`)
- Nach Test: Loesche temporaeres Verzeichnis
- Definiere erwartete Ergebnisse (Anzahl Dateien, erwartete Hashes, Duplikate)

### 16.4 Unit Tests: Fast-Hash
Schreibe `tests/unit/fastHash.test.ts`:

```typescript
describe('computeFastHash', () => {
  it('should produce identical hash for identical files', async () => {
    const hash1 = await computeFastHash(TEST_FILE);
    const hash2 = await computeFastHash(TEST_FILE);
    expect(hash1.hash).toBe(hash2.hash);
  });

  it('should detect single byte change', async () => {
    // Aendere eine Datei um 1 Byte im Rumpf (nicht Header/Footer)
    // Erwartung: Hash UNTERSCHIEDLICH
  });

  it('should handle empty file (0 bytes)', async () => {
    const hash = await computeFastHash(EMPTY_FILE);
    expect(hash.hash).toBe('EMPTY');
  });

  it('should handle files smaller than buffer (0-16KB)', async () => {
    // Datei mit exakt 4 KB Inhalt
    // Erwartung: Kein Buffer Ueberlauf
  });

  it('should detect rename (same content, different name)', async () => {
    // Zwei Dateien mit identischem Inhalt aber unterschiedlichem Namen
    // Erwartung: Gleicher Hash (Duplikat Erkennung)
  });
});
```

### 16.5 Unit Tests: I/O Throttler
Schreibe `tests/unit/ioThrottler.test.ts`:

```typescript
describe('IOThrottler', () => {
  it('should not delay in TURBO mode', async () => {
    const throttler = new IOThrottler('TURBO');
    const start = Date.now();
    for (let i = 0; i < 100; i++) await throttler.throttle(1024);
    expect(Date.now() - start).toBeLessThan(100); // Kein signifikanter Delay
  });

  it('should limit throughput in BACKGROUND mode', async () => {
    const throttler = new IOThrottler('BACKGROUND');
    const start = Date.now();
    for (let i = 0; i < 100; i++) await throttler.throttle(1_000_000); // 1 MB pro Datei
    // Erwartung: Max 30 MB/s, also bei 100 MB = mindestens 3.3 Sekunden
    const elapsed = Date.now() - start;
    expect(elapsed).toBeGreaterThan(3000);
  });
});
```

### 16.6 Unit Tests: Duplikat SQL Queries
Schreibe `tests/unit/duplicateDetection.test.ts`:

```typescript
describe('Duplicate Detection SQL', () => {
  it('should identify POTENTIAL_DUPLICATE based on size + fast-hash', async () => {
    // 1. Erstelle DuckDB :memory: mit Schema
    // 2. Fuelle mit Testdaten (zwei identische Dateien, eine eindeutige)
    // 3. Fuehre SQL aus
    // 4. Erwartung: Genau 2 Dateien als POTENTIAL_DUPLICATE markiert
  });

  it('should mark UNIQUE for files without matches', async () => {
    // Eindeutige Datei -> duplicate_state = UNIQUE
  });

  it('should find candidates without full_hash', async () => {
    // Nur Dateien deren full_hash NULL ist
  });
});
```

### 16.7 Integration Tests: Scan Pipeline
Schreibe `tests/integration/scanPipeline.test.ts`:

```typescript
describe('Scan Pipeline', () => {
  it('should scan a directory and persist all files', async () => {
    // 1. Erstelle temporaeres Verzeichnis mit N definierten Dateien
    // 2. Starte ScanWorker via Controller
    // 3. Warte auf SCAN_COMPLETE
    // 4. Pruefe: files Tabelle enthaelt exakt N Eintraege
    // 5. Pruefe: Alle Pfade und Dateinamen korrekt
    // 6. Pruefe: Fast-Hash wurde fuer jede Datei berechnet
  });

  it('should handle cancellation mid-scan', async () => {
    // 1. Starte Scan mit vielen Dateien
    // 2. Nach 0.5s: cancelScan() aufrufen
    // 3. Erwartung: Scan stoppt, bereits erfasste Chunks bleiben in DB
  });

  it('should not deadlock on backpressure', async () => {
    // 1. Simuliere langsamen Main Process (kuenstlicher Delay im BATCH_ACK)
    // 2. Starte Scan
    // 3. Erwartung: Kein Deadlock, Worker wartet auf ACK
  });

  it('should handle read errors gracefully', async () => {
    // 1. Fuege eine Datei mit restriktiven Berechtigungen hinzu
    // 2. Starte Scan
    // 3. Erwartung: Scan laeuft weiter, fehlerhafte Datei wird uebersprungen
  });
});
```

### 16.8 Integration Tests: Notiz Vererbung
Schreibe `tests/integration/noteInheritance.test.ts`:

```typescript
describe('Note Inheritance', () => {
  it('should inherit folder note to all child files', async () => {
    // 1. Erstelle Medium mit Ordnerstruktur
    // 2. Fuege Ordner-Notiz hinzu
    // 3. Suche: Alle Unterdateien zeigen die geerbte Notiz
  });

  it('should prioritize FILE note over FOLDER note', async () => {
    // 1. Ordner-Notiz: "Fotos"
    // 2. Datei-Notiz: "Urlaub Toskana"
    // 3. Suche: effektive Notiz = "Urlaub Toskana"
  });
});
```

### 16.9 E2E Tests: Playwright for Electron
Schreibe `tests/e2e/app.test.ts`:

```typescript
import { _electron as electron } from 'playwright';

describe('App E2E', () => {
  let app: ElectronApplication;
  let window: Page;

  beforeAll(async () => {
    app = await electron.launch({ args: ['dist/main/index.js'] });
    window = await app.firstWindow();
  });

  afterAll(async () => { await app.close(); });

  it('should show 4 navigation items in sidebar', async () => {
    const sidebarItems = await window.locator('.sidebar-nav-item').all();
    expect(sidebarItems.length).toBe(4);
  });

  it('should navigate to scan view on click', async () => {
    await window.click('text=Scannen');
    await expect(window.locator('h1')).toHaveText('Datenträger Erfassen');
  });

  it('should perform search and display results', async () => {
    // Wenn Testdaten vorhanden sind
    await window.fill('input[aria-label="Suche"]', 'Rechnung');
    await expect(window.locator('.search-results')).toBeVisible();
  });

  it('should show system stats in footer', async () => {
    const footer = window.locator('.status-bar');
    await expect(footer).toBeVisible();
    await expect(footer).toContainText('Medien');
  });
});
```

### 16.10 Test Coverage Ziel
```typescript
// vitest.config.ts
export default defineConfig({
  test: { coverage: { provider: 'v8', thresholds: { branches: 70, functions: 80, lines: 75 } } },
});
```

- Unit Tests: 80% Code Coverage
- Integration Tests: Alle kritischen Pfade (Scan, Duplikat, Suche)
- E2E Tests: Kern Workflow (Suche, Scan, Navigation)

## Akzeptanzkriterien
- [ ] `vitest run` laeuft ohne Fehler
- [ ] Fast-Hash Unit Tests bestehen alle definierten Faelle
- [ ] I/O Throttler Unit Tests bestehen Performance Asserts
- [ ] Duplikat Detection SQL Tests liefern korrekte Status
- [ ] Scan Pipeline Integrationstest erfasst korrekt alle Dateien
- [ ] Scan Cancellation erhaelt bereits erfasste Chunks
- [ ] Notiz Vererbung funktioniert wie spezifiziert
- [ ] Playwright startet Electron und navigiert durch die App
- [ ] Coverage Report zeigt mindestens 70% Abdeckung
- [ ] Alle Tests laufen unabhaengig und ohne Side-Effects

## Abgrenzung
- Kein visuelles Regression Testing (Screenshots)
- Kein Performance Benchmarking in dieser Task (nur Funktionalitaet)
- Temporare Testdaten werden nach jedem Test geloescht
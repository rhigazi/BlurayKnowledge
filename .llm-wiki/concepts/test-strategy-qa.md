---
title: Test-Strategie & Qualitätssicherung (QA)
type: Concept
description: Strategie zur Sicherstellung der Datenintegrität und Systemstabilität durch automatisierte Tests und Fixtures.
status: active
created: 2026-09-05T13:25:00Z
timestamp: 2026-09-05T13:13:21Z
---

# Test-Strategie & Qualitätssicherung (QA)

Um die Zuverlässigkeit eines Offline-Archivs zu garantieren, basiert die Qualitätssicherung auf einer strikten **Test-Pyramide** und einem **Fixtures-basierten Testkit**, das die Integrität des Scanners und der Datenbank sicherstellt.

## 1. Test-Pyramide & Tooling

Wir setzen auf eine effiziente Pipeline mit minimalem Overhead:

*   **Vitest:** Schneller Runner für Unit- und Integrationstests (ESM/TypeScript).
*   **MemFS:** In-Memory-Dateisystem zur Simulation komplexer Verzeichnisstrukturen ohne physische I/O-Last.
*   **DuckDB In-Memory:** Testen von SQL-Logik direkt auf einer flüchtigen `:memory:` Datenbank.
*   **Playwright for Electron:** End-to-End (E2E) Tests für die kritischen Pfade (App-Start, Scan-Prozess, Suche).

## 2. Sicherstellung der Datenintegrität

Das größte Risiko ist der Verlust von Dateien während des Scans oder eine unvollständige Indizierung.

### A. Integritäts-Validierung (Scann-Check)
Durch den Einsatz von `MemFS` werden deterministische Test-Szenarien erstellt. Nach dem Scan wird die Datenbank gegen das Mock-Dateisystem validiert:
*   **Count-Matching:** Die Anzahl der Einträge in `files` muss exakt der Anzahl der Dateien im Mock-System entsprechen.
*   **Path-Matching:** Alle Pfade und Sonderzeichen müssen identisch mit dem Dateisystem sein.

### B. Fast-Hash Verifikation
Da die Duplikat-Erkennung auf dem `Fast-Hash` basiert, werden folgende Fälle getestet:
*   **Identität:** Gleicher Inhalt/Größe $\rightarrow$ identischer Hash.
*   **Kollisions-Resistenz:** Minimale Änderungen im Datei-Rumpf (zwischen Header/Footer) $\rightarrow$ unterschiedlicher Hash.
*   **Edge Cases:** Dateien kleiner als die Puffergröße ($< 16 \text{ KB}$) müssen korrekt verarbeitet werden.

## 3. Test-Matrix & Abdeckung

| Ebene | Fokus | Tooling | Ziel |
| :--- | :--- | :--- | :--- |
| **Unit** | `Fast-Hash`, SQL Queries, Zustand (Zustand) | Vitest | Maximale Logik-Abdeckung |
| **Integration** | Worker-Thread $\leftrightarrow$ Main $\leftrightarrow$ DuckDB | Vitest + Worker Threads | Korrekte Datenflüsse & Transaktionen |
| **E2E** | UI Flow (Suche, Scan-Start, Notizen) | Playwright | Sicherstellung des User-Workflows |

## 4. Kritische Test-Szenarien (Edge Cases)

*   **Abbruch-Szenario:** Simulation eines `SIGINT` oder einer unterbrochenen IPC-Verbindung während eines Scans $\rightarrow$ Erwartung: Vollständiges `ROLLBACK` der Transaktion.
*   **I/O-Fehler:** Simulation von `EACCES` (Zugriff verweigert) oder `ENOENT` (Datei weg) $\rightarrow$ Erwartung: Scan läuft weiter, Fehler wird geloggt.
*   **Großes Volumen:** Test mit $100.000+$ Mock-Einträgen $\rightarrow$ Erwartung: Suchzeit $< 50 \text{ ms}$ und stabiler RAM-Verbrauch.

## timeline
- 2026-09-05T13:25:00Z: Test-Strategie und QA-Plan definiert.
EOF
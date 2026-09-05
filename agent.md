# Projektkarte: BlurayKnowledge (Roadmap & Architektur)

Dies ist die strategische Landkarte zum Projekt, die den Übergang von der Konzeptphase zur Implementierung steuert.

> [!SUCCESS]
> **Aktueller Status:** ✅ **Konzeptphase abgeschlossen**
> **Ziel:** Übergang zur Implementierung der Kernkomponenten (Prototyping).
> **Regel:** Wir bewegen uns nun von der reinen Dokumentation hin zur Erstellung von funktionalem Code.

## 🛠️ Arbeitsweise (Coding-Protokoll)

Um den Fortschritt während der Implementierungsphase transparent zu machen, gilt für den Coder (**Jules**):

### ⚠️ Strenges Konzeptions-Prinzip (Top-Down)
**Roadmap & Architektur müssen immer synchron mit dem Code geführt werden.** Es gilt folgende strikte Reihenfolge:
1. **Konzeption vor Implementierung:** Neue Features dürfen im Code erst implementiert werden, wenn sie zuvor in das entsprechende Konzeptdokument (`concepts/*.md`) integriert und dort festgeschrieben wurden.
2. **Diskussion $\rightarrow$ Konzept $\rightarrow$ Code:** Wenn Änderungen besprochen werden, müssen diese zwingend zuerst in die Konzepte eingearbeitet werden. Erst nach der Aktualisierung der Dokumentation erfolgt die Umsetzung im Code und die abschließende Dokumentation der Änderung.
3. **Synchronität:** Jede signifikante Änderung am Code, die die Architektur beeinflusst, erfordert eine sofortige Aktualisierung der `agent.md` oder der relevanten Konzept-Dateien.

### 📋 Progress-Tracking
Jules muss während der gesamten Arbeit an der Implementierung eine Datei `progress.md` im Root-Verzeichnis führen.
- **Inhalt von `progress.md`:** Diese Datei dient als tägliches Logbuch und soll folgende Punkte enthalten:
    * **Status:** Aktueller Arbeitsstand (z. B. "In Arbeit", "Abgeschlossen", "Blockiert").
    * **Erledigte Tasks:** Liste der heute/in diesem Schritt implementierten Funktionen oder Fixes.
    * **Offene Punkte / TODOs:** Identifizierte technische Schulden oder nächste Schritte.
    * **Herausforderungen:** Probleme oder Entscheidungen, die im Code getroffen wurden.

---

## 🗺️ Architektur-Landkarte (Konzept-Module)

Die Architektur ist in vier funktionale Säulen unterteilt, die durch einen zentralen Orchestrator gesteuert werden.

### 1. Kern-Engine (Resilienz & Durchsatz)
- [x] **Scan Engine Architecture** (`concepts/scan-engine-architecture.md`)
    - *Fokus:* Chunked Checkpointing & ACID-Transaktionen in DuckDB.
- [x] **High-Speed Ingestion** (`concepts/high-speed-ingestion.md`)
    - *Fokus:* Backpressure Management & ACK-basierte Flow-Control.

### 2. Verifikation & Integrität (Präzision)
- [x] **Verification System** (`concepts/verification-system.md`)
    - *Fokus:* 3-Stufen-Modell (Fast-Hash $\rightarrow$ Candidate Matching $\rightarrow$ Deep Scan).

### 3. Ressourcen-Governance (User Experience)
- [x] **I/O-Throttling** (`concepts/io-throttling.md`)
    - *Fokus:* Adaptive Drosselung & Profile (Turbo/Balanced/Background).
- [x] **Global Orchestration** (`concepts/orchestration-strategy.md`)
    - *Fokus:* Zentrales Ressourcen-Budgeting & Prioritäts-Management (P0-P3).

### 4. Datenmodell & Schema
- [x] **Unified Schema Design** (`concepts/metadata-strategy.md`)
    - *Fokus:* Sparse JSON Metadaten & Columnar Performance in DuckDB.

---

## 🚀 Roadmap zur Implementierung

### Phase 1: Finalisierung des Designs (ABGESCHLOSSEN)
- [x] Definition der Core-Engine.
- [x] Design der Flow-Control & Verifikation.
- [x] Entwurf der globalen Orchestrierung.
- [x] Festlegung des Unified Schemas & Metadata Strategy.

### Phase 2: Prototyping (NÄCHSTER SCHRITT)
- [ ] Projektstruktur aufbauen (src/main, src/worker, etc.).
- [ ] Implementierung des `IOThrottler` Moduls in Node.js.
- [ ] Aufbau des minimalen `scanWorker.ts` mit ACK-Logik.
- [ ] Setup der DuckDB-Instanz mit dem neuen Sparse-Schema.

### Phase 3: Integration & Testing
- [ ] Integration von Orchestrator und Worker.
- [ ] Stress-Tests (Memory Pressure & Disk Thrashing).
- [ ] Validierung des Deep-Scan-Workflows.

---
*Hinweis: Diese Datei dient als strategischer Kompass für den Agenten.*
---
title: Konsolidierte Taskliste & Umsetzungs-Roadmap
type: Roadmap
description: Strategische Task-Priorisierung und Implementierungs-Phasen für das BlurayKnowledge Projekt.
status: active
created: 2026-09-05T15:45:00Z
timestamp: 2026-09-05T15:45:00Z
---

# Konsolidierte Taskliste & Umsetzungs-Roadmap

## Prioritätsstufen Overview

* **P0 (Kritisch / Blocker):** Kernarchitektur, Datenintegrität und Systemstabilität (Muss vor dem ersten Release stehen).
* **P1 (Hoch):** Kern-Features, Performance-Optimierung und User Experience.
* **P2 (Mittel):** Komfort-Funktionen, erweiterte Automatisierung und Edge-Case-Handhabung.
* **P3 (Niedrig / Optional):** System-Feinschliff und optionale Power-User-Features.

---

## 1. Phase 1: Datenmodell, IPC & System-Resilienz (Backend-Fundament)

| Priorität | Task-ID | Beschreibung / Deliverable | Status |
| --- | --- | --- | --- |
| **P0** | `TASK-01` | **Unified Schema Implementation:** Erstellung des konsolidierten DuckDB-Schemas (`files`-Tabelle inkl. `metadata` JSON-Sparse-Container, Indizes & Sequences). | 🟩 Bereit |
| **P0** | `TASK-02` | **Transaktionales Checkpointing:** Chunk-Infrastruktur (alle 5.000 Items) mit `BEGIN/COMMIT`-Zyklen und Status-Tracking (`processed_files`, `checkpoint_path`) zur Wiederaufnahme abgebrochener Scans. | 🟩 Bereit |
| **P0** | `TASK-03` | **Backpressure / Flow Control:** Implementierung der `BATCH_DATA` / `BATCH_ACK` Signal-Schleife zwischen Worker Thread und Main Process zur Vermeidung von Out-of-Memory-Crashes auf fast NVMe-Speichern. | 🟩 Bereit |
| **P1** | `TASK-04` | **Main Process Progress-Throttler:** Zeitgesteuerte Aggregation von Progress-Events (`INTERVAL_MS = 100` / 10 FPS) an den Renderer Process via IPC. | 🟩 Bereit |

---

## 2. Phase 2: Ingestion-Engine, Hashes & File-Handling

| Priorität | Task-ID | Beschreibung / Deliverable | Status |
| --- | --- | --- | --- |
| **P0** | `TASK-05` | **Dateisystem-Edge-Cases:** Behandlung von Symlinks (`fs.lstat`), Hardlinks (Inode/Dev-Tracking) und Sparse Files zur Vermeidung von Endlosschleifen und Ghost-Duplikaten. | 🟩 Bereit |
| **P1** | `TASK-06` | **3-Stufen-Verifikation:** Implementierung des gestuften Hash-Prozesses (Exakte Größe $\rightarrow$ Fast-Hash $\rightarrow$ Full-Hash via XXH3-128/BLAKE3). | 🟩 Bereit |
| **P1** | `TASK-07` | **Lazy Background Full-Hash Engine:** On-Demand- und Idle-Hashing-Queue für potenziell identische Candidate-Files. | 🟩 Bereit |
| **P1** | `TASK-08` | **Adaptives I/O-Throttling:** Token-Bucket / Delay-Engine mit Voreinstellungen (`TURBO`, `BALANCED`, `BACKGROUND`) zur Vermeidung von Disk Thrashing. | 🟩 Bereit |

---

## 3. Phase 3: Semantik, Notiz-Hierarchie & User-Control

| Priorität | Task-ID | Beschreibung / Deliverable | Status |
| --- | --- | --- | --- |
| **P1** | `TASK-09` | **Notiz-Vererbung & Priorisierung:** Implementierung der SQL Window Function (`ROW_NUMBER()`) zur Auflösung von `FILE` (1) > `FOLDER` (2) > `MEDIA` (3) Hierarchien. | 🟩 Bereit |
| **P2** | `TASK-10` | **False-Positive Handling:** Bereitstellung der `duplicate_exclusions`-Tabelle und UI-Option *"Trotzdem als eindeutig markieren"*. | 🟩 Bereit |
| **P2** | `TASK-11` | **Auto-Pause bei OS-Activity:** Automatisches Herunterschalten des I/O-Throttlers bei aktiven Maus-/Tastatur-Eingaben des Nutzers. | 🔲 Geplant |

---

## 4. Phase 4: Frontend-Integration & Finalisierung (UI/UX)

| Priorität | Task-ID | Beschreibung / Deliverable | Status |
| --- | --- | --- | --- |
| **P1** | `TASK-12` | **Scan-Progress-UI:** Geglätteter Fortschrittsbalken im Renderer Process auf Basis gedrosselter Metrics (10 FPS) ohne Massen-Objekt-Import im React-State. | 🔲 Geplant |
| **P2** | `TASK-13` | **Resume/Retry Dashboard:** UI-Dialoge für den Umgang mit abgebrochenen Scans (Optionen: *"Partiellen Scan verwerfen"* vs. *"Ab Checkpoint fortsetzen"*). | 🔲 Geplant |
| **P2** | `TASK-14` | **Duplikat-Management-UI:** UI für bestätigte Duplikate vs. unbestätigte Kandidaten (inkl. visueller Kennzeichnung von Full-Hash-Status). | 🔲 Geplant |
| **P3** | `TASK-15` | **Multi-Worker Threading:** Dynamischer Worker-Pool für Multi-Socket-Server oder extrem große Netzlaufwerke/NAS-Systeme. | 🔲 Optional |

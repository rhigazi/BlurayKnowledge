---
title: Scan Engine Architecture: Threading, Transactions & Resilience
type: Concept
description: Technische Spezifikation der hochperformanten, thread-basierten Scan-Engine zur Fehlerbehandlung und Datenintegrität.
status: active
created: 2026-09-05T13:15:00Z
timestamp: 2026-09-05T14:00:00Z
---

# Scan Engine Architecture: Threading, Transactions & Resilience

Die Scan-Engine ist die performance-kritischste Komponente der Anwendung. Um auch bei Millionen von Dateien oder defekten Medien (z. B. BD-R mit Kratzern) stabil zu bleiben, basiert sie auf drei Säulen: **Worker Threads**, **Chunk-basiertes Checkpointing** und **Resilience by Design**.

## 1. Threading-Architektur (Off-Main-Thread)

Um das UI nicht zu blockieren, läuft der Scan in einem dedizierten **Node.js Worker Thread**.

* **Main Process:** Verwaltet die `ScannerManager`-Instanz und die IPC-Kommunikation.
* **Worker Thread (`scanWorker.ts`):** Übernimmt das rekursive Filesystem-Scanning, die Berechnung der Fast-Hashes (Header/Footer) und das Batching der Daten.
* **Kommunikation:** Der Worker sendet via `parentPort.postMessage` regelmäßige `BATCH_DATA` Pakete an den Main Process. Um einen Speicherüberlauf (Backpressure) zu vermeiden, implementiert der Main Process ein Acknowledge-System, das dem Worker signalisiert, wann neue Daten gesendet werden können.

## 2. Datenintegrität & Transaktionssicherheit (Chunked Checkpointing)

Um Korruption durch plötzliche Trennung des Mediums oder I/O-Fehler zu vermeiden und gleichzeitig den massiven Datenverlust eines monolithischen Rollbacks zu verhindern, nutzt die Engine ein **inkrementelles Checkpointing-Modell**.

### Das Problem des monolithischen Rollbacks
Bei großen Datenmengen (z. B. NAS, 16 TB HDDs) führt ein einziger Fehler nach Stunden des Scannens bei einer einzigen Transaktion zum Verlust des gesamten Fortschritts.

### Die Lösung: Chunk-basierte Ingestion
Statt erst am Ende des Scans zu committen, teilt die Pipeline den Scan in gekapselte **Chunk-Transaktionen** auf (z. B. alle 5.000 Dateien).

* **Atomarität pro Chunk:** Jede Chunk-Transaktion beinhaltet das `INSERT` der neuen Dateien und das `UPDATE` des Medien-Status (z. B. `processed_files`, `checkpoint_path`).
* **Resilienz:** Bei einem Abbruch bleiben alle bereits erfolgreich committeten Chunks in der Datenbank erhalten. Ein "Scan fortsetzen" ist möglich, indem die Traversierung am letzten bekannten `checkpoint_path` ansetzt und bekannte Pfade überspringt.

## 3. Fast-Hashing Strategie

Statt eine Datei komplett zu lesen, wird zur schnellen Identifikation von Dubletten ein **Fast-Hash** verwendet:
* **Inhalt:** Erste 8 KB + Letzte 8 KB + Dateigröße.
* **Algorithmus:** `xxHash64` oder `BLAKE3` für extreme Geschwindigkeit.
* **Nutzen:** Ermöglicht die Erkennung von Duplikaten (selbst bei Umbenennung) ohne die Performance optischer Medien zu beeinträchtigen. Für kritische Abgleiche kann ein "Deep Scan" mit vollständigem Hash aktiviert werden.

## 4. Fehlerbehandlung (Resilience Matrix)

| Problem | Ursache | Lösung |
| :--- | :--- | :--- |
| **Berechtigungsfehler** (`EACCES`) | Systemordner / geschützte Files | `try-catch` im Walker; Ordner werden übersprungen und als Warnung geloggt. |
| **I/O Fehler / Bad Sectors** | Defekte Medien | Einzelne Datei wird als `READ_ERROR` markiert; der Gesamtscan läuft weiter. |
| **Medium abgezogen** (`ENOENT`) | Physischer Abbruch | Erkennt mehrfache I/O-Fehler $\rightarrow$ bricht Scan ab $\rightarrow$ Letzter erfolgreicher Chunk bleibt in DB erhalten. |
| **Symlink-Schleifen** | Zirkuläre Verweise | `entry.isSymbolicLink()` wird geprüft und ignoriert. |

## timeline
- 2026-09-05T13:15:00Z: Scan-Engine Architektur und Transaktionsmodell definiert.
- 2026-09-05T14:00:00Z: Umstellung von monolithischem Rollback auf Chunked Checkpointing zur Verbesserung der Resilienz bei Großmedien.

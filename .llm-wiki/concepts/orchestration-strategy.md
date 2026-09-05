---
title: Global Resource Governance & Orchestration Strategy
type: Architecture
description: Zentrales Steuerungsmodell zur Auflösung von Ressourcenkonflikten zwischen Ingestion, Verification und I/O-Throttling.
status: active
created: 2026-09-05T15:30:00Z
timestamp: 2026-09-05T15:30:00Z
---

# Global Resource Governance & Orchestration Strategy

## 1. Das Problem: Modulare Isolation vs. System-Interferenz

Die bisherige Architektur basiert auf spezialisierten, isolierten Modulen (Ingestion, Verification, Throttling). Dies führt zu drei kritischen Systemfehlern:
1.  **Resource Contention:** Ingestion und Background-Verification kämpfen unkoordiniert um die gleiche Disk-Bandbreite.
2.  **Priority Inversion:** Ein Hintergrund-Task (Deep Scan) könnte durch ein unkoordiniertes Backpressure-Modul des Ingestion-Workers blockiert werden, oder umgekehrt.
3.  **State Inconsistency:** Ein Abbruch während eines Verifikationszyklus kann zu inkonsistenten Metadaten führen, wenn der Checkpoint nur den Datei-Eintrag, aber nicht den Verifikations-Status sichert.

## 2. Die Lösung: Der "Central System Orchestrator" (CSO)

Wir führen eine zentrale Steuerungsebene ein, die nicht selbst I/O ausführt, aber die **Ressourcen-Quoten (Budgets)** für alle Subsysteme verwaltet.

### 2.1 Der Priority-Graph

Alle Prozesse im System werden nach einer globalen Priorität klassifiziert:

| Priorität | Klassifizierung | Beschreibung | Ressourcen-Zugriff |
| :--- | :--- | :--- | :--- |
| **P0** | **User-Critical** | UI-Reaktion, direkte User-Aktionen (z.B. Löschen, manueller Scan) | Unlimitiert / Höchste Priorität |
| **P1** | **Active Ingestion** | Der aktuell laufende primäre Scan-Vorgang | Hohes I/O-Budget, unterbricht P2/P3 |
| **P2** | **Verification** | Hintergrund-Verifikation von Duplikaten (Deep Scan) | Dynamisches Budget, wird von P1 verdrängt |
| **P3** | **Maintenance** | Datenbank-Cleanup, Index-Optimierung, Metadaten-Sync | Nur bei System-Idle (niedrigste Priorität) |

### 2.2 Unified Resource Budgeting (URB)

Statt dass der `IOThrottler` im Worker nur lokal drosselt, meldet der CSO dem Gesamtsystem das verfügbare **Global I/O Budget** (basierend auf der aktuellen Systemlast und dem aktiven Profil).

*   **Budget-Verteilung:** Wenn ein P1-Task (Ingestion) aktiv ist, erhält dieser 80% des verfügbaren Durchsatzes. Die restlichen 20% werden zwischen P2 und P3 aufgeteilt.
*   **Dynamic Re-Allocation:** Sobald der P1-Task terminiert oder pausiert, wird das Budget sofort auf P2 (Background Verification) umverteilt.

---

## 3. Atomare State-Transitions (Unified Checkpointing)

Um die Integrität zwischen Datei-Daten und Verifikations-Status zu garantieren, wird das Checkpointing-Modell erweitert. Jede Transaktion im `Chunked Checkpointing` muss nun den **Verifikations-Zustand** mit einschließen.

### Das "Atomic Bundle" Prinzip
Ein Commit gilt nur dann als erfolgreich, wenn er folgendes Paket schließt:
`[ File Entry ] + [ Verificaton Status (Unchecked/Potential/Confirmed) ] + [ Checkpoint Metadata ]`

Dies verhindert, dass eine Datei als "Unique" markiert wird, während ihr Duplikat-Partner im nächsten (verlorenen) Chunk lag.

---

## 4. Sparse Metadata Architecture (Memory Optimization)

Um das Problem des wachsenden RAM-Verbrauchs durch massive Indizes zu lösen, implementieren wir eine **Sparse-Metadaten-Strategie**.

*   **Primary Table (`files`):** Enthält nur die absolut notwendigen Daten für den schnellen Scan (ID, Path, Size, Fast-Hash).
*   **Extended Metadata Table (`file_details`):** Enthält die schweren Daten (`full_hash`, `duplicate_state`, `hash_type`).
*   **Vorteil:** Die primäre Such- und Ingestion-Tabelle bleibt klein und effizient im RAM/Cache. Die schweren Verifikations-Daten werden nur bei Bedarf (Join) geladen.

```sql
-- Primäre Tabelle (schlank für schnelles Scanning & Low Memory)
CREATE TABLE files (
    id UUID PRIMARY KEY,
    media_id UUID,
    file_path TEXT,
    file_size_bytes BIGINT,
    fast_hash VARCHAR
);

-- Erweiterte Tabelle (Sparse: nur für Dateien mit Verifikationsbedarf/Duplikaten)
CREATE TABLE file_verification_details (
    file_id UUID REFERENCES files(id),
    full_hash VARCHAR,
    duplicate_state VARCHAR,
    hash_type VARCHAR,
    PRIMARY KEY (file_id)
);
```

---

## 5. Zusammenfassung der System-Garantien

| Zielsetzung | Mechanismus |
| :--- | :--- |
| **Kein Disk Thrashing** | Globales I/O-Budgeting & Adaptive Throttling |
| **Kein Memory Crash** | Sparse Metadata & Backpressure Management |
| **100% Integrität** | Atomic Bundle Checkpointing |
| **Reaktionsschnelligkeit** | P0-Priority für User-Actions (Preemption) |

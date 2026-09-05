---
title: Synchronisation & Medien-Lebenszyklus (Sync & Lifecycle)
type: Concept
description: Strategie für den Umgang mit Offline-Medien, Delta-Scans und der Verifikation von veralteten Daten.
status: active
created: 2026-09-05T13:45:00Z
timestamp: 2026-09-05T13:33:48Z
---

# Synchronisation & Medien-Lebenszyklus

Da die Anwendung für Offline-Medien (DVDs, HDDs) konzipiert ist, kann keine Echtzeit-Synchronisation stattfinden. Das System muss den Zustand zum Zeitpunkt des letzten Scans verwalten und Veraltungen erkennen.

## 1. Medien-Lebenszyklus (`media.lifecycle_status`)

Um den Zustand eines Mediums im Katalog abzubilden, wird ein `lifecycle_status` eingeführt:

| Status | Beschreibung |
| :--- | :--- |
| `ACTIVE` | Das Medium ist Teil des aktiven, verifizierten Katalogs. |
| `UNVERIFIED` | Das Medium wurde lange nicht mehr gesehen (Warn-Indikator). |
| `ARCHIVED` | Physisch nicht mehr vorhanden, aber Daten bleiben für die Historie erhalten. |
| `MISSING` | Bei Routine-Checks nicht im Laufwerk/am System gefunden. |

## 2. Delta-Scan Engine (Performance-Optimierung)

Um Zeitaufwand bei schreibbaren Medien (USB-Sticks, HDDs) zu minimieren, wird ein **Fast Delta-Scan** implementiert. Er vergleicht Dateisystem-Metadaten mit der Datenbank, bevor teure Hashes berechnet werden.

### Ablauf des Delta-Scans:
1.  **Metadaten-Abgleich:** Vergleich von `path`, `file_size` und `mtime` (Modification Time).
2.  **Klassifizierung:**
    *   **Unverändert:** Wenn Pfad, Größe und MTime identisch sind $\rightarrow$ Überspringen.
    *   **Neu/Geändert:** Wenn Pfad fehlt oder MTime/Größe abweichen $\rightarrow$ **Fast-Hash berechnen**.
3.  **Löschung:** Dateien in der DB, die physisch nicht mehr existieren, werden entfernt.

### SQL-Implementierung (Transaktions-Logik)
Der Scan nutzt eine temporäre Staging-Tabelle (`_delta_stage`), um Änderungen atomar zu verarbeiten:
*   `DELETE` von Dateien, die nicht mehr in der Stage sind.
*   `INSERT/UPSERT` von neuen oder geänderten Dateien (basierend auf Hash-Ergebnis).

## 3. Verifikation & UI-Feedback

*   **Quick-Verify:** Beim Einstecken eines Mediums prüft die App die `volume_uuid`. Bei Übereinstimmung wird der `last_verified_at` Zeitstempel aktualisiert.
*   **Veraltungs-Warnung:** Schreibbare Medien erhalten ein Warn-Badge, wenn der letzte Scan $> 30$ Tage zurückliegt.
*   **Immutable-Markierung:** Nicht-beschreibbare Medien (DVD/BD-R) werden als `IMMUTABLE` markiert; hier gibt es keine Veraltungs-Warnungen.

## timeline
- 2026-09-05T13:45:00Z: Sync & Lifecycle Strategie definiert.
EOF
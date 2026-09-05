---
title: Produktkonzept: Zero-Effort-Katalogisierung
type: Note
description: Das funktionale Kernprinzip der App: Automatisches Indizieren mit optionaler manueller Anreicherung.
status: active
created: 2026-09-05T06:53:40Z
timestamp: 2026-09-05T06:54:04Z
---

# Produktkonzept: Zero-Effort-Katalogisierung

## Kernprinzip: "Zero-Effort-First"
Der Fokus der Anwendung liegt auf maximaler automatisierter Erfassung bei minimalem manuellem Aufwand. Das System unterscheidet zwischen **automatisch generierten Daten** (Dateibäume) und **menschlich kuratierten Informationen** (Notizen/Tags).

## Die Drei-Säulen-Datenstruktur (DuckDB Modell)

Das Datenmodell verknüpft drei logische Einheiten:

1.  **Medien-Index (Physisch):**
    *   Speichert, *welcher* Datenträger (Label/ID) *wo* gelagert wird (z. B. "Regal 2, Mappe B").
2.  **Datei-Index (Digital):**
    *   Der automatisch eingelesene Dateibaum (Dateiname, Pfad, Größe, Zeitstempel). Verknüpft mit der Medien-ID.
3.  **Manuelle Anreicherung (Semantisch):**
    *   Freitext-Notizen oder Tags, die an Dateien oder (vorzugsweise) Ordner gebunden werden, um Kontext zu schaffen, der nicht im Dateinamen steht.

## User Journey

### 1. Erfassung (Der Scan)
*   **Aktion:** Medium einlegen $\rightarrow$ Standort in der App angeben $\rightarrow$ Scan starten.
*   **Ergebnis:** Der gesamte Inhalt wird binnen Sekunden in die DuckDB indiziert. Der Nutzer muss keine Metadaten tippen.

### 2. Suche (Der Nutzwert)
*   **Ebene 1 (Automatisch):** Suche nach Dateinamen oder Pfaden (z. B. "Rechnung").
*   **Ebene 2 (Semantisch):** Suche nach hinterlegten Notizen (z. B. "Toskana"), auch wenn der Begriff nicht im Dateinamen vorkommt.
*   **Ergebnis:** Sofortige Anzeige von:
    *   Dateipfad auf dem Medium.
    *   Name des Datenträgers.
    *   **Physischer Standort** des Mediums.

### 3. Verfeinerung (Optional)
*   Nutzer können gezielt Ordner mit Kontext versehen (z. B. *"Fotos von der Taufe"*), um die Auffindbarkeit für die Zukunft zu erhöhen.

## Vorteile gegenüber klassischer Dateiverwaltung
*   **Keine manuelle Pflege:** Man muss nicht jede Datei benennen; die App indiziert das Bestehende.
*   **Physische Brücke:** Die App schließt die Lücke zwischen dem digitalen Pfad und dem realen Standort im Regal.
*   **Kontext-Suche:** Durch die Notiz-Funktion wird die Suche "intelligent", ohne dass eine KI-gestützte Analyse der Bildinhalte nötig ist.

## timeline
- 2026-09-05T06:53:40Z: Definition des Zero-Effort-Kernkonzepts und Datenmodells.
EOF
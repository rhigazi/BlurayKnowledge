---
title: UI/UX-Konzept & Navigationsfluss
type: Concept
description: Design-Spezifikation der Single Page Application (SPA) Oberfläche, Layout-Struktur und Benutzerführung.
status: active
created: 2026-09-05T12:57:23Z
timestamp: 2026-09-05T12:57:54Z
---

# UI/UX-Konzept & Navigationsfluss

Das Design der **Single Page Application (SPA)** ist auf maximale Übersichtlichkeit, schnelle Reaktionszeiten und Tastaturfreundlichkeit ausgelegt. Da alle Daten lokal in DuckDB gehalten werden, erfolgen Ansichtswechsel und Suchanfragen ohne Ladezeiten.

Die Anwendung gliedert sich in ein **festes Anwendungs-Layout** mit Hauptnavigation und **vier zentrale Bildschirmansichten**.

## 1. Das globale Anwendungs-Layout

Das Fenster ist in drei feste Bereiche unterteilt, die immer sichtbar bleiben:

* **Sidebar (Links):** Schlanke Navigationsleiste zum Umschalten zwischen den Hauptansichten (`Suche`, `Scan`, `Medien`, `Einstellungen`).
* **Header (Oben):** Globale Schnellsuche (immer erreichbar) und System-Statusanzeige.
* **Footer (Unten):** Gesamtstatistik des Katalogs (Anzahl Medien, Dateien, Speicherplatz).

## 2. Die 4 Haupt-Bildschirmansichten

### Ansicht 1: Katalogsuche & Dateimanager (Hauptansicht)
Kombiniert Suchergebnisse mit der Detailansicht für Fundorte und Notizen.
* **Linke Spalte (Ergebnisliste):** Zeigt Ordner, Dateien und Archive mit Such-Highlighting und einem Duplikat-Indikator an.
* **Rechte Spalte (Detail- & Editierbereich):** Zeigt den genauen physischen Standort des Mediums groß an, bietet einen Notiz-Editor und listet alle identischen Duplikate auf.

### Ansicht 2: Datenträger Erfassen (Scan-Center)
Ein geführter 3-Schritt-Assistent zum Einlesen neuer Medien (BD-R, DVD, USB).
* **Sicherheit:** Automatische Erkennung der Volume-UUID zur Vermeidung von Doppeleinlesungen.
* **Echtzeit-Feedback:** Anzeige von Scan-Geschwindigkeit, Dateianzahl und Fortschrittsbalken.

### Ansicht 3: Medien & Standort-Verwaltung
Hierarchie-Ansicht zur Strukturierung der physischen Lagerorte.
* **Funktionalität:** Ermöglicht das einfache Umsortieren ganzer Lagerbereiche (z. B. "Regal 1" nach "Regal 2"), wodurch alle zugeordneten Medien automatisch mitwandern.

### Ansicht 4: Duplikate & Speicheranalyse
Übersicht zur Identifikation von redundanten Backups basierend auf Checksummen.
* **Nutzen:** Anzeige des "verschwendeten" Speicherplatzes und Auflistung der Fundorte identischer Dateien.

## 3. Der Navigationsfluss (User Flow)

Die Anwendung ist darauf optimiert, dass der Nutzer die Hauptansicht (Suche) fast nie verlassen muss.

1.  **Haupt-Workflow (Finden):** Suche $\rightarrow$ Filterung in Echtzeit $\rightarrow$ Klick auf Ergebnis $\rightarrow$ Standort ablesen.
2.  **Erfassungs-Workflow (Neuaufnahme):** `[+ Scan]` $\rightarrow$ Medium & Standort wählen $\rightarrow$ Scan $\rightarrow$ Automatischer Rücksprung zur Suche.
3.  **Pflege-Workflow (Notizen):** Suche $\rightarrow$ Rechts im Notizfeld tippen $\rightarrow$ Sofortige automatische Speicherung.

## 4. Key UX-Features

* **Globaler Shortcut (`Ctrl+F` / `Cmd+F`):** Sofortiger Sprung in das Suchfeld aus jeder Ansicht.
* **Breadcrumb-Pfade:** Anklickbare Pfade zur schnellen Navigation durch die Ordnerstruktur.
* **Fuzzy Search:** Fehlertolerante Suche (z. B. findet "Toskna" $\rightarrow$ "Toskana").

## timeline
- 2026-09-05T12:57:23Z: UI/UX-Konzept und Navigationsfluss definiert.
EOF
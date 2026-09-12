# Task 14: Duplikate & Speicheranalyse

## Ziel
Baue die Duplikat Analyse Ansicht mit Uebersicht aller Kandidaten, bestaetigten Duplikaten und bereinigtem Speicherplatz. Unterstuetze Deep Scan, Exclusion und Bulk Aktionen.

## Schritte

### 14.1 DuplicatesView Komponente
Schreibe `src/renderer/views/DuplicatesView.tsx`:

```tsx
// Drei Bereiche:
// +------------------------------------------+
// | Zusammenfassung (Kacheln oben)             |
// | +----------+ +----------+ +----------+    |
// | | 128  Dup | | 45.2 GB  | | 12  Conf |    |
// | | Kandidat | | Verschw. | | Bestaet. |    |
// | +----------+ +----------+ +----------+    |
// +------------------------------------------+
// | Gruppe 1: report_2023.pdf (4x, 12 MB)    |
// |   [Details] [Ausschliessen] [Deep Scan]   |
// | Gruppe 2: backup.zip (2x, 2.4 GB)         |
// |   [Details] [CONFIRMED] [Loeschen]         |
// | ...                                        |
// +------------------------------------------+
```

- Oben: Statistik Kacheln (Anzahl Kandidaten, verschwendeter Speicher, bestaetigte Duplikate)
- Hauptbereich: Gruppierte Duplikat Listen
- Jede Gruppe: Fast-Hash / Full-Hash, Anzahl Dateien, Gesamtgroesse, Status

### 14.2 Duplikat Statistik Laden
```typescript
async function loadDuplicateStats(): Promise<DuplicateStats> {
  const result = await window.api.getSystemStats();
  // Berechne aus DB:
  // - Anzahl POTENTIAL_DUPLICATE Dateien
  // - Anzahl CONFIRMED_DUPLICATE Dateien
  // - Summe des "verschwendeten" Speichers (file_size * (count - 1))
  // - Nach Typ (Groesste Duplikate zuerst)
}
```

### 14.3 Duplikat Gruppen Liste
Schreibe `src/renderer/components/DuplicateGroupList.tsx`:

```tsx
interface DuplicateGroup {
  hash: string;           // Fast-Hash oder Full-Hash
  hashType: 'FAST' | 'FULL';
  fileCount: number;
  totalWastedBytes: number;
  files: DuplicateFileEntry[];
  status: 'POTENTIAL' | 'CONFIRMED';
}
```

**Jede Gruppe zeigt:**
- Hash Wert (verkuerzt sichtbar)
- Anzahl Dateien `(x4)`
- Gesamte verschwendete Groesse
- Status Badge (Gelb/Rot)
- Liste der einzelnen Dateien mit Pfad, Groesse, Medium, Standort

**Aktionen pro Gruppe:**
- `[Alle Deep Scannen]` (wenn POTENTIAL): Starte Full-Hash fuer alle Dateien der Gruppe
- `[Als bestaetigt markieren]` (nach Deep Scan)
- `[Alle als eindeutig markieren]` -> Batch Exclusion
- `[Alle ausser einer loeschen]` (nur wenn CONFIRMED, mit Schutzmechanismus)

### 14.4 Einzel Datei Aktionen
- `[Im Dateisystem oeffnen]` Button (via `shell.showItemInFolder`)
- `[Aus dieser Gruppe ausschliessen]` -> Exclusion Eintrag
- `[Details anzeigen]` -> Oeffnet SearchView Detail Panel

### 14.5 Bulk Deep Scan
- Button `[Alle Kandidaten deep-scannen]` in der Kopfzeile
- Zeige Progress: "Verifiziere 45 von 128 Kandidaten..."
- Nutze `DeepScanEngine.enqueueCandidates()` und zeige Fortschritt
- Nach Abschluss: Aktualisiere die Ansicht automatisch

### 14.6 Speicheranalyse (Pie Chart oder Balken)
- Zeige Top 10 der groessten Duplikat Gruppen
- Visualisierung: Horizontaler Balken pro Gruppe
- `report_2023.pdf (4x) [============            ] 12.4 MB`
- `backup.zip (2x) [==========================] 2.4 GB`

### 14.7 Batch Aktionen mit Sicherheitsdialog
```tsx
// "Loeschen" von Duplikaten ist eine zerstoerende Aktion:
// Zeige vorher:
// +------------------------------------------+
// | ⚠️ Sicherheitswarnung                      |
// |                                           |
// | Du willst 3 Dateien loeschen:              |
// | - backup_v1.zip (Medium: HDD_01)           |
// | - backup_v2.zip (Medium: HDD_02)           |
// | - backup_v3.zip (Medium: HDD_03)           |
// |                                           |
// | Nur Dateien mit CONFIRMED Status loeschbar.|
// | Erklaere: Gelöschte Dateien koennen nicht  |
// | wiederhergestellt werden (Offline-Medien). |
// |                                           |
// | [Abbrechen] [Ausgewaehlte loeschen]         |
// +------------------------------------------+
```

- Loeschaktionen sind blockiert bis Deep Scan CONFIRMED
- Jede Loeschung erfordert zweifache Bestaetigung (Dialog + Button)
- Loeschung loescht den DB Eintrag (die physische Datei bleibt auf dem Medium)

### 14.8 Sortierung und Filter Optionen
- Sortierung: Nach Groesse (groesste zuerst), nach Anzahl, nach Datum
- Filter: Nur POTENTIAL, Nur CONFIRMED, Beide
- Suchfeld innerhalb der Ansicht: Filtere Gruppen nach Dateinamen

## Akzeptanzkriterien
- [ ] Statistik Kacheln zeigen korrekte Werte (Anzahl, Speicher, Status)
- [ ] Duplikat Gruppen sind nach Fast-Hash/Groesse gruppiert
- [ ] Status (POTENTIAL/CONFIRMED) wird farblich korrekt angezeigt
- [ ] Bulk Deep Scan startet fuer alle Kandidaten mit Progress
- [ ] Exclusion (Kein Duplikat) funktioniert pro Datei oder Gruppe
- [ ] Batch Exclusion fuer komplette Gruppe funktioniert
- [ ] Loesch Dialog erscheint nur bei CONFIRMED Status
- [ ] Top 10 Speicherverschwendung Visualisierung
- [ ] Filter und Sortierung beeinflussen die Anzeige korrekt
- [ ] Nach Deep Scan aktualisiert sich der Status automatisch

## Abgrenzung
- Physisches Loeschen von Dateien ist NICHT implementiert (Offline Medien!)
- Keine automatische Bereinigung (nur Markierung/Exclusion)
- Bulk Deep Scan nutzt die Background Queue aus Task 7
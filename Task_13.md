# Task 13: Medien & Standort Verwaltung

## Ziel
Baue die Medien Verwaltungsansicht fuer die Uebersicht, Bearbeitung und Strukturierung aller Datentraeger. Unterstuetze das Umsortieren von physischen Standorten und die Lebenszyklus Verwaltung.

## Schritte

### 13.1 MediaView Komponente
Schreibe `src/renderer/views/MediaView.tsx`:

- Obere Leiste: `[Medium hinzufuegen]` Button
- Hauptbereich: Tabelle/Liste aller Medien mit Such/Filter Funktion
- Klick auf Medium oeffnet Detail Ansicht

### 13.2 Medien Tabelle
Schreibe `src/renderer/components/MediaTable.tsx`:

```tsx
export function MediaTable() {
  const [mediaList, setMediaList] = useState<MediaEntry[]>([]);
  const [sortBy, setSortBy] = useState<'label' | 'type' | 'scannedAt' | 'fileCount'>('scannedAt');

  useEffect(() => {
    window.api.listMedia().then(setMediaList);
  }, []);
}
```

**Tabellen Spalten:**
| Spalte | Beschreibung |
|--------|-------------|
| Volume Label | Name des Mediums (klickbar) |
| Typ | BD-R, DVD, USB-HDD (mit Icon) |
| Standort | Physischer Ort (editierbar) |
| Status | Lifecycle Status mit farbiger Kennzeichnung |
| Dateien | Anzahl indizierter Dateien |
| Groesse | Gesamtgroesse (formatierte Bytes) |
| Scan Datum | Letzter Scan Zeitstempel |
| Aktionen | Edit, Archive, Delete Buttons |

**Sortierung:** Klick auf Spaltenkopf sortiert die Tabelle

### 13.3 Media Detail & Edit Dialog
Schreibe `src/renderer/components/MediaDetailDialog.tsx`:

```tsx
export function MediaDetailDialog({ mediaId, onClose }: MediaDetailDialogProps) {
  const [media, setMedia] = useState<MediaEntry | null>(null);
  // Lade Media + zugehoerige Datei Statistik
  // Formular zum Editieren aller Felder
}
```

**Anzeige:**
- Volume Label, Volume UUID, Media Type, Storage Location
- Lifecycle Status (mit Button zum Aendern)
- Scan Historie (wann wurde gescannt, wie viele Dateien)
- Liste der letzten 20 indizierten Dateien

**Editierfunktionen:**
- Storage Location aendern (z.B. "Regal 1" -> "Regal 3")
- Media Type aendern
- Lifecycle Status manuell setzen:
  - `[Als vermisst markieren]` -> MISSING
  - `[Archivieren]` -> ARCHIVED (Daten bleiben erhalten)
  - `[Reaktivieren]` -> ACTIVE
- `[Medium loeschen]` mit Bestaetigungsdialog (loescht ALLE zugehoerigen Daten)

### 13.4 Standort Massen Update
- Bulk Operation: Waehle mehrere Medien aus -> `[Standort aendern]`
- Dialog: "Neuen Standort eingeben" -> Update aller selektierten Medien
- Beispiel: Alle Medien von "Regal 1" nach "Regal 2" verschieben

### 13.5 Lebenszyklus Filter und Sortierung
- Filter Status: Alle / Aktiv / Unverified / Archiviert / Vermisst
- Sortierung: Nach Scan Datum (neueste zuerst), nach Label, nach Standort
- Visuelle Marker:
  - 🔴 UNVERIFIED (aelter als 30 Tage)
  - ⚫ ARCHIVED (durchgestrichen, ausgegraut)
  - 🟡 MISSING (Warnfarbe)
  - 🟢 ACTIVE (normal)

### 13.6 Medien Statistik pro Eintrag
- Anzahl Dateien auf dem Medium
- Gesamtgroesse aller Dateien
- Anzahl Duplikat Kandidaten auf diesem Medium
- Letzter Scan Zeitstempel
- Durchschnittliche Dateigroesse
- Anzahl Ordner vs Dateien

### 13.7 Neues Medium anlegen (Formular)
Schreibe `src/renderer/components/CreateMediaForm.tsx`:

```tsx
export function CreateMediaForm({ onCreated }: CreateMediaFormProps) {
  const [label, setLabel] = useState('');
  const [type, setType] = useState<'BD-R' | 'DVD' | 'USB-HDD'>('DVD');
  const [location, setLocation] = useState('');
  // Volume UUID wird automatisch generiert
  // Nach Erstellung: Optional direkt zum Scan weiterleiten
}
```

## Akzeptanzkriterien
- [ ] Medien Tabelle zeigt alle Eintraege mit Spalten Sortierung
- [ ] Klick auf Medium oeffnet Detail Dialog mit allen Daten
- [ ] Storage Location kann editiert werden
- [ ] Lifecycle Status kann manuell geaendert werden
- [ ] Bulk Standort Update funktioniert fuer mehrere Medien
- [ ] Filter (Aktiv/Unverified/Archiviert/Vermisst) funktioniert
- [ ] Neues Medium kann angelegt werden
- [ ] Nach Erstellung wird Direktlink zum Scan angeboten
- [ ] Medien Loeschung erfordert Bestaetigung und loescht alle abhaengigen Daten
- [ ] Die Ansicht aktualisiert sich nach Aenderungen

## Abgrenzung
- Scan Funktionalitaet ist in Task 12 (hier nur Verknuepfung)
- Duplikat Analyse kommt in Task 14
- Keine Batch Import/Export von Medien
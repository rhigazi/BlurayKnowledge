# Task 11: Katalog Suche & Detail Ansicht (Hauptansicht)

## Ziel
Baue die Hauptansicht der Anwendung: eine zweispaltige Ansicht mit Suchergebnissen links und einem Detail/Editier Bereich rechts. Dies ist die View die der User 90% der Zeit sieht.

## Schritte

### 11.1 SearchView Komponente
Schreibe `src/renderer/views/SearchView.tsx`:

```tsx
// Zweispaltiges Layout:
// +-------------------------------------+
// | Ergebnisliste (links) | Detail (rechts) |
// |   - Datei/Ordner Eintraege          |   - Datei Info
// |   - Highlighting                    |   - Pfad Anzeige
// |   - Duplikat Icon                   |   - Medien Standort
// |   - Klick -> Detail                 |   - Notiz Editor
// |                                     |   - Duplikat Liste
// +-------------------------------------+
```

- Linke Spalte (45% Breite): Ergebnisliste, scrollbar
- Rechte Spalte (55% Breite): Detailansicht des selektierten Eintrags
- Beide Spalten teilen sich die globale Query aus dem Header

### 11.2 Ergebnisliste
Schreibe `src/renderer/components/ResultList.tsx`:

```tsx
interface ResultListProps {
  results: SearchResult[];
  selectedId: string | null;
  onSelect: (result: SearchResult) => void;
}
```

- Zeige pro Eintrag: Icon (Datei/Ordner), Dateiname (fett), Pfad (klein), Groesse, Duplikat Status Icon
- Duplikat Status Farben: Grau (UNCHECKED), Gruen (UNIQUE), Gelb (POTENTIAL), Rot (CONFIRMED)
- Suchbegriff Highlighting im Dateinamen und Pfad
- Virtuelles Scrolling fuer 10.000+ Ergebnisse (react-window oder einfaches IntersectionObserver)
- Klick auf Eintrag selektiert ihn fuer die Detail Ansicht

### 11.3 Such Highlighting
Schreibe `src/renderer/components/SearchHighlight.tsx`:

```tsx
export function SearchHighlight({ text, query }: { text: string; query: string }) {
  if (!query || query.length < 2) return <>{text}</>;
  const parts = text.split(new RegExp(`(${escapeRegex(query)})`, 'gi'));
  return (
    <span>
      {parts.map((part, i) =>
        part.toLowerCase() === query.toLowerCase()
          ? <mark key={i}>{part}</mark>
          : part
      )}
    </span>
  );
}
```

### 11.4 Detail Ansicht
Schreibe `src/renderer/components/DetailPanel.tsx`:

**Abschnitt 1: Datei Information**
- Dateiname, Pfad, Groesse (formatierte Bytes)
- Erstellungsdatum, Aenderungsdatum, Scan Datum
- Dateiendung, Media Typ

**Abschnitt 2: Medien Standort (Prominent!)**
- Volume Label
- Media Type (BD-R, DVD, USB-HDD)
- Storage Location (gross und fett)
- `[Im Dateisystem oeffnen]` Button

**Abschnitt 3: Duplikat Info**
- Wenn duplicate_state != UNIQUE: Liste der identischen Dateien
- Gelbe/Rote Status Anzeige
- `[Deep Scan starten]` Button (wenn POTENTIAL)
- `[Als Duplikat bestaetigen]` Button (wenn CONFIRMED)
- `[Kein Duplikat]` Button (False-Positive Exclusion)

**Abschnitt 4: Notiz Editor**
```tsx
export function NoteEditor({ fileId }: { fileId: string }) {
  const [notes, setNotes] = useState<NoteEntry[]>([]);
  const [editingText, setEditingText] = useState('');

  // Lade Notizen (auch geerbte)
  useEffect(() => {
    window.api.getNotesForFile(fileId).then(setNotes);
  }, [fileId]);

  // Auto-Save bei 2 Sekunden Inaktivitaet
  // ... debounced save
}
```

- Zeigt effektive Notiz (inklusive Vererbung)
- Inline Editierfeld mit Auto-Save (Debounce 2s)
- Anzeige ob Notiz von FILE, FOLDER oder MEDIA geerbt wurde

### 11.5 Breadcrumb Navigation
Schreibe `src/renderer/components/BreadcrumbNav.tsx`:

```tsx
// Zeigt den Datei Pfad als klickbare Brotkrumen:
// /Dokumente > /Steuer > /2024
// Klick auf einen Teil -> Zeige Ordner Inhalt
```

- Zerlege `file_path` in einzelne Segmente
- Jedes Segment ist ein klickbarer Link
- Letztes Segment ist der aktuelle Dateiname (nicht klickbar)

### 11.6 Keyboard Navigation
- Pfeiltasten hoch/runter: Durch Ergebnisse navigieren
- Enter: Ausgewaehltes Ergebnis oeffnen
- Escape: Suchfeld leeren / Auswahl aufheben
- Tab/Shift+Tab: Zwischen Ergebnisliste und Detail wechseln

### 11.7 Filter Leiste
Schreibe `src/renderer/components/FilterBar.tsx`:

```tsx
interface SearchFilters {
  mediaId?: number;
  extension?: string;
  minSize?: number;
  maxSize?: number;
  duplicateState?: string;
  dateFrom?: string;
  dateTo?: string;
}
```

- Zusammenklappbare Filter Leiste unter der Suche
- Filter: Dateityp (Datei/Ordner), Groesse (Slider/Min/Max), Datum (Range), Duplikat Status
- Medien Auswahl (nur von bestimmtem Medium)
- `[Filter zuruecksetzen]` Button

## Akzeptanzkriterien
- [ ] Suche liefert Ergebnisse aus DuckDB und zeigt sie in der Liste an
- [ ] Suchbegriff Highlighting funktioniert in Ergebnisliste
- [ ] Klick auf Ergebnis zeigt Detail Ansicht mit allen Informationen
- [ ] Medien Standort wird prominent angezeigt
- [ ] Duplikat Status wird farblich korrekt dargestellt
- [ ] Notizen koennen inline bearbeitet und gespeichert werden (Auto-Save)
- [ ] Breadcrumb Navigation funktioniert
- [ ] Tastatur Navigation (Pfeiltasten, Enter, Escape) funktioniert
- [ ] Filter koennen gesetzt und zurueckgesetzt werden
- [ ] Detail Ansicht zeigt geerbte Notizen mit Quellenangabe

## Abgrenzung
- Scan Funktionalitaet ist noch nicht in dieser View
- Duplikat Management hat eigene View (Task 15)
- Kein Drag-and-Drop oder Dateimanipulation
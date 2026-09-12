# Task 12: Scan Center View (3-Schritt Assistent)

## Ziel
Baue die Scan Ansicht als gefuehrten 3-Schritt Assistenten: Medium Auswahl / Erstellung, Scan Konfiguration, Scan Durchfuehrung mit Echtzeit Fortschritt.

## Schritte

### 12.1 ScanView Komponente
Schreibe `src/renderer/views/ScanView.tsx`:

```tsx
export function ScanView() {
  const [step, setStep] = useState(1); // 1, 2, oder 3
  // Schritt 1: Medium Auswahl
  // Schritt 2: Profil Konfiguration
  // Schritt 3: Scan Durchfuehrung mit Progress
}
```

- Dreistufiger Assistent mit sichtbarem Step Indicator oben
- Rueckwaerts Navigation erlaubt (Schritt 2 -> Schritt 1)
- Schritt 3: Waehrend des Scans keine Navigation moeglich (nur Cancel)

### 12.2 Schritt 1: Medium Auswahl
Schreibe `src/renderer/components/ScanStepMedia.tsx`:

```tsx
export function ScanStepMedia({ onSelect, initialMedia }: ScanStepMediaProps) {
  // Option A: Bestehendes Medium auswaehlen (Volume UUID Erkennung)
  // Option B: Neues Medium anlegen (Label, Typ, Standort eingeben)
}
```

**Option A: Bestehendes Medium scannen**
- Dropdown/Liste aller vorhandenen Medien
- Volume UUID Check: Wenn das eingelegte Medium bereits in der DB ist, zeige Quick-Verify an
- Bei Quick-Verify: `[Nur Lebenszyklus aktualisieren]` oder `[Erneut voll scannen]`

**Option B: Neues Medium anlegen**
- Formula: Volume Label (Pflicht), Media Type (BD-R/DVD/USB-HDD), Storage Location (Freitext)
- Volume UUID wird automatisch generiert (wenn vom OS bereitgestellt, nutze diesen)
- `[Weiter]` Button nur wenn alle Pflichtfelder ausgefuellt

### 12.3 Schritt 2: Scan Profil
Schreibe `src/renderer/components/ScanStepProfile.tsx`:

```tsx
export function ScanStepProfile({ onStart, selectedMedia }: ScanStepProfileProps) {
  const [profile, setProfile] = useState<ScanProfile>('BALANCED');
  const [rootPath, setRootPath] = useState('');
  const [enableDeepScan, setEnableDeepScan] = useState(false);
}
```

**Scan Profil Auswahl (Radio Buttons):**
- Hintergrund (30 MB/s): Minimaler Einfluss, stört nie
- Ausgewogen (150 MB/s): Empfohlen für normales Arbeiten (Default)
- Turbo (Max Speed): Volle Leistung, System wird träge

**Weitere Optionen:**
- Laufwerk Pfad: Textfeld oder OS Dialog zum Auswaehlen
- Deep Scan nach dem Scan aktivieren (automatische Full-Hash Queue)
- `[Scan starten]` Button -> wechselt zu Schritt 3

### 12.4 Schritt 3: Scan Progress
Schreibe `src/renderer/components/ScanProgressView.tsx`:

```tsx
export function ScanProgressView({ mediaId, profile }: ScanProgressViewProps) {
  const [progress, setProgress] = useState<ScanProgress>({
    scannedCount: 0,
    currentFile: '',
    speed: 0,         // Dateien/sec
    estimatedRemaining: 0, // Sekunden
    startedAt: Date.now(),
  });

  useEffect(() => {
    const unsubscribe = window.api.onScanProgress(setProgress);
    window.api.startScan(mediaId, rootPath, profile);
    return () => unsubscribe();
  }, []);

  return (
    <div className="scan-progress">
      <div className="progress-circle">{/* Prozent Anzeige */}</div>
      <div className="progress-details">
        <span>{progress.scannedCount.toLocaleString()} Dateien erfasst</span>
        <span>{formatSpeed(progress.speed)}</span>
        <span>{formatETA(progress.estimatedRemaining)} verbleibend</span>
      </div>
      <div className="current-file">{progress.currentFile}</div>
      <ProgressBar percent={/* ... */} />
      <button className="cancel-button">Scan abbrechen</button>
    </div>
  );
}
```

### 12.5 OS Laufwerk Auswahl Dialog
- Nutze Electron `dialog.showOpenDialog` fuer Laufwerk Auswahl
- Oder: Textfeld mit Pfad Eingabe und `[Browse]` Button
- Platziere diese Option im Schritt 2 Konfiguration
- Stelle sicher dass nur gelesen werden kann (kein Schreibzugriff noetig)

### 12.6 Volume UUID Erkennung
- Lese Volume UUID aus dem Dateisystem (plattformspezifisch)
- Windows: `fs.statSync(rootPath).dev` oder WMI
- Linux/macOS: `stat -f` oder `/proc/self/mountinfo`
- Fallback: Generiere eine UUID basierend auf Laufwerkspfad + Seriennummer
- Speichere in `media.volume_uuid` fuer Quick-Verify

### 12.7 Scan Zusammenfassung nach Abschluss
```tsx
// Nach erfolgreichem Scan:
// +----------------------------------+
// | ✅ Scan abgeschlossen              |
// |                                   |
// | 42.531 Dateien erfasst             |
// | 12 Duplikat Kandidaten gefunden   |
// | 8 Medien im Katalog               |
// |                                   |
// | [Zur Suche] [Duplikate ansehen]    |
// +----------------------------------+
```

- Zeige Zusammenfassung nach Scan Ende
- Links: Zur Suche (Hauptansicht) und zu Duplikaten (wenn welche gefunden)
- Bei Fehlern: Zeige Fehler Zusammenfassung (z.B. "3 Dateien konnten nicht gelesen werden")

### 12.8 Fehler Behandlung waerend Scan
- Bei I/O Fehler: Zeige Warnung aber setze Scan fort
- Bei schwerem Fehler: Breche ab, zeige "Scan fehlgeschlagen" mit Details
- Abbruch via Button: Bestaetigungsdialog "Scan wirklich abbrechen? Bereits erfasste Daten bleiben erhalten."

## Akzeptanzkriterien
- [ ] 3-Schritt Assistent fuehrt durch den Scan Prozess
- [ ] Bestehendes Medium kann ausgewaehlt oder neues angelegt werden
- [ ] Scan Profil (TURBO/BALANCED/BACKGROUND) ist waehlbar
- [ ] Echtzeit Progress (Dateien, Speed, ETA) wird angezeigt
- [ ] Fortschrittsbalken zeigt visuellen Status
- [ ] Scan kann abgebrochen werden, bereits erfasste Daten bleiben
- [ ] Nach Scan Abschluss wird Zusammenfassung angezeigt
- [ ] Volume UUID wird erkannt (Quick-Verify Angebot bei bekanntem Medium)
- [ ] Bei I/O Fehlern laeuft der Scan weiter (Datei wird uebersprungen)
- [ ] Laufwerk Pfad kann via OS Dialog gewaehlt werden

## Abgrenzung
- Resume/Retry Dialog kommt in Task 13 (erweiterte Fehlerbehandlung)
- Auto-Detection User Activity kommt separat
- Deep Scan Queue startet automatisch im Hintergrund (Task 7 Logik)
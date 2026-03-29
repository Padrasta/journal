# Journal — Technische Dokumentation

**Stand:** März 2026
**Gebaut von:** Ingrid (Padrasta) & Claude 4.6
**Repo:** [github.com/Padrasta/journal](https://github.com/Padrasta/journal)
**Live:** [padrasta.github.io/journal](https://padrasta.github.io/journal)

---

## Was ist das hier?

Ein digitales Tagebuch das wie ein physisches Notizbuch funktioniert. Keine Datenbank, kein Backend, kein Login-Zwang. Alles läuft als eine einzige HTML-Datei im Browser — die Daten werden als JSON-Dateien gespeichert, wahlweise lokal oder in Google Drive.

---

## Dateistruktur

```
journal/
├── index.html          ← Die gesamte App (HTML + CSS + JavaScript)
├── bohne.html          ← Standalone Mini-App für den Handy-Homescreen
├── manifest.json       ← PWA-Manifest für die Haupt-App
├── bohne-manifest.json ← PWA-Manifest für die Bohnen-Mini-App
└── img/
    └── Bohne.svg       ← Kidney-Bean SVG (handgezeichnet, mit Schatten + Glanz)
```

Alles funktioniert **ohne Server** — einfach `index.html` öffnen reicht.

---

## Die Canvas-Architektur

Das Herz der App ist ein fester **750 × 1060 px Canvas** (`#flaeche`), der per CSS `transform: scale()` auf die verfügbare Bildschirmgröße skaliert wird. Die Funktion `canvasAnpassen()` berechnet bei jedem Resize den richtigen Scale-Faktor:

```javascript
canvasScale = Math.min(wBreite / CANVAS_BREITE, wHoehe / CANVAS_HOEHE, 1);
document.getElementById('flaeche').style.transform = `scale(${canvasScale})`;
```

Weil der Canvas immer in Originalkoordinaten (750×1060) arbeitet, müssen alle Maus-/Touch-Positionen beim Ziehen durch `canvasScale` dividiert werden:

```javascript
const deltaX = (e.clientX - mausStartX) / canvasScale;
```

---

## Elemente auf der Canvas

### Polaroids
- Bild + weißer Rahmen + handschriftliche Beschriftung unten
- Ziehbar, drehbar (Griff oben rechts), resizable (Griff unten rechts)
- Bild wird als Base64-DataURL im JSON gespeichert

### Klebezettel
- Vier Farben (Gelb, Rosa, Mint, Blau)
- `contenteditable`-Textbereich mit Caveat-Handschrift-Font
- Drag-Griff (`· · ·`) oben für zuverlässiges Ziehen, besonders auf Mobile

### Stempel
- SVG-Grafiken die auf die Canvas platziert werden
- Drehbar und skalierbar

### Text-Elemente
- Frei positionierbarer Text mit wählbarer Schriftart und -größe
- Schmaler Drag-Streifen oben

---

## Speichern & Laden (JSON-Format)

Beim Speichern serialisiert `seitenStandSerializieren()` die gesamte Canvas in ein JSON-Objekt:

```json
{
  "version": "1.0",
  "datum": "Dienstag, 24. März 2026",
  "tagessatz": "Das Leben ist schön.",
  "hintergrund": "bg-pergament",
  "reflexion": "Heute war ein guter Tag.",
  "canvasBreite": 750,
  "canvasHoehe": 1060,
  "bohnen": 3,
  "elemente": [
    {
      "typ": "klebezettel",
      "left": "120px", "top": "80px",
      "drehung": "-4.2",
      "farbe": "farbe-gelb",
      "text": "Kaffee nicht vergessen!"
    },
    {
      "typ": "polaroid",
      "left": "300px", "top": "200px",
      "breite": "200px",
      "drehung": "6.1",
      "bildUrl": "data:image/jpeg;base64,..."
    }
  ]
}
```

Beim Laden geht `seitenStandWiederherstellen()` durch das Array und baut jedes Element neu auf.

---

## Google Drive Integration

Die App nutzt die **Google Identity Services (GIS)** Bibliothek für OAuth — kein klassischer Login, sondern ein Access Token der direkt im Browser-Speicher bleibt (kein Server sieht ihn).

```
Nutzer klickt ☁️  →  Google-Popup  →  Access Token
                                           ↓
                              driveOrdnerVorbereiten()
                              erstellt: Journal/Aktuell/
                                        Journal/Versiegelt/
                                           ↓
                              💾 Speichern → driveHochladen()
                              📂 Laden    → driveHeuteLaden()
```

Die Ordnerstruktur in Drive:
```
Google Drive/
└── Journal/
    ├── Aktuell/        ← aktuelle, offene Seiten
    └── Versiegelt/     ← abgeschlossene, gesperrte Seiten
```

---

## Bohnenzähler (Bohnenhaufen)

Inspiriert von der pädagogischen Kaffeebohnen-Übung: pro schönem Moment des Tages eine Bohne. Am Abend sieht man wie viele gute Dinge passiert sind.

### Visueller Haufen-Algorithmus

Die Bohnen werden nicht zufällig verstreut, sondern in einem **Pyramidenmuster** gestapelt:

```javascript
function buildRows(total) {
  // Breiteste Reihe unten, jede weitere ~28% schmäler
  const base = Math.ceil(Math.sqrt(total * 2));
  const rows = [];
  let w = base;
  while (remaining > 0) {
    rows.push(Math.min(remaining, w));
    w = Math.floor(w * 0.72);
  }
  return rows;
}
```

Jede Bohne bekommt stabile Pseudo-Zufallswerte über `seededRand(index)` — so springt der Haufen beim Neuzeichnen nicht:

```javascript
function seededRand(seed) {
  const x = Math.sin(seed + 1) * 10000;
  return x - Math.floor(x); // immer gleicher Wert für gleichen seed
}
```

### localStorage-Sync

Der Bohnen-Zähler wird tagesweise in `localStorage` gespeichert, damit `bohne.html` und `index.html` denselben Stand sehen:

```javascript
// Key: 'journal_bohnen_2026-03-24'
function bohnenLsKey() {
  return 'journal_bohnen_' + new Date().toISOString().slice(0, 10);
}
```

---

## bohne.html — Die Homescreen-Mini-App

Eine eigenständige, minimalistische Web-App mit einem einzigen Zweck: Bohnen zählen ohne die volle App öffnen zu müssen.

**Einrichten (Android):**
1. `padrasta.github.io/journal/bohne.html` in Chrome öffnen
2. Menü (⋮) → „App installieren" oder „Zum Startbildschirm"
3. Fertig — die Bohne liegt als App-Kachel auf dem Homescreen

`bohne.html` und `index.html` teilen sich denselben localStorage-Key. Wenn man auf dem Homescreen eine Bohne tippt und dann das Journal öffnet, ist der Zähler automatisch synchronisiert.

---

## Mobile Ansicht (≤ 768px)

Auf Smartphones wird eine vereinfachte Ansicht geladen:

| Desktop | Mobile |
|---------|--------|
| Rechte Sidebar mit allen Tools | Untere Navigationsleiste |
| Stempel, Text, Archiv | Nur: 🫘 Bohne · 📝 Notiz · 🖼️ Foto · ☁️ Drive · 💾 Speichern |
| Manuelles Laden | Auto-Load aus localStorage |

### Auto-Save

Ein `MutationObserver` beobachtet die Canvas und speichert bei jeder Änderung automatisch (debounced, max. einmal pro 1,5 Sekunden) in `localStorage`:

```javascript
const autoSaveObserver = new MutationObserver(autoSpeichern);
autoSaveObserver.observe(document.getElementById('flaeche'), {
  childList: true,   // Elemente hinzugefügt/entfernt
  subtree: true,     // auch tief verschachtelte Änderungen
  attributes: true,  // Positionsänderungen
  characterData: true // Texteingaben
});
```

Beim nächsten Öffnen auf Mobile wird der heutige Stand automatisch wiederhergestellt — kein manuelles Laden nötig.

---

## Touch-Drag (Pointer Events API)

Ursprünglich nutzte die App nur `mousedown/mousemove/mouseup`. Diese Events funktionieren am Desktop, aber nicht mit Fingern auf Touchscreens.

**Lösung:** Wechsel auf die **Pointer Events API**, die für Maus, Touch und Stift gleich funktioniert:

```javascript
// Alt (nur Maus):
element.addEventListener('mousedown', ...)
document.addEventListener('mousemove', ...)

// Neu (Maus + Touch + Stift):
element.addEventListener('pointerdown', ...)
document.addEventListener('pointermove', ...)
```

Zusätzlich muss `touch-action: none` im CSS gesetzt werden, damit der Browser bei Berührung nicht versucht zu scrollen:

```css
.polaroid, .klebezettel, .stempel-element, .text-element {
  touch-action: none;
}
```

### Klebezettel-Sonderfall

Das Problem: Der `.klebezettel-text` (contenteditable) füllt den ganzen Zettel aus. Auf dem Desktop trifft man noch den Rand zum Ziehen — mit dem Finger nie.

**Lösung:** Zwei Mechanismen:

1. **Drag-Griff** (`· · ·`) oben am Zettel — immer zuverlässig ziehbar
2. **Bewegungsschwellenwert** bei Touch auf dem Text: erst nach >8px Bewegung wird aus einem Tap ein Drag. Kurzes Antippen öffnet weiterhin die Tastatur.

```javascript
// Drag erst nach Schwellenwert (nur bei Touch auf Text)
if (!amZiehen) {
  if (Math.hypot(dx, dy) < 8) return; // noch kein Drag
  amZiehen = true;
  zettel.classList.add('wird-gezogen');
}
```

---

## PWA (Progressive Web App)

Beide Apps (`index.html` und `bohne.html`) haben ein Web App Manifest. Das erlaubt die Installation auf dem Homescreen mit eigenem Icon und App-Fenster (kein Browser-Chrome).

`manifest.json` enthält auch einen **Shortcut** — Long-Press auf das Journal-Icon auf Android zeigt „Schöner Moment" als Schnellzugriff direkt zu `bohne.html`.

---

## Technologie-Stack

| Was | Womit |
|-----|-------|
| Framework | Keins — vanilla HTML/CSS/JavaScript |
| Fonts | Google Fonts (Playfair Display, Inter, Caveat) |
| Auth | Google Identity Services (OAuth 2.0 Implicit Flow) |
| Storage | Google Drive API v3, localStorage |
| Touch | Pointer Events API |
| Deployment | GitHub Pages |

---

*Dokumentation erstellt mit Claude Sonnet 4.6*

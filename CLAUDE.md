# CLAUDE.md — Apple Health Ultra-Trainings-Webapp

> Vollständige Migrationszusammenfassung für Claude Code.  
> Stand: 26. April 2026 | Datei: `apple_health_analyse.html` (~164 KB, ~3.030 Zeilen)

---

## 1. Projektübersicht

### Was ist das?

Eine **Single-File-HTML-Webapp** zur Analyse von Apple Health Exportdaten — kombiniert mit einem personalisierten **15-Wochen-Trainingsplan** für einen **100km-Ultramarathon mit 3.300 Höhenmetern** am 30. Juli 2026.

### Nutzer

Genau ein Nutzer: Jahrgang 1969 (57 Jahre), 30 Marathons (letzter 2019), aktuell wieder aktiv (~50 km/Woche), Büropendler (5:30 Uhr morgens), max. 4 Läufe/Woche, trainiert unter der Woche auf Schotter und am Wochenende in den Bergen.

### Zweck

1. **Health-Analyse**: Daten aus `Export.xml` (Apple Health) + GPX-Dateien aus `workout-routes/` einlesen, analysieren, visualisieren
2. **Ultra-Trainingsplan**: Personalisierter 15-Wochen-Plan mit Gamification, Check-ins, Notion-Export
3. **Fortschritts-Tracking**: Plan mit realen Laufdaten abgleichen, Kommentare und Anpassungen generieren
4. **Export**: Workouts als JSON/CSV oder direkt in Notion-Datenbank pushen

---

## 2. Tech-Stack

### Sprachen
- **HTML5** (Struktur)
- **CSS3** (alles inline im `<style>`-Block, CSS-Custom-Properties für Theming)
- **Vanilla JavaScript ES2020+** (kein Framework, kein Build-Step)

### Externe Libraries (CDN, keine npm)
```
Chart.js 4.4.3     → Balken-, Linien-, Donut-Charts
Leaflet 1.9.4      → Interaktive GPX-Karten (OpenStreetMap)
```

### Browser-APIs
```
File System Access API (showDirectoryPicker)  → Chrome Desktop, lokal
webkitdirectory input                          → Safari/iPad Fallback
localStorage                                   → Plan-Persistenz, Cache
Blob / URL.createObjectURL                     → Datei-Downloads
Notion REST API v1 (fetch)                    → Direkt-Import
```

### Deployment
- **Keine Server, kein Build** — einfach `apple_health_analyse.html` in Chrome öffnen
- Funktioniert via `file://` Protokoll (lokal)
- Läuft auch auf iPad/iPhone Safari (eingeschränkt: kein Ordner-Picker)

---

## 3. Projektstruktur

```
outputs/
├── apple_health_analyse.html     ← Die komplette Webapp (alles in einer Datei)
├── apple_health_analyzer.py      ← Python-Hilfsskript für Datenanalyse (separat)
├── apple_health_webapp.jsx       ← Alte React-Version (obsolet, Referenz)
└── CLAUDE.md                     ← Diese Datei

uploads/ (Nutzerdaten, nicht in Repo)
├── workouts.json                 ← Export aus der Webapp (letzte 6 Monate)
└── workouts__5_.json             ← Ältere Export-Version
```

### Interne HTML-Datei-Struktur
```
<head>
  CDN-Scripts (Chart.js, Leaflet)
  <style>  ← ~195 Zeilen CSS
</head>
<body>
  <!-- SCREENS (4) -->
  screen-welcome      ← Start wenn Cache vorhanden
  screen-upload       ← Upload-Screen (default active)
  screen-loading      ← Progress während XML-Parse
  screen-dashboard    ← Haupt-App mit Tabs

  <!-- OVERLAYS -->
  checkin-overlay     ← Modal für täglichen Check-in

  <!-- PLAN STYLES -->  ← Extra CSS (muss VOR </script> stehen!)

  <script>  ← ~2.623 Zeilen JS (98 Funktionen)
</body>
```

### Tab-Struktur im Dashboard
```
overview    → Stat-Karten, Monats-/Jahresbalken
trainings   → Sportarten-Donut, Wochentag-Analyse, Heatmap
details     → Trainings-Liste + Detailansicht (Karte, HF-Chart, Höhenprofil)
heartrate   → HF-Zonen, Ruhepuls-Verlauf
export      → Datenqualitäts-Dashboard, JSON/CSV-Download, Notion-Push
runs        → Lauf-Analyse: KPIs, Monats-/Jahresvergleich, Plan-Soll-Ist-Kurve
plan        → 15-Wochen-Trainingsplan mit Gamification
```

---

## 4. Wichtige Architekturentscheidungen

### A: Single-File-HTML — bewusst, kein Build-Step
Alles in einer `.html`-Datei. Keine Dependencies, kein npm, kein Bundler. Der Nutzer öffnet die Datei einfach lokal. Das macht Updates (per Claude) einfach und die App vollständig offline-fähig.

### B: Streaming XML-Parser (4MB-Chunks)
Apple Health `Export.xml` kann 100–500 MB groß sein. Die App parst sie in `CHUNK = 4 * 1024 * 1024` Byte-Blöcken mit `yield` (async Generator) um den Browser nicht zu blockieren. Kein FileReader.readAsArrayBuffer auf einmal.

### C: GPX-Matching via erstem `<trkpt><time>` (UTC)
**Nicht** via Dateiname! Apple-Dateinames (`route_2018-03-16_7.46pm.gpx`) haben lokale Zeit im 12h-Format — timezone-fehleranfällig. Der erste `<trkpt><time>` enthält exaktes UTC → direkt gegen `workout.startMs` vergleichen. Fenster: ±30 min.

```javascript
function gpxFirstTrkptUtc(text){
  const i = text.indexOf('<trkpt');
  const seg = text.slice(i, i+400);
  const m = seg.match(/<time>([^<]+)<\/time>/);
  return Date.parse(m[1]);
}
```

### D: Zwei Upload-Wege je nach Browser
```
Chrome Desktop (lokal)  → showDirectoryPicker() → ganzer Ordner
Safari / iPad / iframe  → <input type="file" onchange="onXmlPicked()">
```
Erkennung: `var inIframe = window.self !== window.top` + `typeof showDirectoryPicker === 'function'`

**Wichtig:** `showDirectoryPicker` funktioniert **nicht** in iframes (z.B. Claude-Vorschau). Datei muss lokal in Chrome geöffnet werden.

### E: localStorage für Plan-Persistenz
```javascript
const PLAN_KEY = 'ultra100k_plan_v6';  // Version bumpen bei Breaking Changes!
// Keys: 'cachedWorkouts', 'lastSyncDate', 'last_checkin_day'
```
Safari privater Modus blockt localStorage → **immer** `try/catch` um jeden localStorage-Aufruf!

### F: Alle Event-Listener inline (onclick/onchange)
Nach vielen Problemen mit `addEventListener` zur Laufzeit (null-Elemente, Timing, doppelte Registrierung) werden Upload-Handler direkt im HTML gesetzt:
```html
<button onclick="onFolderClick()">
<input  onchange="onXmlPicked(event)">
```

### G: DOM-Reihenfolge ist kritisch
Das `<script>`-Tag muss **nach** allen HTML-Elementen stehen (vor `</body>`). Das `checkin-overlay` und Plan-Styles müssen ebenfalls **vor** dem `</script>` stehen — sonst gibt `getElementById` null zurück und der gesamte JS-Start schlägt fehl.

### H: HR-Zonen basieren auf gemessener MaxHR=159
**Nicht** auf der Formel 220-Alter=164. MaxHR aus Daten gemessen.
```javascript
const MAX_HR = 159;
ZONE_DEFS: Z1(50-60%), Z2(60-70%), Z3(70-80%), Z4(80-90%), Z5(90-100%)
Z2 = 95-111 bpm = 6.8-7.2 min/km (sehr langsam — wichtigste Erkenntnis!)
```

---

## 5. Konventionen & Regeln

### Coding-Stil
- **Kein TypeScript**, kein Linter, keine Formatierung — bewusst minimalistisch
- `const` / `let` für alles außer Loop-Counter
- `var` nur in `showUploadButtons()` (wegen Browser-Kompatibilität und Hoisting)
- Template-Literals für HTML-Strings in render-Funktionen
- Arrow-Functions für Callbacks, `function`-Deklarationen für Top-Level

### Namenskonventionen
```
render*()          → Baut/aktualisiert einen Tab (renderPlan, renderRuns, …)
build*()           → Initialisiert größere UI-Bereiche (buildDashboard)
show*()            → Zeigt Screen/Overlay (showScreen, showUpload, showWelcomeOnLoad)
on*()              → Event-Handler (onFolderClick, onXmlPicked)
handle*()          → Komplexe Event-Verarbeitung (handlePlanJsonUpload)
calc*()            → Berechnung ohne Side-Effects (calcXP, calcLevel)
parse*()           → Parsing-Funktionen (parseGPX, parseGPXElevationOnly)
UPPER_CASE         → Globale Konstanten (PLAN_KEY, RACE_DATE, ATHLETE)
_privateHelper     → Nicht verwendet — alles ist global im Script
```

### CSS-Variablen (Design-Token)
```css
--bg:#05050e        /* Hintergrund */
--panel:#0c0c1e     /* Panel/Card innen */
--card:#131326      /* Card-Hintergrund */
--border:#32325a    /* Borders */
--txt:#eaeaf8       /* Text */
--dim:#9898c0       /* Gedimmter Text */
--accent:#a898ff    /* Lila — primäre Akzentfarbe */
--green:#1affa0     /* Erfolg, Z2 */
--orange:#ffaa55    /* Warnung, Z4 */
--red:#ff5577       /* Fehler, Z5 */
--blue:#33ddff      /* Info */
--yellow:#ffe566    /* Höhenmeter, Z3 */
```

### Do's
- localStorage **immer** in `try/catch` wrappen
- `PLAN_KEY` versionieren (`v7`, `v8`…) bei Breaking Changes am Plan-Schema
- Alte Plan-Keys beim Start löschen: `['v1','v2',...].forEach(k=>localStorage.removeItem(k))`
- `async/await` + `await yF()` nach rechenintensiven Operationen (Browser-Refresh)
- `node --check datei.js` nach größeren JS-Änderungen

### Don'ts
- **Kein** `showDirectoryPicker` in Iframe-Kontext aufrufen
- **Keine** Event-Listener zur Laufzeit auf Upload-Elemente — nur inline `onclick`
- **Keine** `const` für Variablen die vor ihrer Deklarationszeile aufgerufen werden (TDZ)
- **Nicht** HTML-Elemente nach dem `</script>` platzieren wenn sie im Script referenziert werden
- **Nicht** `dayNames` oder andere Variablen doppelt deklarieren (führt zu SyntaxError)
- **Kein** `webkitdirectory` auf großen Ordnern (>500 Dateien) — Chrome bricht stillschweigend ab

---

## 6. Build- & Run-Befehle

### Starten (lokal)
```bash
# Einfach in Chrome öffnen:
open -a "Google Chrome" apple_health_analyse.html

# Oder Doppelklick auf die Datei im Finder
```

### Syntax-Check nach JS-Änderungen
```bash
# Node.js muss installiert sein
node --check apple_health_analyse.html   # funktioniert nicht direkt

# JS extrahieren und prüfen:
python3 -c "
with open('apple_health_analyse.html') as f: c = f.read()
js = c[c.find('<script>\n// ═══'):c.rfind('</script>')]
with open('/tmp/check.js','w') as f: f.write(js[8:])
"
node --check /tmp/check.js
```

### Plan-Key bumpen (nach Schema-Änderungen)
```bash
# In der HTML-Datei:
# const PLAN_KEY = 'ultra100k_plan_v6';  →  'ultra100k_plan_v7'
# Außerdem in der Purge-Liste ergänzen:
# ['v1','v2','v3','v4','v5','v6'].forEach(k=>localStorage.removeItem(k))
```

### Debugging im Browser
```javascript
// In der DevTools-Console:
ALL_WORKOUTS.length           // Anzahl geladene Workouts
GPX_INDEX.size                // Anzahl gematchte GPX-Dateien
loadPlan()                    // Aktueller Planstand
calcXP(loadPlan())            // XP-Stand
localStorage.clear()          // Cache + Plan löschen → kompletter Reset
```

---

## 7. Offene TODOs & nächste Schritte

### Upload
- [ ] **Ordner-Picker auf iPad**: `showDirectoryPicker` wird auch auf neueren iPadOS-Versionen eventuell unterstützt — testen
- [ ] **Drag & Drop**: Ordner auf Upload-Screen ziehen als dritte Option (für Chrome bereits vorbereitet gewesen, dann entfernt)

### GPX & Höhenmeter
- [ ] **`elevationDown` im Export**: Abwärts-Höhenmeter werden berechnet und angezeigt, aber noch nicht vollständig im Notion-Export integriert
- [ ] **precomputeElevation Progress**: Fortschrittsanzeige beim Laden von 1.794 GPX-Dateien verbessern (aktuell nur alle 20 Dateien ein Update)

### Trainingsplan
- [ ] **Dienstreise-Modus**: Plan anpassen wenn Wochenende wegfällt (Alternative: Do/Fr als Ersatz-Long-Run)
- [ ] **Auto-Anpassung nach Upload**: Nach jedem `workouts.json`-Upload automatisch Plan neu bewerten und Volumenziele anpassen
- [ ] **Morning Prompt aktiv testen**: `maybeShowDailyCheckin()` — täglich prüfen ob Vortag eingetragen
- [ ] **Notion Kalender**: CSV-Export testen und in Notion-Datenbank importieren

### Analyse
- [ ] **Trail-Anteil**: Läufe mit GPX nach Trail vs. Asphalt kategorisieren
- [ ] **Pace-Zonen-Chart**: Wie viel % der km wurden in welcher Pace-Zone gelaufen
- [ ] **Wochen-Vorschau**: Nächste Planwoche prominent im Dashboard anzeigen

---

## 8. Bekannte Probleme / Fallstricke

### KRITISCH: DOM-Reihenfolge
Das `checkin-overlay` und Plan-Styles **müssen** vor dem `</script>`-Tag stehen. Wenn sie danach stehen, gibt `document.getElementById('checkin-overlay')` beim Script-Start `null` zurück → TypeError → gesamter JS-Start bricht ab → schwarzer Bildschirm.

### KRITISCH: localStorage in Safari Private Mode
Safari im privaten Modus wirft `SecurityError` bei jedem localStorage-Zugriff. **Alle** localStorage-Calls müssen in `try/catch` sein. Gilt auch für normalen Safari wenn Datenschutzeinstellungen restriktiv sind.

### KRITISCH: const TDZ (Temporal Dead Zone)
```javascript
// FALSCH — crasht mit ReferenceError:
initUploadUI();           // ruft _hasDP auf
const _hasDP = ...;       // wird erst hier deklariert

// RICHTIG — var oder inline:
function initUploadUI(){
  var hasDP = typeof window.showDirectoryPicker === 'function';
}
```
`const`/`let` werden **nicht** gehoisted. `function`-Deklarationen schon.

### KRITISCH: showDirectoryPicker nur Top-Level
`showDirectoryPicker` funktioniert **nicht** in iframes, also nicht in der Claude.ai Vorschau. Datei muss lokal in Chrome geöffnet werden. Erkennung: `window.self !== window.top`.

### WARNUNG: webkitdirectory Limit
`<input webkitdirectory>` auf dem apple_health_export-Ordner (1.800+ GPX + Export.xml) führt in Chrome zum stillen Abbruch des `change`-Events. Lösung: `showDirectoryPicker` für Chrome, einfacher Datei-Input für Safari.

### WARNUNG: Plan-Version
Bei Schema-Änderungen am Plan-Objekt (neue Felder, geänderte Struktur) **muss** `PLAN_KEY` auf `v7`, `v8` etc. hochgezählt werden, sonst lädt der Browser alten inkompatiblen Plan aus dem Cache.

### WARNUNG: dayNames Duplikat
`const dayNames` darf nur **einmal** in `generatePlan()` deklariert werden. Doppelte `const`-Deklaration im selben Scope = SyntaxError der alles einfriert.

### WARNUNG: GPX-Matching
Apple GPX-Dateinamen (`route_2018-03-16_7.46pm.gpx`) enthalten **lokale** Zeit, **nicht** UTC. Matching **niemals** über Dateinamen machen — immer über ersten `<trkpt><time>` (UTC) gegen `workout.startMs`.

### INFO: Max Runs per Week
Nutzer kann max. **4 Läufe/Woche** machen. Plan-Template: `['R','EL','R','EL','R','LLH','EL']` — Mo/Mi/Fr sind immer Ruhetage. R-Tage werden in der UI-Pill-Ansicht herausgefiltert.

### INFO: Bergläufe und Z2
Bei Bergläufen geht der Puls **immer** in Z3-Z4 — das ist normal und kein Fehler. Bergauf wird nach **RPE 5-6** gesteuert (nicht nach Puls). Gehen ab ~12% Steigung ist explizite Ultra-Strategie.

---

## Athleten-Profil (hardcoded in `ATHLETE`-Objekt)

```javascript
const ATHLETE = {
  weeklyKmBase:  55,    // Ø 50km/Woche, Peak 62km (aus Health-Daten)
  realMaxHR:     159,   // gemessen, nicht Formel
  paceZ2:        6.8,   // min/km bei 95-111 bpm
  paceTrail:     9.5,   // min/km schwerer Trail
  targetFinish:  13.5,  // Stunden Zielzeit 100k
  maxRunsPerWeek: 4,
};
```

**Fitness-Stand (23. April 2026):**
- Ø 50 km/Woche, Peak-Woche 62 km
- Längste Läufe: 31,3 km flach + 25,6 km/1.812 hm Trail
- 31 Hikes / 34.300 hm in 6 Monaten
- 84% aller Läufe in Z3-Z4 → Z2-Disziplin = größter Hebel
- **Absolvierte Plan-Einheiten**: Di 21.4. ✅ (9,9 km / HR 131) · Do 23.4. ✅ (10,5 km / HR 117 = echtes Z2!)

---

*Generiert aus Konversationsverlauf und Quellcode-Analyse. Stand: 26. April 2026.*

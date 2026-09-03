# Rod's Kochkalkulator – Projektdokumentation

Letzter Stand: App-Version **v1.1.1** (Stand: 3. September 2026)

Diese Datei ist für zwei Zielgruppen gedacht: Rod selbst (Gedächtnisstütze) und jede neue Claude-Instanz, der Rod diese Datei in einem neuen Chat gibt. Wenn du eine Claude-Instanz bist, die das hier liest: lies das komplett durch, bevor du irgendwas am Projekt änderst.

---

## 1. Was das hier ist

Ein selbstgebauter Kalorien-/Makro-Rechner für's Kochen (kcal, Protein, Kohlenhydrate, Fett, Leucin/MPS-Trigger), mit:
- Rezept-Verwaltung (speichern, laden, löschen)
- Eigener Lebensmittel-Datenbank
- Topf/Pfannen-Gewichte für Tara-Berechnung (Bruttogewicht minus Topfgewicht)
- Getrennter Bereich für "Beilagen/Extras" (nicht mitgekochte Zutaten wie Reis, Nudeln, Parmesan)

Läuft als **eine einzige HTML-Datei** (`index.html`), gehostet über **GitHub Pages**, Daten liegen in **Dropbox**.

**Live-URL:** `https://schwarzwelpe.github.io/Kochkalkulator/`

---

## 2. Warum diese Architektur (die Vorgeschichte, kurz)

Ursprünglich lief der Kalkulator als React-Artefakt direkt in einem Claude-Chat. Zwei Kernprobleme haben zum Umbau geführt:
1. Speicherung über `localStorage` bzw. die Claude-eigene `window.storage`-API war in der Umgebung, in der die Datei lief (Datei-Vorschau statt echtes natives Artefakt), **nicht zuverlässig** – Rezepte gingen beim erneuten Öffnen verloren.
2. Ein Diagnose-Test hat bewiesen, dass in dieser Umgebung weder `window.storage` noch Aufrufe an `api.anthropic.com` funktionieren (beide Bridges fehlen in der Datei-Vorschau-Sandbox).

**Lösung:** Raus aus der Claude-Sandbox, rein in eine echte, unabhängige Webseite (GitHub Pages), mit echter Cloud-Anbindung (Dropbox API direkt, nicht über Claude-Bridges).

**Wichtige Lektion für die Zukunft:** Falls jemals wieder jemand vorschlägt, das als Claude-Artefakt zu bauen: NICHT TUN, außer es ist ausdrücklich gewünscht als Wegwerf-Prototyp. Die GitHub+Dropbox-Lösung ist die dauerhafte, funktionierende Variante.

---

## 3. Die drei Bausteine

### 3.1 GitHub (Code-Hosting)
- **Repo:** `github.com/Schwarzwelpe/Kochkalkulator` (öffentlich, Pflicht für kostenloses GitHub Pages)
- **Datei:** `index.html` im Root des Repos – das ist die GANZE App (HTML + CSS + JavaScript in einer Datei, keine Build-Schritte, kein npm, kein React/Babel – siehe Abschnitt 5 warum)
- **GitHub Pages:** aktiviert unter Settings → Pages → Branch: `main`, Ordner: `/ (root)`
- **Updates:** aktuell durch manuellen Upload (Add file → Upload files → alte Datei wird ersetzt → Commit changes). Ein Personal Access Token für direkten `git push` durch Claude ist in Planung (siehe Abschnitt 8).

### 3.2 Dropbox (Datenspeicherung)
- **Dropbox App:** "Kochkalkulator", App-Typ: **Scoped Access, App folder** (nicht "Full Dropbox")
- **App Key:** `nns9uy3os0aolu9`
- **Berechtigungen (Permissions-Tab in der Dropbox App Console):** `files.content.write` und `files.content.read` müssen aktiviert und mit "Submit" bestätigt sein
- **Redirect URI (OAuth):** `https://schwarzwelpe.github.io/Kochkalkulator/` (muss exakt mit der Live-URL inkl. abschließendem Schrägstrich übereinstimmen)
- **Datendatei:** `/Apps/Kochkalkulator/kochkalkulator-daten.json` (das ist der App-Folder-Pfad; aus Sicht der App selbst heißt der Pfad einfach `/kochkalkulator-daten.json`, weil Scoped-Access-Apps automatisch in ihrem eigenen Ordner landen)
- Diese Datei enthält: die Lebensmittel-Datenbank (`db`), alle gespeicherten Rezepte (`recipes`), die Töpfe/Pfannen (`toepfe`)

### 3.3 Der Login-Mechanismus (Dropbox OAuth mit PKCE)
- Kein Client-Secret nötig (PKCE = Proof Key for Code Exchange, sicher genug für reine Browser-Apps)
- Beim ersten Login pro Gerät/Browser: Klick auf "Mit Dropbox verbinden" → Weiterleitung zu Dropbox → Login/Bestätigung dort → zurück zur App mit einem Autorisierungscode → App tauscht den Code gegen Access Token + Refresh Token
- **Access Token:** kurzlebig (paar Stunden)
- **Refresh Token:** langlebig, wird in `localStorage` des jeweiligen Browsers gespeichert (Key: `dbx_tokens`), erneuert den Access Token automatisch im Hintergrund
- **Wichtig:** Login ist pro Browser/Gerät getrennt. iPhone Safari und Mac Chrome brauchen je einen einmaligen Login. Das ist normal, kein Bug.
- Bei Problemen: auf der Fehler-Seite gibt's einen "Abmelden & neu versuchen"-Button, der die gespeicherten Tokens löscht

---

## 4. Datenstruktur (`kochkalkulator-daten.json`)

```json
{
  "db": [
    { "id": 1, "num": 1, "name": "Basmati-Reis Oryza", "kcal": 354, "protein": 9, "carbs": 77, "fat": 1, "leu": 0.065 }
  ],
  "recipes": [
    {
      "id": "eindeutige_id",
      "name": "Rezeptname",
      "ingredients": [ { "dbId": 2, "grams": "250" } ],
      "extras": [ { "dbId": 5, "grams": "350" } ],
      "endW": "1200",
      "pInput": "600",
      "pMode": "grams",
      "savedAt": "3.9.2026"
    }
  ],
  "toepfe": [
    { "id": "t1", "name": "Hoffmann Topf (groß)", "weight": 2503 }
  ],
  "savedAt": "ISO-Zeitstempel der letzten Änderung"
}
```

**Felder-Erklärung:**
- `db[].leu` = Leucin-Anteil am Protein dieses Lebensmittels (für die MPS-Trigger-Ampel). Fällt auf `0.065` zurück, wenn nicht gesetzt.
- `recipes[].ingredients` = die eingekochten Zutaten (Rohgewicht)
- `recipes[].extras` = Beilagen, die NICHT mitgekocht werden (werden nach der Portionsberechnung addiert)
- `recipes[].endW` / `pInput` / `pMode` = der zuletzt gespeicherte Zustand von Fertiggewicht und Portionsgröße für dieses Rezept

---

## 5. Technische Entscheidungen (und warum)

- **Kein React, kein Babel, kein Build-Schritt:** Ein früherer Versuch mit React + Babel-Standalone (Live-Übersetzung von JSX im Browser) ist mit einem `SyntaxError` gescheitert. Statt das zu debuggen, wurde komplett auf **reines, klassisches JavaScript** (ES5-Stil, `var` statt `let/const` wo möglich, keine JSX) umgestellt. Das ist robuster, weil nichts im Browser live übersetzt werden muss.
- **Manuelles DOM-Rendering statt Framework:** Die App rendert HTML-Strings direkt in `innerHTML` und hängt danach Event-Listener dran (`wireRechnerEvents()` etc.). Kein Virtual DOM, keine Abhängigkeiten. Funktioniert zuverlässig, ist nur etwas mehr Code pro Feature.
- **Globale Fehlerbehandlung:** `window.addEventListener("error", ...)` und `"unhandledrejection"` fangen JS-Fehler ab und zeigen sie als lesbaren Text auf der Seite an (statt einer stillen schwarzen Seite). Das war entscheidend, um den Babel-Fehler überhaupt sichtbar zu machen.
- **Vor jedem Datei-Upload:** die JavaScript-Syntax wird mit `node --check` lokal geprüft, bevor Rod die Datei hochlädt. Diese Disziplin bitte beibehalten.

---

## 6. Wie man Änderungen macht

### 6.1 Reine Datenänderungen (neues Rezept, neues Lebensmittel, Wert korrigieren)
**Kein Upload nötig!** Claude kann mit den Dropbox-Tools direkt in `/Apps/Kochkalkulator/kochkalkulator-daten.json` schreiben (lesen mit `Dropbox:fetch`, dann `Dropbox:delete` + `Dropbox:create_file` zum Überschreiben, da `create_file` nicht direkt überschreiben kann). Rod muss dafür nur die App im Browser neu laden.

### 6.2 Code-/Feature-Änderungen (neue Funktion, neues UI-Element)
1. Claude bearbeitet `index.html` (lokal im Sandbox-Dateisystem unter `/mnt/user-data/outputs/index.html`)
2. **Immer vor dem Ausliefern:** Syntax prüfen mit:
   ```
   python3 -c "import re; c=open('index.html').read(); m=re.search(r'<script>(.*?)</script>', c, re.DOTALL); open('/tmp/test.js','w').write(m.group(1))"
   node --check /tmp/test.js
   ```
3. Versionsnummer in `APP_VERSION` hochzählen (aktuell `"1.1.1"`)
4. Datei präsentieren, Rod lädt sie manuell bei GitHub hoch (Add file → Upload files → Commit changes) – ODER, falls das Token-Setup (Abschnitt 8) steht: Claude pusht direkt per `git`

---

## 7. Bekannte Stolpersteine (falls's nochmal knallt)

| Symptom | Ursache | Lösung |
|---|---|---|
| Ganze Seite schwarz, keine Fehlermeldung | JS-Fehler ohne sichtbare Anzeige | Ist behoben durch die globalen Error-Handler; falls es doch wieder passiert, Browser-Konsole checken |
| "missing_scope: files.content.read" | Dropbox-App-Berechtigungen nicht aktiviert/gespeichert | Dropbox App Console → Permissions → Haken setzen → Submit → **danach in der App neu einloggen**, alte Tokens kennen die neuen Rechte nicht |
| Fehler-Screen ohne Abmelden-Button, man hängt fest | War ein Bug in v1.0.0, seit v1.0.1 hat auch der Fehler-Screen einen Abmelden-Button | Falls trotzdem: Browser-Website-Daten für die Domain löschen (iPhone: Einstellungen → Safari → Erweitert → Website-Daten) |
| Dropdown-Auswahl sprengt Layout auf dem Handy | Native `<select>`-Breite richtet sich nach dem längsten Eintrag | CSS-Regel `select, input { min-width: 0; }` behebt das |
| Rezept ist nach Reload weg | localStorage/window.storage in Claude-Artefakt-Sandbox unzuverlässig | Genereller Grund für den kompletten Umbau auf GitHub+Dropbox; sollte in dieser Architektur nicht mehr auftreten |

---

## 8. Geplant, aber noch nicht umgesetzt

**Direkter Git-Push durch Claude**, damit Rod Änderungen nur noch per Zuruf machen muss, ohne selbst bei GitHub hochzuladen:
- Rod müsste einen **Fine-grained Personal Access Token** erstellen (github.com/settings/tokens), beschränkt auf das Repo "Kochkalkulator", Berechtigung "Contents: Read and write"
- Claude nutzt den Token dann für `git push` direkt aus der Sandbox
- **Wichtig:** Dieser Token wird NICHT dauerhaft in Claudes Gedächtnis gespeichert (Sicherheitsprinzip). Rod muss ihn in jeder neuen Unterhaltung, in der Code-Änderungen gewünscht sind, erneut mitteilen, oder die gesamte Wartung findet in einer einzigen langlaufenden Unterhaltung statt.

---

## 9. Für eine neue Claude-Instanz: Checkliste beim Wiedereinstieg

1. Lies diese Datei komplett.
2. Falls Rod eine Datenänderung will (Rezept, Lebensmittel): direkt per Dropbox-Tools in `/Apps/Kochkalkulator/kochkalkulator-daten.json` schreiben. Vorher immer erst mit `Dropbox:fetch` den aktuellen Stand holen, nichts blind überschreiben.
3. Falls Rod eine Code-Änderung will: `index.html` besorgen (entweder aus GitHub via `web_fetch` auf die Live-URL zum Nachvollziehen, oder Rod hat sie lokal), bearbeiten, **Syntax mit `node --check` prüfen**, dann liefern.
4. Nie wieder React/Babel/Artefakt-Ansatz für dieses Projekt vorschlagen, außer explizit gewünscht.
5. Versionsnummer bei jeder Code-Änderung hochzählen.

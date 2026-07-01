# KeePass Web - Offline Passwort-Manager

Ein komplett offline-faehiger Passwort-Manager als Single-Page HTML-Datei. Keine Installation, kein Server, kein Tracking. Alles laeuft ausschliesslich im Browser.

## Quick Start

1. **`index.html` herunterladen** (oder das Repo clonen)
2. **Datei im Browser oeffnen** (Doppelklick oder Datei > Oeffnen)
3. **Master-Passwort waehlen** (min. 6 Zeichen, Bestaetigung erforderlich)
4. **Fertig** - Passwoerter erstellen, speichern, kopieren

Kein Server, kein Build-Tool, keine Abhaengigkeiten. Funktioniert sogar ohne Internetverbindung.

### Hosting-Optionen (optional)

Die HTML-Datei kann auch als Website gehostet werden, um sie von mehreren Geraeten zu erreichen:

- **GitHub Pages** - Repo aktivieren unter Settings > Pages, HTTPS inklusive
- **Cloudflare Pages** - Drag & Drop der HTML-Datei
- **Lokaler Webserver** - `python3 -m http.server 8080` im Verzeichnis

Auch gehostet bleiben alle Daten lokal im Browser (localStorage). Der Server liefert nur die HTML-Datei aus.

---

## Features

- **AES-256-GCM Verschluesselung** - Militaerstandard, direkt im Browser via Web Crypto API
- **PBKDF2 Key Derivation** - 600.000 Iterationen gegen Brute-Force
- **Passwort-Generator** - Konfigurierbare Laenge (8-128 Zeichen), Zeichenklassen waehlbar
- **Passwort-Staerke-Anzeige** - Visuelle Bewertung beim Erstellen
- **Kategorien** - Login, Finanzen, E-Mail, Social Media, Arbeit, Sonstiges
- **Suche und Filter** - Echtzeit-Suche nach Titel, Benutzername, URL
- **Vault Export/Import** - Verschluesselte `.vault`-Datei fuer Backup und Geraetewechsel
- **Auto-Lock** - Automatische Sperrung nach 10 Minuten Inaktivitaet
- **Clipboard-Schutz** - Zwischenablage wird nach 30 Sekunden automatisch geleert
- **Passwort-Sharing** - Einzelne Eintraege verschluesselt teilen (Zwei-Kanal-Prinzip: Link + Schluessel separat)
- **Tastaturkuerzel** - `Ctrl+N` (Neu), `Ctrl+L` (Sperren), `Ctrl+E` (Export), `Esc` (Modal schliessen)
- **Responsive Design** - Funktioniert auf Desktop, Tablet und Smartphone
- **Dark Mode** - Augenschonendes dunkles Design

---

## Architektur

### Design-Prinzip: Zero-Trust, Zero-Server

```
+--------------------------------------------------+
|                    Browser                        |
|                                                   |
|  +------------+    +-------------+    +---------+ |
|  | Lock Screen| -> | Crypto Layer| -> | Vault   | |
|  | (Auth)     |    | (AES+PBKDF2)|    | (State) | |
|  +------------+    +-------------+    +---------+ |
|        |                  |                |      |
|        v                  v                v      |
|  +------------+    +-------------+    +---------+ |
|  | Master-    |    | Web Crypto  |    | local-  | |
|  | Passwort   |    | API         |    | Storage | |
|  +------------+    +-------------+    +---------+ |
|                                                   |
|  Kein Netzwerkverkehr. Keine externen Ressourcen. |
+--------------------------------------------------+
```

Die gesamte Anwendung besteht aus einer einzigen HTML-Datei (~26 KB) mit eingebettetem CSS und JavaScript. Es gibt:

- **Keinen Server** - Kein Backend, keine API-Calls, keine Datenbank
- **Keine externen Abhaengigkeiten** - Kein CDN, keine Bibliotheken, keine Frameworks
- **Keinen Netzwerkverkehr** - Kann vollstaendig offline betrieben werden
- **Keine Cookies** - Daten liegen ausschliesslich in localStorage

### Verschluesselungs-Pipeline

```
Master-Passwort
      |
      v
  [PBKDF2: 600.000 Iterationen, SHA-256, 16-Byte Salt]
      |
      v
  AES-256-GCM Key
      |
      +---> Encrypt(Vault-JSON) ---> localStorage
      |
      +---> Encrypt(Vault-JSON) ---> .vault Export-Datei
```

**Schritt-fuer-Schritt:**

1. Der Benutzer gibt das Master-Passwort ein
2. Ein zufaelliger 16-Byte Salt wird generiert (oder aus localStorage geladen)
3. PBKDF2 leitet aus Passwort + Salt einen AES-256-Schluessel ab (600.000 Iterationen)
4. Ein zufaelliger 12-Byte Initialisierungsvektor (IV) wird pro Verschluesselung generiert
5. Der Vault (JSON) wird mit AES-256-GCM verschluesselt
6. Salt und verschluesselte Daten werden in localStorage gespeichert

**Warum diese Parameter?**
- **PBKDF2 mit 600.000 Iterationen** - OWASP-Empfehlung (2023+), macht Brute-Force extrem langsam
- **AES-256-GCM** - Authenticated Encryption, schuetzt sowohl Vertraulichkeit als auch Integritaet
- **Zufaelliger IV pro Vorgang** - Verhindert Musteranalyse bei wiederholter Verschluesselung
- **Web Crypto API** - Nativ im Browser, hardwarebeschleunigt, kein JavaScript-Krypto

### Datenmodell

```
Vault (verschluesselt in localStorage)
 |
 +-- version: 2
 +-- entries[]
      +-- id:        UUID (crypto.randomUUID)
      +-- title:     String
      +-- category:  "login" | "finance" | "email" | "social" | "work" | "other"
      +-- username:  String
      +-- password:  String
      +-- url:       String (optional)
      +-- notes:     String (optional)
      +-- created:   Timestamp (ms)
      +-- modified:  Timestamp (ms)
```

### localStorage-Struktur

| Key | Inhalt |
|-----|--------|
| `keepassweb_vault` | Base64-kodierter, AES-256-GCM verschluesselter Vault (IV + Ciphertext) |
| `keepassweb_salt` | Base64-kodierter 16-Byte PBKDF2-Salt |

### Export-Dateiformat (.vault)

```
[16 Bytes Salt][12 Bytes IV][N Bytes AES-256-GCM Ciphertext]
```

Die `.vault`-Datei ist ein Binaer-Blob. Ohne Master-Passwort ist der Inhalt nicht rekonstruierbar. Die Datei kann sicher auf USB-Sticks, in Cloud-Speicher oder per E-Mail transportiert werden.

---

## Aufbau der Datei

Die `index.html` ist in drei Bloecke gegliedert:

### 1. CSS (~90 Zeilen)

- CSS Custom Properties fuer konsistentes Theming (`--bg`, `--accent`, `--text`, etc.)
- Responsive Layout mit Flexbox
- Mobile-First Breakpoint bei 600px
- Animationen fuer Toast-Benachrichtigungen und Hover-Effekte

### 2. HTML (~100 Zeilen)

| Bereich | Beschreibung |
|---------|-------------|
| **Lock Screen** (`#lockScreen`) | Master-Passwort Eingabe, Vault-Erstellung, Import-Button |
| **App** (`#app`) | Toolbar (Suche, Filter, Neu-Button), Eintrags-Liste, Footer-Aktionen |
| **Entry Modal** (`#entryModal`) | Formular zum Erstellen/Bearbeiten mit Passwort-Generator |
| **Toast** (`#toast`) | Kurzlebige Benachrichtigungen (Erfolg/Fehler) |

### 3. JavaScript (~420 Zeilen)

| Modul | Zeilen | Funktion |
|-------|--------|----------|
| **Crypto** | 200-235 | `deriveKey()`, `encrypt()`, `decrypt()` - Web Crypto API Wrapper |
| **Storage** | 237-265 | `saveVault()`, `loadVault()`, `createVault()` - localStorage I/O |
| **Auth** | 267-333 | Login/Logout, Vault-Erkennung, Lock Screen Steuerung |
| **Rendering** | 335-371 | Eintrags-Liste mit Suche, Filter, Kategorie-Icons |
| **CRUD** | 373-465 | Erstellen, Bearbeiten, Loeschen von Eintraegen |
| **Generator** | 467-515 | Passwort-Generierung mit konfigurierbaren Optionen, Staerke-Meter |
| **Export/Import** | 517-563 | Vault als verschluesselte Binaer-Datei ex-/importieren |
| **UX** | 565-613 | Tastaturkuerzel, Auto-Lock Timer, Clipboard-Timeout |

---

## Benutzung

### Vault erstellen (Erststart)

Beim ersten Oeffnen der App gibt es keinen existierenden Vault. Die App fragt nach einem neuen Master-Passwort:

1. Master-Passwort eingeben (min. 6 Zeichen)
2. Passwort bestaetigen
3. "Vault erstellen" klicken

Das Passwort wird **nirgends gespeichert**. Vergisst du es, sind die Daten unwiderruflich verloren.

### Eintrag hinzufuegen

1. `+ Neu` klicken (oder `Ctrl+N`)
2. Felder ausfuellen:
   - **Titel** (Pflicht) - z.B. "Google", "Sparkasse", "AWS"
   - **Kategorie** - fuer Filterung und visuelle Sortierung
   - **Benutzername** - Login-Name oder E-Mail
   - **Passwort** - manuell eingeben oder generieren lassen
   - **URL** - Link zur Login-Seite (optional)
   - **Notizen** - Zusatzinfos wie Sicherheitsfragen (optional)
3. "Speichern" klicken

### Passwort generieren

Im Eintrag-Formular:

1. Wuerfel-Icon neben dem Passwort-Feld klicken
2. Optionen anpassen:
   - Laenge (Standard: 20 Zeichen)
   - Grossbuchstaben (A-Z)
   - Kleinbuchstaben (a-z)
   - Ziffern (0-9)
   - Sonderzeichen (!@#$%...)
3. "Neu" fuer ein weiteres zufaelliges Passwort

Die Staerke-Anzeige unter dem Feld zeigt die Qualitaet visuell an (rot bis gruen).

### Passwoerter kopieren

In der Eintrags-Liste:

- **Personen-Icon** - Kopiert den Benutzernamen
- **Schluessel-Icon** - Kopiert das Passwort (Zwischenablage wird nach 30 Sekunden geleert)
- **Link-Icon** - Oeffnet die hinterlegte URL

### Vault sichern (Export)

1. "Vault exportieren" im Footer klicken (oder `Ctrl+E`)
2. Eine `.vault`-Datei wird heruntergeladen
3. Diese Datei sicher aufbewahren (USB-Stick, verschluesselter Cloud-Speicher)

Die Export-Datei ist AES-256-GCM verschluesselt und ohne das Master-Passwort wertlos.

### Vault wiederherstellen (Import)

1. Auf dem Lock Screen: "Vault importieren" klicken
2. `.vault`-Datei auswaehlen
3. Das Master-Passwort der Export-Datei eingeben
4. Der Vault wird geladen und in localStorage gespeichert

### Passwort teilen

Einzelne Eintraege koennen sicher mit anderen Personen geteilt werden. Das Prinzip: Link und Schluessel werden ueber **verschiedene Kanaele** gesendet (z.B. Link per E-Mail, Schluessel per WhatsApp).

**Senden:**
1. Eintrag oeffnen (auf Karte klicken)
2. "Teilen" klicken
3. **Link kopieren** und ueber Kanal 1 senden (z.B. E-Mail, Slack)
4. **Schluessel kopieren** und ueber Kanal 2 senden (z.B. WhatsApp, Signal)

**Empfangen:**
1. Link im Browser oeffnen
2. Den separat erhaltenen Schluessel eingeben
3. "Entschluesseln" klicken
4. Passwort, Benutzername etc. werden angezeigt (kopierbar)

**Sicherheit:** Der Link enthaelt den verschluesselten Eintrag (AES-256-GCM), aber nicht den Schluessel. Selbst wenn der Link abgefangen wird, ist er ohne den separat gesendeten Schluessel wertlos. Alles bleibt client-seitig.

### Master-Passwort aendern

1. Im Footer "Passwort aendern" klicken
2. Neues Passwort eingeben (min. 6 Zeichen)
3. Neues Passwort bestaetigen

Der Vault wird sofort mit dem neuen Passwort neu verschluesselt. Bestehende Export-Dateien behalten ihr altes Passwort.

### Sperren

- "Sperren" im Footer klicken
- `Ctrl+L` druecken
- **Automatisch** nach 10 Minuten Inaktivitaet

---

## Sicherheitshinweise

**Was die App schuetzt:**
- Passwoerter im Ruhezustand (verschluesselt in localStorage und Export-Dateien)
- Gegen Brute-Force Angriffe (600.000 PBKDF2-Iterationen)
- Gegen Datenmanipulation (GCM Authenticated Encryption)
- Gegen Clipboard-Leaks (automatisches Leeren nach 30s)

**Was die App NICHT schuetzt:**
- Gegen Keylogger oder Malware auf dem Geraet
- Gegen physischen Zugriff auf ein entsperrtes Geraet
- Gegen Browser-Extensions mit Zugriff auf localStorage
- Gegen das Vergessen des Master-Passworts (kein Recovery moeglich)

**Empfehlungen:**
- Starkes, einzigartiges Master-Passwort verwenden (20+ Zeichen)
- Regelmaessig Vault exportieren und Backup sicher aufbewahren
- Browser-Profil ohne unbekannte Extensions verwenden
- Auf oeffentlichen Rechnern nach Nutzung den localStorage leeren

---

## Browser-Kompatibilitaet

Benoetigt einen modernen Browser mit Web Crypto API Unterstuetzung:

- Chrome / Chromium 37+
- Firefox 34+
- Safari 11+
- Edge 79+
- Chrome fuer Android
- Safari fuer iOS 11+

Getestet und optimiert fuer Chrome auf ChromeOS (Chromebooks).

---

## Lizenz

MIT

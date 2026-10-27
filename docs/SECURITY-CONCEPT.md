# MovieTracker – Sicherheitskonzept

> Stand: 2026-10-27 · Status: **Planung**. Technische Details mit „(Spike)“ werden in Phase 1 verifiziert.
> Ergänzt Briefing §3.2 und §6 sowie [PLAN.md](PLAN.md).

---

## 1. Was geschützt wird

**Schutzgut sind die Inhalte der Datenbank**: Titel, Notizen, Tags und vor allem **Links** (z. B. zu Streaming-Angeboten), die verraten, was und wo jemand schaut.

Nicht geschützt werden muss, **dass** MovieTracker benutzt wird. Der App-Name ist unkritisch.

## 2. Gegen wen geschützt wird

| # | Angreifer / Situation | Im Schutzumfang? |
|---|---|---|
| A1 | Jemand bekommt die Dateien in die Hand (USB-Stick verloren, Kopie des Containers, Backup) | ja |
| A2 | Ein fremder Rechner, auf dem die App lief, wird später untersucht (Besitzer, Admin, Forensik) | ja, so weit technisch möglich |
| A3 | Jemand schaut auf den Bildschirm oder sieht eine Bildschirmfreigabe/Aufnahme | teilweise |
| A4 | Andere Programme bzw. Cloud-Dienste auf dem Rechner (Zwischenablage-Verlauf, Recall, Browser-Sync) | teilweise |
| A5 | Jemand hat Zugriff auf die Dateien, während die App gesperrt ist (z. B. eingehängter Container, entsperrter Rechner) | ja, wenn App-Verschlüsselung aktiv |
| A6 | Schadsoftware / Keylogger auf dem Rechner, während die App läuft | **nein**, nicht lösbar (s. §6) |
| A7 | Manipulierte App-Dateien („Evil Maid“) | **nein** (s. §6) |

## 3. Schutzschichten im Überblick

```
┌─────────────────────────────────────────────────────────────┐
│ VeraCrypt-Container (optional, nicht Teil der App)           │  → A1, A7
│  ┌───────────────────────────────────────────────────────┐  │
│  │ App-Verschlüsselung (optional, SQLCipher)              │  │  → A1, A5
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ movietracker.db                                  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
 Laufzeit: keine Spuren außerhalb des App-Ordners           → A2
 Oberfläche: Zwischenablage, Bildschirm, Links             → A3, A4
```

**VeraCrypt (oder ein anderer verschlüsselter Container) ist eine mögliche Ergänzung, keine Voraussetzung.** Viele Nutzer legen die App z. B. auf einen normalen USB-Stick oder in einen Ordner und verlassen sich **ausschließlich auf die App**. Für sie ist die App-Verschlüsselung der einzige Schutz der Inhalte (A1), und die App-Dateien selbst sind ungeschützt (A7). Das Konzept muss deshalb ohne VeraCrypt tragfähig sein. Alle Maßnahmen in §4 gelten unabhängig davon.

Wer VeraCrypt zusätzlich nutzt, bekommt zwei sich ergänzende Schichten: VeraCrypt schützt alles im Ruhezustand inklusive der App selbst. Die App-Verschlüsselung schützt die Inhalte zusätzlich, **solange der Container eingehängt ist**. In dieser Zeit kann jedes Programm des Nutzers die Dateien lesen (A5).

---

## 4. Maßnahmen im Detail

### 4.1 Daten im Ruhezustand

**Ohne App-Verschlüsselung (Standard):** `movietracker.db` ist eine normale SQLite-Datei im Klartext. Schutz nur durch VeraCrypt bzw. das Dateisystem.

**Mit App-Verschlüsselung:**

| Baustein | Technik |
|---|---|
| Datenbank | SQLCipher 4: jede Seite AES-256 verschlüsselt + HMAC-SHA512 (Manipulation wird erkannt). Gilt auch für Indizes und das Journal |
| Datenschlüssel (DEK) | 256 Bit zufällig (Kryptografie-Zufallsgenerator), wird SQLCipher direkt als Rohschlüssel übergeben (`PRAGMA key = "x'…'"`), damit entfällt SQLCipher's eigenes PBKDF2 |
| Passwort → Schlüssel (KEK) | Argon2id mit Salt; Parameter so kalibriert, dass Ableiten ca. 0,5–1 s dauert und auch auf 8-GB-Rechnern läuft (z. B. 64–256 MiB, Spike) |
| Einhüllen | DEK wird mit KEK per authentifizierter Verschlüsselung (AES-256-GCM oder XChaCha20-Poly1305, Spike) verschlüsselt |
| Wiederherstellungsschlüssel | 256 Bit zufällig, angezeigt als gruppierte Zeichenkette zum Aufschreiben; hüllt denselben DEK ein zweites Mal ein |
| `bootstrap.json` | nur: `encrypted`, KDF-Parameter, Salt, die beiden eingehüllten DEKs. Keine Inhalte, keine Zählungen, keine Einstellungen |
| Passwort ändern | DEK neu einhüllen (Millisekunden); DB bleibt unverändert |
| Aktivieren nachträglich | `sqlcipher_export` in neue Datei → atomarer Tausch → Klartextdatei löschen |
| Passwortqualität und Durchprobieren | s. §4.8 |
| Schlüssel im Arbeitsspeicher | s. §4.9 |

### 4.2 Spuren auf dem Rechner (Laufzeit)

| Maßnahme | Technik |
|---|---|
| Schreiben nur in den App-Ordner | Alle Pfade aus `Platform.resolvedExecutable`; kein `path_provider` / `shared_preferences` auf Desktop |
| Fremde Schreibzugriffe umleiten | Linux: `XDG_*`, `TMPDIR`, `GSETTINGS_BACKEND=memory`, Shader-Cache aus, vor Engine-Start im Runner gesetzt. Windows: `TEMP`/`TMP` vor Engine-Start → `data/runtime/` |
| Keine Systemdialoge | Eigener Import/Export-Dialog → keine „Zuletzt verwendet“-Einträge |
| Logs | Standardmäßig aus; nur feste Event-Codes ohne Inhalte; liegen in der DB |
| Kein Netzwerk ohne ausdrückliche Aktion | Keine Telemetrie, keine Crash-Reports, keine Abfragen im Hintergrund. Spätere Online-Funktionen (z. B. Metadaten, PLAN §3.4) nur auf ausdrückliche Aktion des Nutzers |
| Neutraler Fenstertitel | Nie den geöffneten Titel anzeigen |
| Nachweis | Portabilitätstests (Linux: `bwrap` + `strace`, Windows: Process Monitor) in CI und vor jedem Release |

### 4.3 Zwischenablage

| Maßnahme | Technik |
|---|---|
| Kein Verlauf / Cloud-Sync (Windows) | Kopieren über nativen Runner-Code mit den Formaten `ExcludeClipboardContentFromMonitorProcessing`, `CanIncludeInClipboardHistory = 0`, `CanUploadToCloudClipboard = 0` (Spike) |
| Hinweis an Zwischenablage-Manager (Linux) | Zusätzlicher Typ `x-kde-passwordManagerHint: secret` (wird u. a. von KDE Klipper beachtet) |
| Automatisches Leeren | Optional nach X Sekunden, nur wenn die Zwischenablage noch den kopierten Wert enthält |

### 4.4 Bildschirm

| Maßnahme | Technik |
|---|---|
| Links nur im Detail | Liste/Kacheln zeigen höchstens die Domain |
| Aufnahmeschutz (Windows, **Einstellung, Standard: aus**) | `SetWindowDisplayAffinity(WDA_EXCLUDEFROMCAPTURE)`: Fenster erscheint in Screenshots, Bildschirmaufnahmen, Bildschirmfreigaben und (laut Microsoft) Windows Recall als schwarze Fläche (Spike). Nachteil: Man kann die App selbst nicht mehr per Screenshot oder Remote Desktop zeigen, deshalb schaltet der Nutzer ihn selbst ein |
| Android (später) | `FLAG_SECURE` bei aktiver Verschlüsselung |
| Sperre | Auto-Sperre als **Einstellung** (nur mit Verschlüsselung): nach Inaktivität (Minuten), beim Minimieren, vor Ruhezustand / Bildschirmsperre jeweils einzeln wählbar. Beim Sperren: DB schließen, UI-Zustand verwerfen |

### 4.5 Links öffnen
Standardaktion ist **Kopieren**. „Öffnen“ ist eine bewusste Aktion. Beim ersten Mal erscheint ein Hinweis, dass der Browser den Link im Verlauf speichert und ggf. in ein Browser-Konto synchronisiert.

### 4.6 Exporte

| Fall | Verhalten |
|---|---|
| Verschlüsselung an | Export standardmäßig verschlüsselt (Argon2id + authentifizierte Verschlüsselung, eigenes Passwort oder DB-Passwort) |
| Verschlüsselung aus | Klartext-JSON mit Option „Export mit Passwort schützen“ |
| Ablage | `data/exports/`, neutrale Dateinamen |

### 4.7 Abhängigkeiten
Jede Abhängigkeit wird vor Aufnahme auf Datei-/Netzwerkzugriffe geprüft (Briefing §3.1.4) und mit `pubspec.lock` festgeschrieben. Zur Laufzeit lädt die App nichts nach.

### 4.8 Schutz gegen Durchprobieren von Passwörtern (Brute Force)

Es gibt zwei Angriffswege, und eine Bremse in der App wirkt nur bei einem davon:

| Angriffsweg | Beispiel | Bremst eine Verzögerung in der App? |
|---|---|---|
| **Online** | Jemand sitzt vor der gesperrten App und tippt Passwörter | ja |
| **Offline** | Jemand kopiert `bootstrap.json` (+ `movietracker.db`) und probiert mit eigener Software und Grafikkarten | **nein**, der App-Code läuft dabei gar nicht |

Der Offline-Angriff ist der realistische: Wer an die Dateien kommt, nimmt sie mit. Deshalb sind die eigentlichen Bremsen **Argon2id und die Passwortstärke**. Eine Verzögerung in der App ist nur eine Ergänzung.

**Maßnahmen:**
1. **Argon2id-Kosten (Hauptbremse, wirkt auch offline):** Jeder Versuch kostet ca. 1 s Rechenzeit **und** z. B. 256 MiB Arbeitsspeicher (Spike). Weil das Verfahren speicherintensiv ist, kann ein Angreifer auf Grafikkarten nur wenige Versuche parallel rechnen. Die Parameter liegen in `bootstrap.json` und werden mit der Datei mitgenommen; ein Angreifer kann sie nicht absenken, ohne dass der Schlüssel falsch wird.
2. **Passwortstärke empfehlen, nicht vorschreiben:** Stärkeanzeige mit Empfehlung (z. B. ≥ 12 Zeichen oder Passphrase aus ≥ 4 Wörtern) und Erklärung, was ein schwaches Passwort bedeutet. Technisch ist jedes Passwort erlaubt. Optional generiert die App eine zufällige Passphrase.
3. **Keine Wartezeit nach Fehlversuchen** (Entscheidung): bremst nur den Online-Weg und wäre per Neustart umgehbar.
4. **Hardwaregebundene Begrenzung** (wie bei Smartphone-PINs über TPM / Secure Enclave): auf dem Desktop **nicht möglich**, weil sie den Schlüssel an einen Rechner binden würde, was der Portabilität widerspricht. Für Android später prüfen.
5. **Später optional: Schlüsseldatei als zweiter Faktor** (Datei auf einem separaten Stick). Ohne die Datei ist Durchprobieren aussichtslos, egal wie schwach das Passwort ist.

**Größenordnung** (grobe Annahme: Angreifer schafft 1.000 Versuche/s gegen Argon2id mit 256 MiB; gilt nur für **zufällig** gewählte Passwörter, selbst ausgedachte sind deutlich schwächer):

| Passwort | Kombinationen | Zeit, bis alle durchprobiert sind |
|---|---|---|
| 6 Zeichen, Kleinbuchstaben + Ziffern | 2 · 10⁹ | ca. 25 Tage |
| 8 Zeichen, nur Kleinbuchstaben | 2 · 10¹¹ | ca. 7 Jahre |
| 4 zufällige Wörter (Wortliste mit 7.776 Wörtern) | 4 · 10¹⁵ | ca. 100.000 Jahre |
| 10 Zeichen, Groß-/Kleinbuchstaben + Ziffern | 8 · 10¹⁷ | ca. 27 Mio. Jahre |

### 4.9 Arbeitsspeicher

Im entsperrten Zustand müssen Inhalte und Datenschlüssel im Arbeitsspeicher liegen, sonst kann die App nicht arbeiten. Ziel ist, **Schlüsselmaterial** möglichst kurz und geschützt zu halten und zu verhindern, dass Speicherinhalte **auf die Festplatte** des Rechners gelangen.

| Maßnahme | Technik |
|---|---|
| SQLCipher-Speicherschutz | `PRAGMA cipher_memory_security = ON`: SQLCipher überschreibt freigegebenen Speicher und sperrt seine Speicherseiten gegen Auslagerung (`mlock` / `VirtualLock`) (Spike: Auswirkung auf Leistung und Grenzen) |
| Schlüssel außerhalb des Dart-Speichers | DEK und KEK in nativem Speicher (per FFI) statt als Dart-Objekte; gegen Auslagerung gesperrt, nach Gebrauch überschrieben |
| Passwort kurzlebig | Das Eingabefeld hält das Passwort zwangsläufig als Dart-String; nach dem Ableiten wird das Feld sofort geleert und nicht weiter referenziert |
| Sperren vor Ruhezustand / Bildschirmsperre | Windows: `WM_POWERBROADCAST` (`PBT_APMSUSPEND`), `WTS_SESSION_LOCK`. Linux: logind-Signale `PrepareForSleep` / `Lock` (Spike). App sperrt, wenn in den Einstellungen aktiviert (§4.4) |
| Keine Absturz-Speicherauszüge | Eigene Absturzbehandlung im Runner, damit Windows Error Reporting keinen Speicherauszug ablegt, ohne die Registry zu ändern (Spike) |

**Was bleibt (s. §5.2):** Inhalte in Dart-Objekten bleiben nach dem Sperren im Speicher, bis er überschrieben wird. Wird der Rechner im **entsperrten** Zustand in den Ruhezustand versetzt oder lagert das System aus, können Reste auf die Festplatte gelangen. Ein Administrator oder Schadsoftware kann den Prozessspeicher jederzeit auslesen.

---

## 5. Lücken und Restrisiken

### 5.1 Grundsätzlich
| Risiko | Erklärung | Umgang |
|---|---|---|
| Unverschlüsselt betrieben | Ohne App-Verschlüsselung (und ohne VeraCrypt o. Ä.) liegen alle Inhalte im Klartext. Für Nutzer ohne VeraCrypt ist das der Normalfall, wenn sie nicht aktiv verschlüsseln | Einrichtungsassistent fragt beim ersten Start, ohne Vorauswahl (aktive Entscheidung); Text und Hilfe erklären das Risiko und dass eine spätere Aktivierung auf SSDs Klartextreste hinterlassen kann |
| Schwaches Passwort | Wer `bootstrap.json` hat, kann offline Passwörter durchprobieren; Argon2id bremst nur | Argon2id, Stärkeanzeige mit Empfehlung, Passphrase-Generator (§4.8); keine Pflicht |
| Passwort + Wiederherstellungsschlüssel verloren | Daten sind unwiederbringlich weg | Deutlicher Hinweis bei Aktivierung |

### 5.2 Arbeitsspeicher
| Risiko | Erklärung | Umgang |
|---|---|---|
| Entschlüsselte Daten im RAM | Während die App entsperrt läuft, liegen Inhalte und der DEK im Speicher | nicht vermeidbar |
| Dart kann Speicher nicht zuverlässig löschen | Strings sind unveränderlich, die Speicherbereinigung kopiert Objekte; Passwort und Inhalte bleiben ggf. auch nach dem Sperren im Speicher, bis er überschrieben wird | Schlüssel in nativem Speicher, Passwort kurzlebig (§4.9); Rest dokumentieren |
| Auslagerungsdatei / Ruhezustand | Speicherinhalte können auf die Festplatte des Rechners geschrieben werden | SQLCipher-Speicherschutz, automatisches Sperren vor Ruhezustand (§4.9); dokumentieren |
| Absturzberichte | Windows Error Reporting kann bei einem Absturz Berichte mit Speicherauszug außerhalb des App-Ordners ablegen | eigene Absturzbehandlung (§4.9, Spike) |

### 5.3 Spuren auf fremden Rechnern
| Risiko | Erklärung | Umgang |
|---|---|---|
| Prefetch, UserAssist, Jump Lists, Shell-History | Zeigen, dass MovieTracker gestartet wurde (kein Inhalt) | akzeptiert (App-Name unkritisch) |
| Grafiktreiber-Caches | Treiber legen Shader-Caches ggf. in `%LOCALAPPDATA%` ab (kein Inhalt) | im Spike messen, dokumentieren |
| Fremde Zwischenablage-Manager | Halten sich nicht zwingend an die Hinweise (§4.3), v. a. unter Linux | automatisches Leeren; dokumentieren |
| Jedes Programm kann die Zwischenablage lesen | Solange der Link dort liegt | kurze Leerungszeit; Hinweis |
| Browser-Verlauf und Browser-Sync | Nach „Link öffnen“ außerhalb der Kontrolle der App | Kopieren als Standard, Hinweis beim Öffnen |
| Aufnahmeschutz nur unter Windows | Unter Linux (insb. X11) kann jedes Programm den Bildschirm lesen | dokumentieren |

### 5.4 Dateien und Metadaten
| Risiko | Erklärung | Umgang |
|---|---|---|
| Sichtbare Metadaten | Größe und Änderungszeit von `movietracker.db`; `bootstrap.json` verrät, dass verschlüsselt wird | akzeptiert |
| Klartextreste nach nachträglicher Aktivierung | SSDs/Flash überschreiben gelöschte Blöcke nicht sofort | Hinweis: Verschlüsselung am besten direkt bei der Einrichtung aktivieren |
| Klartext-Exporte | Bei deaktivierter Verschlüsselung | Option „Export mit Passwort schützen“ |

### 5.5 Außerhalb des Schutzumfangs
| Risiko | Warum nicht lösbar |
|---|---|
| Schadsoftware / Keylogger (A6) | Wer auf dem Rechner mitliest, sieht Passwort und Inhalte. Auf fremden Rechnern ist das das größte reale Risiko |
| Manipulierte App (A7) | Wer Schreibzugriff auf den App-Ordner hat, kann die Exe austauschen und z. B. das Passwort mitschneiden; die App kann sich nicht selbst vertrauenswürdig prüfen. Schutz durch einen verschlüsselten Container (z. B. VeraCrypt) oder einen Ort, an den niemand sonst schreiben kann. Ergänzend: Releases mit SHA-256-Prüfsummen (später ggf. signiert), damit Nutzer ihre Kopie prüfen können |
| Abstreitbarkeit | Es ist erkennbar, dass verschlüsselte Daten existieren |

---

## 6. Nachweise (Tests)
- **Portabilitätstests** Linux/Windows: kein Schreibzugriff außerhalb des App-Ordners
- **Klartext-Test**: Canary-Strings → Verschlüsselung an → keine Canaries in `data/`
- **Log-Test**: Canaries tauchen in keinem Log auf
- **Roundtrip-Tests**: Verschlüsselung an/aus/an, Passwort ändern, Wiederherstellungsschlüssel, Export/Import
- **Zwischenablage-Test** (Windows): kopierter Link erscheint nicht im Zwischenablage-Verlauf
- **Manipulations-Test**: veränderte Seite in `movietracker.db` bzw. veränderte `bootstrap.json` → sauberer Fehler, kein Datenverlust

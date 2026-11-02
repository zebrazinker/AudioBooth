# Spec 003 – Apple-Watch-App neu gestalten

| | |
|---|---|
| **Status** | Draft |
| **Datum** | 2026-11-02 |
| **Design** | [AudioBooth Redesign (Canvas)](https://claude.ai/artifact/66Qha78m1FkuVMvqcfvLmT) – Reihe *Apple Watch*: *Start*, *Player*, *Wiedergabe auf*, *Offline* |
| **Abhängigkeit** | Spec 001 (Akzentfarbe, Medientypen, Download-Markierung) – visuell |
| **Plattform** | watchOS 10+ (Deployment Target 10), Liquid-Glass-Optik ab watchOS 26 |
| **Code** | `AudioBooth Watch App/`, `AudioBoothWatchWidget/`, iPhone-Seite `AudioBooth/Services/WatchConnectivityManager.swift` |

**Leitprinzipien:** Ein Blick, ein Tap. Was gerade läuft, steht oben. Offline ist ein normaler Zustand, kein Fehler. Orientierung an Apple Podcasts / Spotify auf der Watch: große „Jetzt läuft“-Kachel, kompakte Listen, Steuerung mit großen Zielen.

---

## 1. Ist-Zustand

| Bereich | Heute | Datei |
|---|---|---|
| Start | Abschnitte „Downloading“, „Continue Listening“, „Available Offline“, „More“ + Refresh-Button unten | `Screens/ContinueListening/ContinueListeningView.swift` |
| Player (Uhr) | Cover, Marquee-Titel, Zeiten, Toolbar unten: −30 / Play mit Fortschrittsring / +30; „…“-Sheet: Kapitel, Auf iPhone abspielen, Tempo, Download | `Screens/Player/PlayerView.swift`, `PlayerOptions/` |
| Player (iPhone) | System-`NowPlayingView` + Tempo | `Screens/RemotePlayer/RemotePlayerView.swift` |
| Player-Zugang | Toolbar-Icon oben rechts wechselt zwischen ⌚ und 📱 | `Screens/ContentView.swift` |
| Daten | Weiterhören, Abschnitte, Fortschritt über WatchConnectivity vom iPhone; Wiedergabe direkt vom Server oder lokal | `WatchConnectivityManager.swift`, `Services/DownloadManager.swift` |
| Komplikationen | Rund, Rechteck, Ecke mit Fortschritt | `AudioBoothWatchWidget/Complications/` |

## 2. Probleme

| # | Problem | Ursache |
|---|---|---|
| W1 | Geladene Titel erscheinen **doppelt** („Continue Listening“ und „Available Offline“) | `ContinueListeningViewModel.updateAllRows()` filtert nicht |
| W2 | Ohne Daten **endloser Ladekreis** statt Hinweis | `ContinueListeningView` zeigt `ProgressView`, wenn alle Listen leer |
| W3 | Refresh / „More“ ohne iPhone-Verbindung tun **still nichts** | `refreshContinueListening`, `fetchSection` brechen bei `!isReachable` nur mit Log ab |
| W4 | Fest codierte Navy-Farben, passen nicht zum App-Theme | `Color(red: 0.1, green: 0.1, blue: 0.2)` in Row und Start |
| W5 | Hörbücher und Podcast-Folgen gemischt in einer Liste | Start |
| W6 | Player-Icon oben rechts wechselt (⌚/📱), unklar; Tap auf 📱 beendet still den Uhr-Player | `ContentView.toolbar`, `playerManager.clearCurrent()` |
| W7 | Refresh nur als Button am Listenende | Start |
| W8 | „by \(author)“ hart englisch; 6 von 37 Strings ohne Übersetzung | `PlayerView.content`, `Localizable.xcstrings` |
| W9 | Skip-Symbole zeigen immer „30“, obwohl das Intervall konfigurierbar ist | `gobackward.30` / `goforward.30` vs. `skipForwardInterval` |

## 3. User Stories

| ID | Als Hörer möchte ich … | damit … |
|---|---|---|
| US-1 | auf der Startseite sofort sehen, was läuft, und es mit einem Tap steuern | ich nicht suchen muss |
| US-2 | jeden Titel nur einmal sehen, mit Markierung, ob er auf der Uhr liegt | die Liste kurz bleibt |
| US-3 | Hörbücher und Podcast-Folgen getrennt sehen | ich mich schneller zurechtfinde |
| US-4 | wählen, ob auf der Uhr oder dem iPhone abgespielt wird | ich nicht raten muss, wo es läuft |
| US-5 | ohne iPhone verständlich informiert werden und meine geladenen Titel hören | ich beim Laufen nicht hängen bleibe |
| US-6 | die Bedienung in meiner Sprache und mit meinem Skip-Intervall sehen | sie korrekt ist |

## 4. Funktionale Anforderungen

### FR-1 Startseite (Design: *Watch · Start*)
- FR-1.1 Titel „AudioBooth“ in Akzentfarbe, keine Toolbar-Icons für den Player mehr (W6).
- FR-1.2 **„Jetzt läuft“-Kachel** (nur wenn etwas aktiv ist, Uhr **oder** iPhone):
  - Hintergrund: Verlauf aus Cover-Farbe (gleiche Logik wie Spec 002 FR-1.2, Wert vom iPhone mitgeliefert oder lokal berechnet)
  - Zeile „Jetzt auf der Uhr“ / „Jetzt auf dem iPhone“ mit Equalizer-Symbol
  - Cover 44 pt, Titel (2 Zeilen), Restzeit, Fortschrittsbalken, runder Play/Pause-Button
  - Tap auf Kachel → Player; Tap auf Button → nur Play/Pause
- FR-1.3 **Weiterhören**: nur Hörbücher, jeder Titel **einmal** (W1). Liegt er auf der Uhr: ✓ in Akzentfarbe vor der Restzeit. Der aktuell laufende Titel erscheint nicht zusätzlich.
- FR-1.4 **Neue Folgen**: Podcast-Folgen getrennt (W5), Podcast-Name klein über dem Titel.
- FR-1.5 **Mediathek**: Zeilen „Auf der Uhr (n)“, „Hörbücher“, „Podcasts“ → Unterlisten (bisherige `homeSections` und `SectionDetailView`). „Auf der Uhr“ ersetzt den Abschnitt „Available Offline“.
- FR-1.6 „Wird geladen“ nur noch als Fortschrittsring am Cover der betreffenden Zeile, kein eigener Abschnitt.
- FR-1.7 Aktualisieren per **Pull-to-Refresh** (`.refreshable`); Refresh-Button entfällt (W7).
- FR-1.8 Zeilen: Hintergrund `#1C1C1E`-Äquivalent aus Theme, Eckradius 16, Cover 38 pt, Radius 7 (W4).

### FR-2 Leere und Offline-Zustände (Design: *Watch · Offline*)
- FR-2.1 Keine Daten und iPhone nicht erreichbar → Zustand „iPhone nicht erreichbar“: Symbol, Titel, Text „Geladene Titel kannst du weiter hören.“, Buttons **„Auf der Uhr (n)“** (nur wenn n > 0) und **„Erneut versuchen“** (W2).
- FR-2.2 Keine Daten, iPhone erreichbar, Server leer → „Noch nichts gehört“ + Hinweis „Starte ein Hörbuch auf dem iPhone“.
- FR-2.3 Laden ≤ 10 s zeigt `ProgressView`; danach Wechsel in FR-2.1/2.2.
- FR-2.4 Gecachte Daten vorhanden, iPhone nicht erreichbar → normale Startseite + dezente Zeile oben „Offline · zuletzt aktualisiert ‹Zeit›“.
- FR-2.5 Pull-to-Refresh / „Mediathek“-Unterlisten ohne Verbindung → kurze Meldung „iPhone nicht erreichbar“ statt stillem Abbruch (W3).

### FR-3 Player (Design: *Watch · Player*)
- FR-3.1 Hintergrund: Verlauf aus Cover-Farbe (FR-1.2).
- FR-3.2 Oben: Schließen (Chevron runter) links, „…“ rechts – System-Toolbar.
- FR-3.3 Titel (1 Zeile, Marquee wie bisher, `marqueeLoopMode` bleibt), darunter „Kapitel ‹n› · noch ‹Kapitel-Restzeit›“.
- FR-3.4 Steuerung mittig: Zurück-Skip · Play/Pause (68 pt, weiß, Fortschrittsring wie bisher) · Vor-Skip. Skip-Symbole passend zum eingestellten Intervall (`gobackward.10/15/30/45/60`, Fallback `gobackward`) (W9).
- FR-3.5 Dünner Gesamt-Fortschrittsbalken unter der Steuerung.
- FR-3.6 Unten drei Pillen: **Tempo** („1,2×“) · **Wiedergabeort** („Uhr“ / „iPhone“) · **Kapitel**.
- FR-3.7 „…“-Menü behält: Download / Von der Uhr entfernen, Lauftext-Modus. „Auf iPhone abspielen“ wandert in FR-4.
- FR-3.8 Digitale Krone = Lautstärke (bestehend); Doppeltipp-Geste = Play/Pause (bestehend, watchOS 11+).
- FR-3.9 Spielt das iPhone: derselbe Player-Aufbau (statt System-`NowPlayingView`), gesteuert per WatchConnectivity; Pille zeigt „iPhone“.

### FR-4 Wiedergabeort (Design: *Watch · Wiedergabe auf*)
- FR-4.1 Liste mit zwei Optionen: **Apple Watch** (Untertitel: verbundene Kopfhörer bzw. „Kopfhörer verbinden“) und **iPhone** (Untertitel: „Verbunden“ / „Nicht erreichbar“, dann deaktiviert).
- FR-4.2 Auswahl übergibt die Position: Uhr → iPhone via `playOnPhone(_:currentTime:)`; iPhone → Uhr startet lokalen Player an der vom iPhone gemeldeten Position.
- FR-4.3 Der bisher laufende Player wird **pausiert, nicht verworfen**; Ersatz für `clearCurrent()` im Toolbar-Pfad (W6).
- FR-4.4 Keine Kopfhörer verbunden und „Apple Watch“ gewählt → System-Routenauswahl (`AVRoutePickerView`-Äquivalent) öffnen.

### FR-5 Lokalisierung & Theme
- FR-5.1 „by \(author)“ → lokalisierter String „von %@“; alle 37+ Strings in allen 8 Sprachen (W8).
- FR-5.2 Farben aus einem Watch-Theme (Akzent, Hintergrund Zeile, sekundärer Text); keine `Color(red:green:blue:)` in Views (W4).
- FR-5.3 Ab watchOS 26 System-Glass für Toolbar und Pillen; darunter `.ultraThinMaterial`.

### FR-6 Komplikationen
- FR-6.1 Bestehende Familien bleiben. Akzentfarbe aus Theme; Rechteck zeigt Titel + „noch ‹Zeit›“ + Balken.
- FR-6.2 Tap öffnet direkt den Player des aktuellen Titels (statt Startseite).

## 5. Nicht-funktionale Anforderungen

- NFR-1 Startseite aus Cache sichtbar < 500 ms nach App-Start.
- NFR-2 Alle Tap-Ziele ≥ 44 pt (Pillen 32 pt hoch, aber ≥ 44 pt breit).
- NFR-3 VoiceOver: Kachel „Jetzt läuft“ als ein Element mit Aktionen Play/Pause und Öffnen; ✓-Markierung „Auf der Uhr“.
- NFR-4 Dynamic Type bis xxxLarge: Titel umbrechen statt abschneiden.
- NFR-5 Kein zusätzlicher Akkuverbrauch: keine neuen Timer/Polling; Farbextraktion einmal pro Cover, gecacht.

## 6. Non-Goals

- Bibliothek komplett auf der Watch durchsuchen (Suche).
- E-Books auf der Watch.
- Standalone-Login ohne iPhone.
- Neue WatchConnectivity-Nachrichten außer für FR-3.9 (iPhone-Steuerung) und optional Cover-Farbe.

## 7. Akzeptanzkriterien (Gherkin)

```gherkin
Szenario: Keine Dubletten
  Angenommen "Eine kurze Geschichte von fast allem" liegt auf der Uhr und ist in "Weiterhören"
  Wenn ich die Startseite öffne
  Dann erscheint der Titel genau einmal unter "Weiterhören" mit ✓
  Und unter Mediathek steht "Auf der Uhr (n)" inklusive dieses Titels

Szenario: Jetzt läuft
  Angenommen auf dem iPhone läuft "Das Zeitalter der Unschärfe"
  Wenn ich die Watch-App öffne
  Dann zeigt die oberste Kachel "Jetzt auf dem iPhone" mit Titel und Fortschritt
  Wenn ich den Pause-Button der Kachel tippe
  Dann pausiert das iPhone

Szenario: Wiedergabeort wechseln
  Angenommen auf der Uhr läuft ein Titel bei 1:02:10
  Wenn ich im Player "Uhr" tippe und "iPhone" wähle
  Dann spielt das iPhone ab 1:02:10
  Und der Uhr-Player ist pausiert, nicht beendet
  Und die Pille zeigt "iPhone"

Szenario: Offline ohne Daten
  Angenommen die Watch hat keine gecachten Daten und das iPhone ist nicht erreichbar
  Wenn ich die App öffne
  Dann sehe ich nach höchstens 10 Sekunden "iPhone nicht erreichbar"
  Und keinen endlosen Ladekreis

Szenario: Offline mit Downloads
  Angenommen 3 Titel liegen auf der Uhr und das iPhone ist nicht erreichbar
  Dann kann ich über "Auf der Uhr (3)" einen Titel abspielen

Szenario: Refresh ohne iPhone
  Angenommen das iPhone ist nicht erreichbar
  Wenn ich die Startseite nach unten ziehe
  Dann erscheint "iPhone nicht erreichbar"

Szenario: Skip-Intervall
  Angenommen das Skip-Intervall ist auf 15 Sekunden eingestellt
  Dann zeigen die Skip-Buttons "15"

Szenario: Sprache
  Angenommen die Uhr ist auf Deutsch eingestellt
  Dann steht im Player "von Tobias Hürter" statt "by Tobias Hürter"
```

## 8. Technischer Plan

| # | Datei(en) | Änderung |
|---|---|---|
| T1 | `ContinueListeningViewModel.swift` | Dubletten entfernen, `nowPlaying`, Trennung `books`/`episodes`, `onWatchCount` (W1, W5) |
| T2 | `ContinueListeningView.swift` | Neues Layout FR-1, `.refreshable`, Zustände FR-2 mit Timeout (W2, W7) |
| T3 | neu `Components/NowPlayingCard.swift` | Kachel FR-1.2 |
| T4 | `ContinueListeningRow.swift` | Theme-Farben, ✓/Ring statt orangem Text (W4) |
| T5 | `WatchConnectivityManager.swift` (Watch) | `isReachable` publizieren; Refresh/Fetch liefern Ergebnis/Fehler zurück (W3); Remote-Steuerbefehle (FR-3.9) |
| T6 | `PlayerView.swift`, neu `Components/PlayerPills.swift` | Layout FR-3, Skip-Symbole nach Intervall (W9), lokalisierter Autor (W8) |
| T7 | neu `Screens/Output/OutputPickerView.swift`; `ContentView.swift` | FR-4; Toolbar-Icon und `clearCurrent()`-Pfad entfernen (W6) |
| T8 | `RemotePlayerView.swift` | Durch gemeinsamen Player mit Remote-Model ersetzen (FR-3.9) |
| T9 | iPhone `Services/WatchConnectivityManager.swift` | Optional Cover-Farbe in `watchBookItem` mitschicken; Remote-Befehle entgegennehmen |
| T10 | `Localizable.xcstrings` (Watch) | Fehlende Übersetzungen, „von %@“ (W8) |
| T11 | `AudioBoothWatchWidget/Complications/*` | Theme-Farbe, Deep-Link in Player (FR-6) |

**Aufwand:** ca. 3–4 Tage. **Quick Wins vorab (≈ 2 h):** T1 (nur Dubletten), T2 (nur Leerzustand), T10, W9.

## 9. Testplan

- Unit: Dubletten-Logik, Zustandsautomat Start (lädt / leer / offline / Daten), Skip-Symbol-Mapping.
- Previews/Snapshots: 41 mm, 45 mm, 49 mm (Ultra); alle Zustände FR-2; Deutsch + längste Sprache.
- Manuell auf Gerät: Flugmodus am iPhone, Wechsel Uhr ↔ iPhone während Wiedergabe, Kopfhörer getrennt, Komplikations-Tap, VoiceOver.

## 10. Offene Fragen

1. Soll „Jetzt läuft“ auch einen zuletzt pausierten Titel zeigen (z. B. bis 24 h)? (Annahme: ja)
2. Ersetzen wir die System-`NowPlayingView` für iPhone-Wiedergabe wirklich (FR-3.9, mehr Aufwand) oder behalten sie und passen nur den Einstieg an?
3. Cover-Farbe vom iPhone mitschicken (weniger Last auf der Uhr) oder lokal berechnen?

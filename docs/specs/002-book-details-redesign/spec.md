# Spec 002 – Hörbuch-Detailseite neu gestalten

| | |
|---|---|
| **Status** | Draft |
| **Datum** | 2026-11-02 |
| **Design** | [AudioBooth Redesign (Canvas)](https://claude.ai/artifact/66Qha78m1FkuVMvqcfvLmT) – Screen *Hörbuch-Details* |
| **Abhängigkeit** | Spec 001 (Tab-Navigation, Liquid Glass, Akzentfarbe) – visuell; technisch unabhängig umsetzbar |
| **Plattform** | iOS / iPadOS 18+, Liquid Glass ab iOS 26 |
| **Code** | `Screens/BookDetails/` (`BookDetailsView.swift`, `BookDetailsViewModel.swift`, `Components/*`) |

**Leitprinzipien:** Eine klare Hauptaktion · das Wichtigste oben · Details auf Abruf.

---

## 1. Problem

Ist-Zustand (`BookDetailsView.swift`):

| # | Beobachtung | Ursache im Code |
|---|---|---|
| P1 | Cover-Hintergrund endet mit harter Kante | `ParallaxHeader` + `LazyImage … .blur(radius: 5).opacity(0.3)` ohne Übergang |
| P2 | Play-Button wirkt „kaputt“ (hellblauer Block links) | `actionBackground(progress:)` füllt den Button als Fortschrittsbalken |
| P3 | Autoren und Sprecher belegen je einen eigenen Abschnitt mit Chips | `headerSection`: „Authors“ / „Narrators“ |
| P4 | Metadaten als lange Liste mit fetten Labels | `MetadataSection` |
| P5 | Fortschritt doppelt (Cover-Badge, Button, `ProgressCard`) | `ProgressCard`, `Cover.progress`, `actionBackground` |
| P6 | Beschreibung blendet aus, ohne „Mehr“ sichtbar | `frame(maxHeight: 180)` + Fade |
| P7 | Kapitelliste endlos, alle Einträge gleich gewichtet; Tracks/Sessions gleichrangig als Segmente | `ContentTab` `.chapters/.tracks/.ebooks/.sessions` |
| P8 | Zurück-/Mehr-Buttons überlagern Text beim Scrollen ohne Lesbarkeitsschutz | Eigene Buttons statt Toolbar |

## 2. Ziel

Eine ruhige, moderne Detailseite: großes Cover, **eine** Hauptaktion, Fortschritt **einmal**, kompakte Fakten, Kapitel um die aktuelle Position, alles Weitere auf Abruf.

## 3. User Stories

| ID | Als Hörer möchte ich … | damit … |
|---|---|---|
| US-1 | sofort sehen, wo ich stehe (%, Kapitel, Restzeit) | ich nicht suchen muss |
| US-2 | mit einem Tap weiterhören | die Hauptaktion eindeutig ist |
| US-3 | Autor und Sprecher auf einen Blick sehen und antippen | ich zu deren Werken komme |
| US-4 | Dauer, Jahr, Sprache, Größe kompakt sehen | ich nicht scrollen muss |
| US-5 | die Kapitel rund um meine Position sehen und zu allen springen können | die Liste übersichtlich bleibt |
| US-6 | Tracks, Sitzungen, Lesezeichen weiterhin finden | keine Funktion verloren geht |

## 4. Funktionale Anforderungen

### FR-1 Header
- FR-1.1 Hintergrund: Verlauf von der **dominanten Cover-Farbe** (oben) zur Seitenfarbe (unten), Höhe ≈ 620 pt, kein harter Übergang. Ersetzt Blur-Bild (P1).
- FR-1.2 Farbe wird aus dem geladenen Cover berechnet (z. B. `CIAreaAverage` auf verkleinertem Bild), gecacht pro `bookID`; Fallback: neutrales Grau. Helligkeit wird so begrenzt, dass weißer Text ≥ 4.5:1 Kontrast hat.
- FR-1.3 Cover zentriert, 230 pt, Eckradius 14, weicher Schatten. Tap → Vollbild (bestehend). **Kein** Fortschritts-Badge auf dem Cover (P5).
- FR-1.4 Darunter zentriert: Titel (26 pt bold), Untertitel (15 pt, sekundär), Zeile „von ‹Autor› · gelesen von ‹Sprecher›“ – Namen als Links in Akzentfarbe (P3). Mehrere Namen kommagetrennt, ab 3 „und X weitere“.
- FR-1.5 Zurück und „…“ als System-Toolbar-Items (iOS 26: automatisch Glass; P8). Navigation-Bar-Titel erscheint beim Scrollen über den Titel hinaus.

### FR-2 Aktionen & Fortschritt
- FR-2.1 Primärbutton (54 pt, Kapsel, volle Breite minus Nebenbuttons, hell auf dunkel): Label nach bestehender `playButtonText`-Logik (Abspielen / Weiterhören / Pause / Erneut hören). **Keine Fortschrittsfüllung** im Button (P2) – `actionBackground` entfällt.
- FR-2.2 Rechts zwei runde Glass-Buttons (54 pt): **Download** (Zustände: herunterladen / lädt mit Ring / geladen ✓) und **Zur Warteschlange**.
- FR-2.3 Hat das Item ein E-Book: zusätzlicher Sekundär-Button „Lesen“ unter dem Primärbutton (statt Segment `.ebooks`).
- FR-2.4 Darunter **eine** Fortschrittszeile: 4-pt-Balken in Akzentfarbe, darunter links „‹%› · Kapitel ‹n›“, rechts „noch ‹Restzeit›“. Nicht gestartet → ausgeblendet; beendet → „Gehört am ‹Datum›“.
- FR-2.5 `ProgressCard` entfällt; „Begonnen am“ wandert in Informationen (FR-6).

### FR-3 Fakten-Leiste
- FR-3.1 Vier gleich breite Spalten mit Trennlinien: **Dauer · Jahr · Sprache · Größe** (Label klein/caps, Wert 17 pt bold, Einheit klein) (P4).
- FR-3.2 Fehlt ein Wert, wird die Spalte weggelassen (Grid passt sich an, min. 2).

### FR-4 Beschreibung & Genres
- FR-4.1 Beschreibung max. 4 Zeilen, darunter Button „Mehr lesen“ / „Weniger“ (P6). HTML wie bisher bereinigt.
- FR-4.2 Genres und Tags als kleine Chips (32 pt), antippbar → gefilterte Bibliothek.

### FR-5 Kapitel
- FR-5.1 Abschnitt „Kapitel“ mit Link „Alle ‹n›“ rechts.
- FR-5.2 Inline werden **5 Kapitel** gezeigt: 2 vor dem aktuellen, das aktuelle, 2 danach (am Anfang/Ende entsprechend verschoben).
- FR-5.3 Zustände: gehört = Haken + sekundäre Textfarbe; aktuell = Play-Symbol, Akzentfarbe, leicht getönte Zeile; offen = Nummer.
- FR-5.4 Tap auf ein Kapitel springt dorthin (bestehendes Verhalten).
- FR-5.5 „Alle“ öffnet eine Sheet-/Push-Liste mit allen Kapiteln, automatisch zum aktuellen gescrollt (`ScrollViewReader`).
- FR-5.6 Segment-Picker Chapters/Tracks/eBooks/Sessions **entfällt** (P7).

### FR-6 Informationen
- FR-6.1 Gruppierte Liste (abgerundete Karte): Verlag, Begonnen, Genre, Serie (falls vorhanden), Dateien/Tracks, ISBN/ASIN (falls vorhanden).
- FR-6.2 Inhalte aus bisheriger `MetadataSection`, ohne Icons, Label links sekundär, Wert rechts.

### FR-7 „…“-Menü
- FR-7.1 Enthält: **Tracks**, **Hörsitzungen**, Lesezeichen, Zu Sammlung hinzufügen, Als gehört markieren / Fortschritt zurücksetzen, Teilen, Download löschen.
- FR-7.2 Tracks und Hörsitzungen öffnen die bestehenden `TracksContent` / `SessionsContent` als Push-Ziel.

### FR-8 iPad / breite Layouts
- FR-8.1 Bestehendes Zwei-Spalten-Layout (`simpleCover`) bleibt: links Cover + Aktionen + Fortschritt mit Farbverlauf, rechts Fakten, Beschreibung, Kapitel, Informationen.

## 5. Nicht-funktionale Anforderungen

- NFR-1 Farbextraktion off-main, < 20 ms auf iPhone 12; Seite rendert zuerst mit Fallback-Farbe, Übergang animiert (0,25 s).
- NFR-2 Kontrast aller Texte ≥ 4.5:1 – auch auf hellen Covern (FR-1.2).
- NFR-3 VoiceOver: Fortschrittszeile als ein Element („10 Prozent, Kapitel 17, noch 11 Stunden 31 Minuten“); runde Buttons mit `accessibilityLabel`.
- NFR-4 Dynamic Type bis AX3: Fakten-Leiste bricht auf 2×2 um.
- NFR-5 „Bewegung reduzieren“: kein Parallax.

## 6. Non-Goals

- Podcast-Detailseite (`Screens/PodcastDetails`) – eigene Spec, gleiche Bausteine.
- E-Book-Reader, Player.
- Neue Server-Daten.

## 7. Akzeptanzkriterien (Gherkin)

```gherkin
Szenario: Fortschritt nur einmal
  Angenommen ein Hörbuch mit 10 % Fortschritt in Kapitel 17
  Wenn ich die Detailseite öffne
  Dann sehe ich genau eine Fortschrittsanzeige "10 % · Kapitel 17" und "noch 11 Std. 31 Min."
  Und der Button "Weiterhören" hat keine Fortschrittsfüllung
  Und das Cover hat kein Prozent-Badge

Szenario: Nicht gestartet
  Angenommen ein Hörbuch ohne Fortschritt
  Dann heißt der Button "Abspielen"
  Und es wird keine Fortschrittszeile angezeigt

Szenario: Header-Verlauf
  Wenn die Detailseite geladen ist
  Dann geht der Hintergrund ohne sichtbare Kante von der Cover-Farbe in die Seitenfarbe über

Szenario: Kapitel um die aktuelle Position
  Angenommen das aktuelle Kapitel ist 17 von 42
  Dann sehe ich inline die Kapitel 15 bis 19
  Und Kapitel 17 ist hervorgehoben
  Wenn ich "Alle 42" tippe
  Dann öffnet sich die vollständige Liste, gescrollt zu Kapitel 17

Szenario: Tracks und Sitzungen erreichbar
  Wenn ich auf "…" tippe
  Dann sehe ich "Tracks" und "Hörsitzungen"

Szenario: Autor antippen
  Wenn ich in der Zeile "von Tobias Hürter" auf den Namen tippe
  Dann öffnet sich die Autorenseite

Szenario: Helles Cover
  Angenommen ein überwiegend weißes Cover
  Dann ist der Titel trotzdem mit Kontrast ≥ 4.5:1 lesbar
```

## 8. Technischer Plan

| # | Datei(en) | Änderung |
|---|---|---|
| T1 | neu `Components/CoverTint.swift` | Dominante Farbe aus Cover (`CIAreaAverage`), Helligkeit clampen, Cache pro `bookID` |
| T2 | `BookDetailsView.swift` (Header, `ParallaxHeader`-Nutzung) | Verlauf statt Blur-Bild; Cover ohne Badge; zentrierte Titelzeile mit Autor/Sprecher-Links; Toolbar-Items |
| T3 | `BookDetailsView.swift` (`actionButtons`, `actionBackground`) | Neuer Primärbutton + 2 runde Buttons; `actionBackground` entfernen; „Lesen“-Button bei E-Book |
| T4 | neu `Components/ProgressLine.swift`; `ProgressCard` entfernen | Fortschrittszeile (FR-2.4) |
| T5 | neu `Components/FactsBar.swift`; `MetadataSection` → `InfoSection` | FR-3, FR-6 |
| T6 | `ChaptersContent` | Modus „Ausschnitt“ (5 um aktuell) + „Alle“-Ziel mit Autoscroll; Zustände FR-5.3 |
| T7 | `BookDetailsView.Model.ContentTab` | Segment-Picker entfernen; Tracks/Sessions ins „…“-Menü als Push |
| T8 | `Localizable.xcstrings` | Neue Strings |

**Aufwand:** ca. 2–3 Tage inkl. iPad-Layout und Test.

## 9. Testplan

- Unit: `CoverTint` (helles/dunkles/fehlendes Cover, Kontrast), Kapitel-Ausschnitt (Anfang, Mitte, Ende, < 5 Kapitel).
- Snapshot: nicht gestartet / in Arbeit / beendet / mit E-Book / ohne Metadaten; hell & dunkel; AX3.
- Manuell: iPad, VoiceOver, „Bewegung reduzieren“, Offline (lokales Buch).

## 10. Offene Fragen

1. Farbverlauf immer aus dem Cover oder optional neutral (Einstellung)?
2. Lesezeichen: im „…“-Menü (Annahme) oder eigener Abschnitt unter Kapitel?
3. Sitzungen ganz ausblenden für Nutzer, die sie nie öffnen?

# Spec 001 – Tab-Navigation: Hörbücher, Podcasts & E-Books als feste Tabs

| | |
|---|---|
| **Status** | Draft v2 |
| **Datum** | 2026-11-02 |
| **Design** | [AudioBooth Redesign (Canvas)](https://claude.ai/artifact/66Qha78m1FkuVMvqcfvLmT) – Screens *Start*, *Hörbücher*, *Podcasts*, *E-Books* |
| **Plattform** | iOS / iPadOS 18+, Liquid Glass ab iOS 26 |
| **Nicht betroffen** | CarPlay, watchOS (siehe Non-Goals) |

**Leitprinzipien:** modern (Liquid Glass) · einfach · übersichtlich. Eine feste Tab-Leiste, ein Ort pro Medientyp, keine versteckten Menüs für die Hauptnavigation.

---

## 1. Problem

Heute ist die Navigation **bibliotheksabhängig**:

- Die aktive Bibliothek wird über ein Menü oben links gewählt (`HomePage.serverMenuToolbarItem`, `Screens/Home/HomePage.swift`).
- Die Tab-Leiste (`ContentView.swift`) baut sich je nach `libraries.current.mediaType` komplett um:
  - Buch-Bibliothek: Home · Library · Collections · Downloads · Suche
  - Podcast-Bibliothek: Home · Latest · Library · Downloads · Suche
- E-Books liegen in einer eigenen Buch-Bibliothek und sind nur über dasselbe Menü erreichbar.
- Folge: Tabs „springen“, gleiche Labels („Library“) bedeuten Unterschiedliches, Medienwechsel braucht drei Taps in einem versteckten Menü.

## 2. Ziel

Hörbücher, Podcasts und E-Books sind **gleichwertige, feste Tabs**. Die Tab-Leiste ändert sich nie. Der Bibliothekswechsel oben links entfällt.

```
┌───────────────────────────────────────────┐  ┌────┐
│ Start │ Hörbücher │ Podcasts │ E-Books    │  │ 🔍 │
└───────────────────────────────────────────┘  └────┘
```

## 3. User Stories

| ID | Als … | möchte ich … | damit … |
|---|---|---|---|
| US-1 | Nutzer | mit **einem Tap** zwischen Hörbüchern, Podcasts und E-Books wechseln | ich nicht ins Menü oben links muss |
| US-2 | Nutzer | dass die Tab-Leiste immer gleich aussieht | ich mich auf die Position der Tabs verlassen kann |
| US-3 | Nutzer | Serien, Autoren, Sprecher und Sammlungen im jeweiligen Medien-Tab finden | es keine Extra-Tabs braucht |
| US-4 | Podcast-Hörer | neueste Folgen und abonnierte Podcasts im Podcasts-Tab sehen | es keinen eigenen „Latest“-Tab braucht |
| US-5 | Nutzer | Downloads, Server und Einstellungen über einen Profil-Button erreichen | die Tab-Leiste schlank bleibt |
| US-6 | Nutzer mit eigener Bibliotheks-Benennung | festlegen, welche Buch-Bibliothek Hörbücher und welche E-Books enthält | die Tabs richtig befüllt sind |

## 4. Funktionale Anforderungen

### FR-1 Tab-Leiste
- FR-1.1 Tabs (fix, in dieser Reihenfolge): **Start · Hörbücher · Podcasts · E-Books**, plus **Suche** als separater Such-Tab.
- FR-1.2 **Downloads ist kein Tab mehr** (Platz für E-Books; max. 5 Elemente auf dem iPhone). Downloads erreichbar über:
  - Profil-Menü → „Downloads“ (bisherige `DownloadsRootPage`)
  - Filter „Nur geladene“ in jedem Medien-Tab (FR-3.4)
- FR-1.3 Tabs sind unabhängig von `libraries.current` sichtbar, sobald ein Server authentifiziert ist.
- FR-1.4 Existiert für einen Medientyp keine Bibliothek, wird dessen Tab ausgeblendet.
- FR-1.5 Erneutes Tippen auf den aktiven Tab poppt zum Root / scrollt nach oben (bestehendes `onTabItemTapped()`).
- FR-1.6 Navigations-State (Pfad, Scrollposition, Segment) jedes Tabs bleibt beim Wechsel erhalten.

### FR-2 Medientyp ↔ Bibliothek
Audiobookshelf kennt nur `MediaType.book` und `.podcast` (`API/.../Models/Library.swift`). E-Books brauchen daher eine App-seitige Zuordnung.

- FR-2.1 Neuer App-Typ `MediaTab { audiobooks, podcasts, ebooks }`.
- FR-2.2 Podcast-Bibliotheken → `podcasts` (automatisch).
- FR-2.3 Buch-Bibliotheken → standardmäßig `audiobooks`; Name enthält „ebook“ / „e-book“ / „bücher“ ohne „hör“ (case-insensitive) → `ebooks`.
- FR-2.4 Zuordnung überschreibbar unter Einstellungen → „Bibliotheken“ (pro Server persistiert in `server.storage`).
- FR-2.5 Mehrere Bibliotheken pro Tab: Picker „Bibliothek: ‹Name› ▾“ unter dem Titel; bei genau einer entfällt er.
- FR-2.6 **Phase 1:** Tab-Wechsel setzt `libraries.current` auf die zuletzt genutzte Bibliothek dieses Tabs (pro Tab persistiert).
- FR-2.7 Tab-Wechsel darf **nicht** den Bild-Cache leeren. Heute: `ImagePipeline.shared.cache.removeAll()` im Setter von `LibrariesService.current` → nur noch beim Serverwechsel.

### FR-3 Hörbücher-Tab
- FR-3.1 Großer Titel „Hörbücher“, rechts Glass-Button „Sortieren/Filtern“.
- FR-3.2 Segmented Control: **Titel · Serien · Autoren · Sammlungen** (Sprecher als Filter unter „Autoren“). Bestehender Picker in `LibraryRootPage` wird erweitert; `CollectionsRootPage` als Segment eingebettet.
- FR-3.3 Quadratisches 3er-Grid, Fortschritt als 3-px-Linie am Cover-Unterrand.
- FR-3.4 Filter-Menü enthält „Nur geladene“.

### FR-4 Podcasts-Tab
- FR-4.1 Großer Titel „Podcasts“, Glass-Button „+“ (falls Berechtigung).
- FR-4.2 Segmented Control: **Neueste Folgen · Abonniert** (`LatestView` / `PodcastsRootPage`).
- FR-4.3 Horizontale Reihe abonnierter Podcasts mit Badge „neue Folgen“.
- FR-4.4 Folgen als Liste: Podcast · Datum, Titel, 2 Zeilen Beschreibung, Pill „▶ Dauer“, Download, Mehr.

### FR-5 E-Books-Tab
- FR-5.1 Großer Titel „E-Books“, Glass-Button „Sortieren/Filtern“.
- FR-5.2 Segmented Control: **Titel · Serien · Autoren · Sammlungen** (gleiche Komponente wie FR-3.2).
- FR-5.3 Abschnitt „Weiterlesen“ (Fortschritt in %) über dem Grid.
- FR-5.4 Grid im Buchformat 2:3; Tap öffnet direkt den Reader (`EbookReaderView`), Long-Press die Details.

### FR-6 Start-Tab
- FR-6.1 Datum + Begrüßung („Hallo, ‹Name›“) als großer Titel.
- FR-6.2 Rechts: Profil-Button (Avatar mit Status-Punkt gem. `connectionStatusColor`). Menü: Downloads, Server-URL primär/alternativ, Server-Details, Server verwalten, Einstellungen. **Keine Bibliotheksliste.**
- FR-6.3 Phase 1: Inhalte aus `libraries.current`. Phase 2: gemischt aus allen Tabs (§7).

### FR-7 Liquid Glass / Visuelles Design
- FR-7.1 **iOS 26:** ausschließlich System-Komponenten, damit Liquid Glass automatisch greift:
  - `TabView` mit `Tab(...)`, Suche als `Tab(role: .search)` (erscheint als separater Glass-Kreis rechts)
  - `tabBarMinimizeBehavior(.onScrollDown)`
  - Mini-Player in `tabViewBottomAccessory`
  - Toolbar-Buttons ohne eigene Hintergründe (System-Glass); eigene schwebende Elemente mit `.glassEffect()`
  - Keine eigenen Hintergründe/Farben auf Tab-Bar oder Toolbar.
- FR-7.2 **iOS 18–25:** gleiche Struktur, Tab-Bar Standard, schwebende Elemente mit `.ultraThinMaterial`.
- FR-7.3 Inhalte scrollen unter Tab-Bar und Mini-Player durch (kein Abschneiden, `safeAreaInset` korrekt).
- FR-7.4 Eine Akzentfarbe über `AppTheme` (Vorschlag Bernstein `#F2A33A`) für aktiven Tab, Fortschritt, Links. Sonst neutral.
- FR-7.5 Typografie: System (SF Pro), Large Titles 34 pt bold, Abschnittstitel 22 pt bold.
- FR-7.6 Mini-Player: Cover, Titel, „Podcast/Autor · noch X Min.“, Play/Pause; Fortschritt als 2-px-Linie am Unterrand.

## 5. Nicht-funktionale Anforderungen

- NFR-1 Tab-Wechsel < 300 ms bis Inhalt sichtbar (aus Cache).
- NFR-2 Keine Netzwerk-Requests beim reinen Tab-Wechsel, wenn Daten < 5 min alt.
- NFR-3 VoiceOver: eindeutige Tab-Labels; Profil-Button-Label inkl. Verbindungsstatus.
- NFR-4 Neue Strings in `Localizable.xcstrings` (de, en).
- NFR-5 Dynamic Type bis AX3 ohne abgeschnittene Tab-Labels.
- NFR-6 „Transparenz reduzieren“ (Bedienungshilfen) wird respektiert – Glass fällt auf opake Flächen zurück (System-Verhalten, nicht überschreiben).

## 6. Non-Goals

- Kein Umbau von CarPlay (`CarPlay/CarPlayTabBar.swift`) oder watchOS.
- Keine Änderung an Player-, Download- oder Sync-Logik.
- Keine neuen Server-API-Aufrufe.
- Keine bibliotheksübergreifende Suche (Phase 2+).

## 7. Phase 2 (separat zu spezifizieren)

Echter paralleler Kontext: jeder Tab hält seine eigene Bibliothek, Start zeigt **gemischt** „Weiterhören“, „Weiterlesen“ und „Neue Folgen“ aus allen Bibliotheken. Erfordert, `libraries.current` (≈ 100 Verwendungen in 26 Dateien, u. a. `PlayerManager`, `DownloadManager`, `PinnedPlaylistManager`, `SearchViewModel`) durch einen injizierten Bibliotheks-Kontext zu ersetzen. Aufwand ca. 1–2 Wochen.

## 8. Akzeptanzkriterien (Gherkin)

```gherkin
Szenario: Feste Tab-Leiste
  Angenommen ein Server mit Bibliotheken "Hörbücher", "Podcasts" und "E-Books"
  Wenn die App startet
  Dann zeigt die Tab-Leiste "Start, Hörbücher, Podcasts, E-Books" und den Such-Button
  Und die Tab-Leiste ändert sich beim Wechsel zwischen Tabs nicht

Szenario: Wechsel mit einem Tap
  Angenommen ich bin im Tab "Hörbücher"
  Wenn ich auf "E-Books" tippe
  Dann sehe ich "Weiterlesen" und das Grid meiner E-Book-Bibliothek

Szenario: Automatische E-Book-Zuordnung
  Angenommen der Server hat die Buch-Bibliotheken "Hörbücher" und "E-Books"
  Dann erscheint "Hörbücher" im Tab "Hörbücher"
  Und "E-Books" im Tab "E-Books"

Szenario: Zuordnung manuell ändern
  Angenommen eine Buch-Bibliothek "Lesestoff"
  Wenn ich sie in Einstellungen > Bibliotheken dem Typ "E-Books" zuordne
  Dann erscheint sie im Tab "E-Books"

Szenario: Fehlender Medientyp
  Angenommen der Server hat keine Podcast-Bibliothek
  Dann wird der Tab "Podcasts" nicht angezeigt

Szenario: Zustand bleibt erhalten
  Angenommen ich habe im Tab "Hörbücher" ein Buch-Detail geöffnet
  Wenn ich zu "Podcasts" und zurück wechsle
  Dann sehe ich weiterhin das Buch-Detail

Szenario: Bild-Cache bleibt erhalten
  Angenommen Cover im Tab "Hörbücher" wurden geladen
  Wenn ich zu "Podcasts" und zurück wechsle
  Dann werden die Cover ohne erneuten Download angezeigt

Szenario: Downloads erreichbar
  Wenn ich auf den Profil-Button tippe
  Dann sehe ich "Downloads", "Server-Details", "Server verwalten" und "Einstellungen"
  Und keine Bibliotheksliste

Szenario: Liquid Glass
  Angenommen iOS 26
  Wenn ich in einer Liste nach unten scrolle
  Dann minimiert sich die Tab-Leiste
  Und Inhalte sind unter Tab-Leiste und Mini-Player durchscheinend sichtbar
```

## 9. Technischer Plan (Phase 1)

| # | Datei(en) | Änderung |
|---|---|---|
| T1 | `API/.../LibrariesService.swift` | `MediaTab`-Zuordnung (Heuristik + Override), `lastUsed(for: MediaTab)`; Cache-Clear nur bei Serverwechsel |
| T2 | `ContentView.swift` | `TabSelection` → `.home, .audiobooks, .podcasts, .ebooks, .search`; feste Tabs in `modernTabView` und `legacyTabView`; beim Tab-Wechsel `libraries.current` setzen; Downloads-Tab entfernen |
| T3 | `Screens/Library/LibraryRootPage.swift` | Wiederverwendbar für Hörbücher **und** E-Books (Parameter `MediaTab`); Segmente Titel/Serien/Autoren/Sammlungen; Bibliotheks-Picker; Filter „Nur geladene“ |
| T4 | neu: `Screens/EBooks/` (dünn) | „Weiterlesen“-Sektion + 2:3-Grid, Tap → Reader |
| T5 | `PodcastsRootPage.swift`, `LatestView.swift` | Container mit Segment „Neueste Folgen / Abonniert“, Podcast-Reihe |
| T6 | `HomePage.swift`, `HomePageModel.swift` | Profil-Button statt `serverMenuToolbarItem`, Downloads-Eintrag, Begrüßung |
| T7 | `Screens/Settings/` | Abschnitt „Bibliotheken“: Zuordnung Hörbücher/E-Books |
| T8 | Mini-Player, `AppTheme` | Fortschrittslinie, Akzentfarbe, eigene Hintergründe entfernen (Glass) |
| T9 | `Localizable.xcstrings` | Neue Strings |

**Aufwand Phase 1:** ca. 2,5–3 Tage inkl. Test.

## 10. Testplan

- Unit: Zuordnungs-Heuristik (Namen, Override), `lastUsed(for:)`, Serverwechsel.
- UI-Test: Szenarien aus §8 auf iOS 18 und iOS 26.
- Manuell: iPad, Dynamic Type AX3, VoiceOver, „Transparenz reduzieren“, Server ohne Podcast-/E-Book-Bibliothek, Offline-Modus.

## 11. Offene Fragen

1. Akzentfarbe Bernstein als Default oder nur als Theme-Option?
2. iPad / Mac Catalyst: Tab-Leiste oder `.sidebarAdaptable`-Sidebar? (Vorschlag: `tabViewStyle(.sidebarAdaptable)` – fast gratis)
3. Gemischte Bibliotheken (Hörbuch + E-Book im selben Item): erscheint es in beiden Tabs? (Annahme Phase 1: nur in der zugeordneten Bibliothek)

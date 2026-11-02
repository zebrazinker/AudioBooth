# Spec 001 – Tab-Navigation: Hörbücher & Podcasts als feste Tabs

| | |
|---|---|
| **Status** | Draft |
| **Datum** | 2026-11-02 |
| **Design** | [AudioBooth Redesign (Canvas)](https://claude.ai/artifact/66Qha78m1FkuVMvqcfvLmT) – Screens *Start*, *Hörbücher*, *Podcasts* |
| **Betroffene Plattform** | iOS / iPadOS (iOS 18+, Liquid-Glass-Variante ab iOS 26) |
| **Nicht betroffen** | CarPlay, watchOS, Mac Catalyst-spezifische Menüs (siehe Non-Goals) |

---

## 1. Problem

Heute ist die Navigation **bibliotheksabhängig**:

- Die aktive Bibliothek wird über ein Menü oben links gewählt (`HomePage.serverMenuToolbarItem`, `Screens/Home/HomePage.swift`).
- Die Tab-Leiste (`ContentView.swift`) baut sich je nach `libraries.current.mediaType` komplett um:
  - Buch-Bibliothek: Home · Library · Collections · Downloads · Suche
  - Podcast-Bibliothek: Home · Latest · Library · Downloads · Suche
- Folge: Tabs „springen“, gleiche Labels („Library“) bedeuten Unterschiedliches, und der Wechsel Hörbuch ↔ Podcast erfordert drei Taps in einem versteckten Menü.

## 2. Ziel

Hörbücher und Podcasts sind **gleichwertige, feste Tabs** in der unteren Leiste. Die Tab-Leiste ändert sich nie. Der Bibliothekswechsel oben links entfällt als primäre Navigation.

## 3. User Stories

| ID | Als … | möchte ich … | damit … |
|---|---|---|---|
| US-1 | Hörer | mit **einem Tap** zwischen Hörbüchern und Podcasts wechseln | ich nicht ins Menü oben links muss |
| US-2 | Hörer | dass die Tab-Leiste immer gleich aussieht | ich mich auf die Position der Tabs verlassen kann |
| US-3 | Hörer mit mehreren Buch-Bibliotheken (z. B. „Hörbücher“, „E-Books“) | innerhalb des Hörbücher-Tabs die Bibliothek wechseln | ich trotzdem alle Bibliotheken erreiche |
| US-4 | Hörer | Serien, Autoren, Sprecher und Sammlungen im Hörbücher-Tab finden | ich keinen eigenen „Collections“-Tab brauche |
| US-5 | Podcast-Hörer | neueste Folgen und abonnierte Podcasts im Podcasts-Tab sehen | ich keinen eigenen „Latest“-Tab brauche |
| US-6 | Nutzer | Server-Status, Serverwechsel und Einstellungen über einen Profil-Button erreichen | diese Funktionen nicht verloren gehen |

## 4. Funktionale Anforderungen

### FR-1 Tab-Leiste
- FR-1.1 Die Tab-Leiste enthält genau: **Start · Hörbücher · Podcasts · Downloads · Suche** (Reihenfolge fix).
- FR-1.2 Die Tabs sind unabhängig von `libraries.current` sichtbar, sobald ein Server authentifiziert ist.
- FR-1.3 Hat der Server **keine** Bibliothek eines Typs, wird der zugehörige Tab ausgeblendet (nicht deaktiviert).
- FR-1.4 Erneutes Tippen auf den aktiven Tab scrollt nach oben / poppt zum Root (bestehendes `onTabItemTapped()`-Verhalten bleibt).
- FR-1.5 iOS 26: Suche bleibt `Tab(role: .search)`; `tabBarMinimizeBehavior(.onScrollDown)` und `tabViewBottomAccessory` (Mini-Player) bleiben erhalten.
- FR-1.6 iOS 18–25 (`legacyTabView`): gleiche Tabs, Suche als normaler Tab.

### FR-2 Bibliotheks-Kontext pro Tab (Phase 1)
- FR-2.1 Wird der Tab **Hörbücher** aktiv, wird `libraries.current` auf die zuletzt genutzte Bibliothek mit `mediaType == .book` gesetzt (Fallback: erste Buch-Bibliothek).
- FR-2.2 Analog setzt der Tab **Podcasts** die zuletzt genutzte Bibliothek mit `mediaType == .podcast`.
- FR-2.3 Die „zuletzt genutzte“ Bibliothek je Typ wird pro Server persistiert (`server.storage`, neuer Key je `MediaType`).
- FR-2.4 Start, Downloads und Suche verwenden weiterhin `libraries.current` (= zuletzt über Hörbücher/Podcasts gewählt).
- FR-2.5 Ein Tab-Wechsel darf **nicht** den globalen Bild-Cache leeren (heute: `ImagePipeline.shared.cache.removeAll()` im Setter von `LibrariesService.current`). Der Cache-Reset ist auf Serverwechsel zu beschränken.
- FR-2.6 Der Navigations-State (`NavigationStack`-Pfad, Scrollposition, gewähltes Segment) jedes Tabs bleibt beim Hin- und Herwechseln erhalten.

### FR-3 Hörbücher-Tab
- FR-3.1 Großer Titel „Hörbücher“.
- FR-3.2 Gibt es > 1 Buch-Bibliothek, steht darunter ein Picker „Bibliothek: ‹Name› ▾“ (Menu). Bei genau einer Bibliothek entfällt er.
- FR-3.3 Segmented Control: **Titel · Serien · Autoren · Sprecher · Sammlungen** (erweitert den bestehenden Picker in `LibraryRootPage` um Serien/Sammlungen; `CollectionsRootPage` wird als Segment eingebettet).
- FR-3.4 Grid-Darstellung mit Fortschrittsbalken am Cover-Unterrand (siehe Design).

### FR-4 Podcasts-Tab
- FR-4.1 Großer Titel „Podcasts“, Button „+“ (Podcast hinzufügen, falls Berechtigung vorhanden).
- FR-4.2 Segmented Control: **Neueste Folgen · Abonniert** (`LatestView` bzw. `PodcastsRootPage`).
- FR-4.3 Optional (Phase 2): horizontale Reihe abonnierter Podcasts mit Badge „neue Folgen“.
- FR-4.4 Mehrere Podcast-Bibliotheken: Picker wie FR-3.2.

### FR-5 Start-Tab
- FR-5.1 Begrüßung statt Bibliotheksname in der Toolbar.
- FR-5.2 Rechts oben: Profil-Button (Avatar mit Status-Punkt grün/orange/rot/grau gem. `connectionStatusColor`). Inhalt: Server-URL primär/alternativ, Server-Details, Server verwalten, Einstellungen.
- FR-5.3 Die Bibliotheksliste wird aus diesem Menü **entfernt**.
- FR-5.4 Phase 1: Inhalte wie bisher aus `libraries.current`. Phase 2: gemischte Inhalte (siehe §7).

### FR-6 Mini-Player & Theme
- FR-6.1 Mini-Player unverändert funktional; optisch: Fortschritt als 2-px-Linie am Unterrand.
- FR-6.2 Akzentfarbe ist bereits über `AppTheme` steuerbar – kein Hardcoding neuer Farben.

## 5. Nicht-funktionale Anforderungen

- NFR-1 Tab-Wechsel Hörbücher ↔ Podcasts < 300 ms bis Inhalt sichtbar (aus Cache).
- NFR-2 Keine zusätzlichen Netzwerk-Requests beim reinen Tab-Wechsel, wenn Daten < 5 min alt.
- NFR-3 VoiceOver: Tabs mit eindeutigen Labels; Profil-Button `accessibilityLabel` inkl. Verbindungsstatus.
- NFR-4 Lokalisierung: neue Strings in `Localizable.xcstrings` (de, en mindestens).
- NFR-5 Dynamic Type bis AX3 ohne abgeschnittene Tab-Labels.

## 6. Non-Goals

- Kein Umbau von CarPlay (`CarPlay/CarPlayTabBar.swift`) oder watchOS.
- Keine Änderung an Player, Downloads-Logik, Sync.
- Keine neuen Server-API-Aufrufe.
- Keine bibliotheksübergreifende Suche (Phase 2+).

## 7. Phase 2 (separat zu spezifizieren)

Echter paralleler Kontext: Hörbücher- und Podcasts-Tab halten je eine eigene Bibliothek, Start zeigt **gemischt** „Weiterhören“ und „Neue Folgen“ aus allen Bibliotheken. Erfordert, `libraries.current` (≈ 100 Verwendungen in 26 Dateien, u. a. `PlayerManager`, `DownloadManager`, `PinnedPlaylistManager`, `SearchViewModel`) durch einen injizierten Bibliotheks-Kontext zu ersetzen. Aufwand ca. 1–2 Wochen.

## 8. Akzeptanzkriterien (Gherkin)

```gherkin
Szenario: Feste Tab-Leiste
  Angenommen ein Server mit einer Buch- und einer Podcast-Bibliothek
  Wenn die App startet
  Dann zeigt die Tab-Leiste "Start, Hörbücher, Podcasts, Downloads, Suche"
  Und die Tab-Leiste ändert sich nicht, wenn ich zwischen Tabs wechsle

Szenario: Wechsel mit einem Tap
  Angenommen ich bin im Tab "Hörbücher"
  Wenn ich auf "Podcasts" tippe
  Dann sehe ich die neuesten Folgen meiner Podcast-Bibliothek
  Und libraries.current ist die Podcast-Bibliothek

Szenario: Zustand bleibt erhalten
  Angenommen ich habe im Tab "Hörbücher" ein Buch-Detail geöffnet
  Wenn ich zu "Podcasts" und zurück wechsle
  Dann sehe ich weiterhin das Buch-Detail

Szenario: Keine Podcast-Bibliothek
  Angenommen der Server hat nur Buch-Bibliotheken
  Dann wird der Tab "Podcasts" nicht angezeigt

Szenario: Mehrere Buch-Bibliotheken
  Angenommen der Server hat "Hörbücher" und "E-Books"
  Wenn ich im Tab "Hörbücher" den Picker auf "E-Books" stelle
  Dann zeigt das Grid die E-Books
  Und nach App-Neustart ist "E-Books" im Tab "Hörbücher" vorausgewählt

Szenario: Bild-Cache bleibt erhalten
  Angenommen Cover im Tab "Hörbücher" wurden geladen
  Wenn ich zu "Podcasts" und zurück wechsle
  Dann werden die Cover ohne erneuten Download angezeigt

Szenario: Server-Funktionen erreichbar
  Wenn ich auf den Profil-Button tippe
  Dann sehe ich Verbindungsstatus, "Server-Details" und "Server verwalten"
  Und keine Bibliotheksliste
```

## 9. Technischer Plan (Phase 1)

| # | Datei(en) | Änderung |
|---|---|---|
| T1 | `API/.../LibrariesService.swift` | `lastUsed(for: MediaType) -> Library?` + Setter, persistiert in `server.storage`; Cache-Clear aus `current`-Setter in Serverwechsel verschieben |
| T2 | `ContentView.swift` | `TabSelection` → `.home, .audiobooks, .podcasts, .downloads, .search`; beide TabViews auf feste Tabs umbauen; bei Tab-Wechsel `libraries.current = lastUsed(for:)` |
| T3 | `Screens/Library/LibraryRootPage.swift` | Segmente Serien + Sammlungen ergänzen, `CollectionsRootPage`-Inhalt einbetten; Bibliotheks-Picker (FR-3.2) |
| T4 | `Screens/Library/PodcastsRootPage.swift`, `Screens/Latest/LatestView.swift` | Gemeinsamer Container mit Segment „Neueste Folgen / Abonniert“; Picker (FR-4.4) |
| T5 | `Screens/Home/HomePage.swift`, `HomePageModel.swift` | `serverMenuToolbarItem` → Profil-Button, Bibliotheksliste entfernen, Begrüßung |
| T6 | `Localizable.xcstrings` | Neue Strings |
| T7 | `AppTheme` / Mini-Player | Fortschrittslinie, Feinschliff gem. Design |

**Aufwand Phase 1:** ca. 1,5–2 Tage inkl. Test.

## 10. Testplan

- Unit: `LibrariesService.lastUsed(for:)` – Persistenz, Fallback, Serverwechsel.
- UI-Test: Szenarien aus §8 auf iOS 18 (legacy) und iOS 26 (modern).
- Manuell: iPad (Split View / `isPartiallyFolded`), Dynamic Type AX3, VoiceOver, Server ohne Podcast-Bibliothek, Offline-Modus.

## 11. Offene Fragen

1. Sollen E-Books einen eigenen Tab bekommen oder im Hörbücher-Tab per Picker bleiben? (Annahme: Picker)
2. Wohin mit „Sprecher“ – eigenes Segment oder unter „Autoren“? (Annahme: eigenes Segment, wie heute)
3. Soll die Akzentfarbe (Design: Bernstein) Default werden oder nur eine Theme-Option?
4. Mac Catalyst: Tab-Leiste oder Sidebar?

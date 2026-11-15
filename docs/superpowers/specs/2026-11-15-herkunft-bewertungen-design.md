# Herkunft und Bewertungen auf jeder Kachel — Design

> Stand: 2026-11-15 · Status: Entwurf, wartet auf Freigabe
> Teil 2 von 4 des Vorhabens „KI-Vorschläge“ (Teil 1 „Weiterschauen“ ist live)

## 1. Absicht

Der Betreiber und seine Frau suchen abends etwas zum Schauen. Ein Blick auf die
Kachel soll sagen, was sie erwartet. Dafür reichen zwei Dinge:

- **Woher kommt der Titel?** Netflix, HBO, Apple TV+, Disney+ … Jeder Dienst
  hat seinen eigenen Stil, den man als Mensch kennt („heute was Leichtes →
  Netflix“, „was Härteres → Paramount+/HBO“). Kinofilme bekommen „KINO“.
- **Was sagen Kritiker, was sagen Zuschauer?** Die beiden weichen oft stark
  voneinander ab, und genau dieser Unterschied ist interessant. Beide Werte
  kommen von Rotten Tomatoes: 🍅 Tomatometer (Kritiker) und 🍿 Popcornmeter
  (Zuschauer).

Heute zeigt jede Kachel nur den TMDB-Stern (`vote_average`), und das sagt wenig.

## 2. Ziele

1. Jede Film- und Serien-Kachel im Web und auf tvOS zeigt oben ein
   **Herkunfts-Label** und unten **🍅 Kritiker % · 🍿 Zuschauer %**.
2. Der **Hover im Web** (bestehende `PreviewTile`) zeigt zusätzlich alle
   weiteren Bewertungen: IMDb, Metacritic, Letterboxd, Trakt, TMDB und den
   MDBList-Gesamtwert.
3. Die **Detailseiten** (Web und tvOS) zeigen Label und alle Bewertungen.
4. Der TMDB-Stern verschwindet überall, wo er heute steht.
5. Die Daten kommen **einmal zentral aus der API**; Web und tvOS fragen dort.
   Der MDBList-Key bleibt auf dem Server.
6. Die Daten liegen in einer **Datenbank-Tabelle**, damit sie Deploys und
   Neustarts überleben und Teil 4 (KI-Vorschläge) sie wiederverwenden kann.

## 3. Nicht-Ziele

- **Kachel wird auf tvOS beim Verweilen größer** (wie bei Netflix): später,
  eigener Schritt (Wunsch des Betreibers).
- Wo man einen Titel *heute* in Deutschland streamen kann (Watch-Provider).
- Eigene Bewertungen der Nutzer (Teil 3).
- Sender-Logos als Bild. Das Label ist Text auf einer farbigen Fläche.
- Filtern oder Sortieren nach Bewertungen (Teil 4).

## 4. Anti-Ziele

- Ein falsches Label. Lieber gar kein Label als „NETFLIX“ auf einem Kinofilm.
- Seiten laden spürbar langsamer als heute, weil sie auf MDBList warten.
- Der MDBList-Key landet in der Apple-TV-App oder im Browser.
- Das Tageslimit (100 000 Anfragen, Plus-Stufe) oder das 5-Minuten-Limit
  (1 000 Lesezugriffe) wird gerissen, weil jede Kachel jedes Mal neu fragt.
- Ein Fehler bei MDBList bricht eine Seite oder eine Kachel.

## 5. Rahmen (Ist-Zustand)

- **Kacheln holen ihre Daten direkt bei TMDB**, nicht über die API:
  - Web: Server-Komponenten über `openmedia-web/src/lib/tmdb.ts`. Kacheln:
    `components/movie/movie-card.tsx`, `components/show/show-card.tsx`
    (Grids `movie-grid.tsx`, `show-grid.tsx`, Suche `search-movie-card.tsx`),
    Hover `components/preview/preview-tile.tsx`, Hero
    `components/movie/hero-banner.tsx`, Detail
    `components/movie/movie-detail-hero.tsx` (Film- und Serienseite).
    API-Daten: `components/home/continue-row.tsx`,
    `components/library/library-card.tsx`, `app/watchlist/page.tsx`.
  - tvOS: alle Poster-Kacheln laufen durch
    `OpenMedia/Components/PosterGrid.swift` → `PosterCard` (`PosterItem` mit
    `MediaLink.movie(id)`/`.show(id)`); „Weiterschauen“ durch
    `Components/ContinueRow.swift`. Detail: `Screens/MovieDetailView.swift`,
    `Screens/ShowDetailView.swift` (`metaLine`). tvOS zeigt heute gar keine
    Bewertung.
- TMDB-Stern heute: `movie-card.tsx:61`, `show-card.tsx:51`,
  `hero-banner.tsx:48`, `movie-detail-hero.tsx:84`, `preview-tile.tsx:198`.
- **API-Schlüssel in der DB-Konfiguration**: `ConfigCategory`/`ConfigEntry`
  (AES-GCM verschlüsselt), gelesen mit `getEntry(category, key, true)` aus
  `src/lib/config-service.ts`; Vorbild `treasure-maps.ts:78`
  (`indexer`/`treasureMapsApiKey`). Kategorien werden per SQL-Migration
  angelegt (Vorbild `20261107180000_add_episode_requests`). Pflege über die
  Admin-Seite `openmedia-web/src/app/admin/config`.
- **MDBList** (geprüft am 2026-11-15 mit dem echten Key, Wiki
  `llm-wiki/web-sources/mdblist-api.md`):
  - `GET https://api.mdblist.com/tmdb/{movie|show}/{tmdbId}/?apikey=…&append_to_response=releases`
  - Serien haben das Feld **`network`** (`HBO`, `Apple TV`, `Netflix`, `Disney+`).
  - Filme haben **`releases.digital.detail`**: `Netflix`, `Amazon Prime Video`,
    `Disney+, Disney +` bei Originals; `null` oder `TVOD` bei Kinofilmen.
  - `ratings[]` = `{source, value, score, votes}`; Quellen `tomatoes`
    (Kritiker %), `popcorn` (Zuschauer %), `imdb` (0–10), `metacritic`
    (0–100), `metacriticuser`, `letterboxd` (0–5), `trakt` (%), `tmdb` (%),
    `rogerebert`, `myanimelist`. Fehlende Quellen kommen mit `null`.
  - `score` = MDBList-Gesamtwert 0–100.
  - **Falle:** Die Batch-Abfrage `POST /tmdb/movie/` liefert *kein* `network`
    und *keine* `releases`. Für das Label braucht es Einzelabfragen.
  - Limits pro Konto: 100 000/Tag, 1 000 Lesezugriffe pro festem
    5-Minuten-Fenster; `429` mit `Retry-After`.

## 6. Design

### 6.1 Label-Regel

| Art | Quelle | Ergebnis |
|---|---|---|
| Serie | `network` gesetzt | normalisierter Name (Tabelle unten), sonst der echte Name |
| Serie | `network` leer | kein Label |
| Film | `releases.digital.detail` ist ein bekannter Abo-Dienst | dessen Label |
| Film | `detail` ist `null`, leer oder `TVOD` | **KINO** |
| Film | `detail` ist ein anderer Name | dieser Name (neutral) |

`detail` kann mehrere Namen mit Komma enthalten (`Disney+, Disney +`); es
zählt der erste.

Normalisierung (Vergleich ohne Groß-/Kleinschreibung und ohne Leerzeichen):

| Rohwerte | Label | Marke |
|---|---|---|
| Netflix | NETFLIX | `netflix` |
| Amazon Prime Video, Prime Video, Amazon | PRIME VIDEO | `prime` |
| Disney+, Disney Plus | DISNEY+ | `disney` |
| Apple TV, Apple TV+ | APPLE TV+ | `apple` |
| HBO, HBO Max, Max | HBO | `hbo` |
| Paramount+, Paramount Plus | PARAMOUNT+ | `paramount` |
| Sky, Sky Atlantic, Sky One, Sky Deutschland | SKY | `sky` |
| (Film-Regel) | KINO | `cinema` |
| alles andere | Name in Großbuchstaben | `other` |

Die Marke bestimmt die Farbe der Label-Fläche (Web und tvOS gleich, neutral
für `other`). Die Tabelle steht an **einer** Stelle in der API; die Clients
bekommen `label` und `brand` fertig.

### 6.2 Datenmodell

Neue Tabelle `title_meta`, ein Eintrag pro Titel:

| Feld | Bedeutung |
|---|---|
| `kind` | `movie` oder `show` |
| `tmdbId` | TMDB-ID; eindeutig zusammen mit `kind` |
| `label`, `brand` | Ergebnis von 6.1, beide nullable |
| `ratings` | JSON: die Quellen aus `ratings[]` als `{source: {value, votes}}`, ohne `null`-Werte |
| `score` | MDBList-Gesamtwert, nullable |
| `year` | Erscheinungsjahr aus MDBList (`year`), nullable; für die Auffrisch-Regel |
| `status` | `found` oder `not_found` (MDBList kennt den Titel nicht) |
| `fetchedAt` | wann zuletzt bei MDBList geholt |

Keine Beziehung zu anderen Tabellen: Kacheln zeigen auch Titel, die nicht in
`nzb_movies`/`nzb_shows` stehen.

**Auffrischen:** Ein Eintrag gilt als veraltet nach 7 Tagen, bei Titeln aus
dem aktuellen oder letzten Jahr (Bewertungen ändern sich dort schnell) nach
1 Tag, bei `not_found` nach 1 Tag. Die Zahlen stehen als Konstanten an einer
Stelle.

### 6.3 API

Neue Route `POST /meta` (requireAuth), Body
`{ items: [{ kind: "movie" | "show", tmdbId: number }] }`, höchstens 60
Einträge (mehr → 400).

Antwort `{ items: [{ kind, tmdbId, label, brand, ratings, score }] }` in der
Reihenfolge der Anfrage. Für unbekannte oder fehlerhafte Titel kommen
`label`, `brand`, `score` als `null` und `ratings` als `{}`.

`ratings` in der Antwort (nur vorhandene Felder):

| Feld | Quelle | Anzeige |
|---|---|---|
| `critics` | `tomatoes` | 🍅 % |
| `audience` | `popcorn` | 🍿 % |
| `imdb` | `imdb.value` | ★ 0–10 |
| `metacritic` | `metacritic` | 0–100 |
| `letterboxd` | `letterboxd.value` | 0–5 |
| `trakt` | `trakt` | % |
| `tmdb` | `tmdb` | % |

Ablauf:
1. Alle angefragten Titel aus `title_meta` lesen.
2. **Fehlende** Titel sofort bei MDBList holen, höchstens 6 gleichzeitig,
   Zeitlimit 8 s je Abfrage, dann speichern und mit ausliefern.
3. **Veraltete** Titel sofort mit den alten Daten ausliefern und im
   Hintergrund auffrischen (nicht auf die Antwort warten). Ein Titel wird
   nicht doppelt gleichzeitig geholt.
4. MDBList-Fehler: `404`/`not found` → `not_found` speichern. `429`,
   Zeitüberschreitung, Netzfehler → nichts speichern, `null` ausliefern,
   einmal loggen. Ist kein Key konfiguriert → `null` für alle, kein Fehler.

Key: Konfigkategorie `metadata` („Metadaten“, „Zugänge zu Film- und
Seriendatenbanken (MDBList)“), Eintrag `mdblistApiKey`, verschlüsselt. Die
Kategorie legt die Migration an; den Wert trägt der Betreiber in der
Admin-Seite ein.

### 6.4 Web

- Ein kleiner Client-Baustein sammelt alle Kachel-Anfragen eines Moments und
  schickt **eine** `POST /meta` (über den Backend-Proxy), statt eine pro
  Kachel; mehr als 60 Titel werden in 60er-Blöcke geteilt. Ergebnisse werden
  im Browser für die Sitzung gemerkt.
- `hero-banner` und `movie-detail-hero` sind Server-Komponenten: sie holen die
  Werte entweder serverseitig mit dem Cookie-Token oder über einen kleinen
  Client-Baustein (Builder-Entscheidung).
- Kacheln (`movie-card`, `show-card`, `library-card`, Watchlist-Kachel,
  „Weiterschauen“) zeigen:
  - oben links auf dem Poster das Label als kleine farbige Fläche;
  - in der Infozeile statt Stern: `🍅 36% · 🍿 92%`; fehlt ein Wert, fällt er
    weg; fehlen beide, steht nur das Jahr.
  - Bis die Daten da sind, ist der Platz leer (kein Springen der Höhe: die
    Infozeile hat feste Höhe).
- `PreviewTile` (Hover) zeigt zusätzlich alle weiteren Werte aus der Tabelle
  in 6.3 und den MDBList-Gesamtwert.
- `hero-banner` und `movie-detail-hero`: Label und alle Werte statt Stern.
- Zugänglichkeit: Label und Werte haben einen Text für Screenreader
  („Kritiker 36 Prozent, Zuschauer 92 Prozent“).

### 6.5 tvOS

- `OpenMediaKit`: Modell `TitleMeta` und `APIClient.meta(_ items:)`; die Marke
  → Farbe und die Formatierung der Werte als reine, getestete Funktionen.
- `PosterGrid` holt die Metadaten für alle seine Kacheln mit einer Anfrage
  (bei mehr als 60 Kacheln in 60er-Blöcken);
  `PosterCard` zeigt Label (oben auf dem Poster) und `🍅 · 🍿` unter dem Titel.
- `ContinueRow`: gleiche Anzeige (Episoden → `kind: show` mit der Serien-ID).
- Detailseiten: Label und alle Werte in einer Zeile unter `metaLine`.

## 7. Ansätze (betrachtet)

| Ansatz | Bewertung |
|---|---|
| **API + Datenbank-Tabelle** | gewählt: überlebt Deploys, schnell nach dem ersten Abruf, Teil 4 nutzt dieselben Daten |
| API + nur Arbeitsspeicher (`ttl-cache`) | verworfen: nach jedem Deploy alles neu holen (1–2 s pro Seite), Teil 4 bräuchte die Tabelle trotzdem |
| Web und tvOS fragen MDBList direkt | verworfen: Key in der App, Logik doppelt |

## 8. Entscheidungen

**Vom Betreiber:**
- Kinofilme bekommen „KINO“ (Option A), Originals den Namen des Dienstes.
- Auf der Kachel: beide Rotten-Tomatoes-Werte, 🍅 Kritiker und 🍿 Zuschauer.
- Weitere Werte im Web-Hover; tvOS-Vergrößerung später.
- Daten in einer DB-Tabelle (Weg A).
- Key liegt in 1Password; er gehört auf den Server.

**Vorschläge des Designers:**
- Feste Label-Liste mit Marken-Farben, unbekannte Sender mit echtem Namen neutral.
- Fehlende Werte werden weggelassen, kein Ersatz durch TMDB.
- Key in der Admin-Konfiguration (Kategorie `metadata`).
- Auffrischen nach 7 Tagen / 1 Tag (neue Titel, `not_found`), Auslieferung
  veralteter Daten mit Auffrischen im Hintergrund.
- Höchstens 60 Titel pro Anfrage, 6 gleichzeitige MDBList-Abfragen.
- Der TMDB-Stern verschwindet auch im Hero und auf den Detailseiten.

## 9. Dem Builder überlassen

- Genaue Farben je Marke (passend zu `variables`/Tailwind-Theme; vor dem
  Melden im Browser und auf tvOS ansehen).
- Wie der Web-Sammelbaustein gebaut ist (Hook, Kontext, Modul), solange es eine
  Anfrage pro Moment und Seite bleibt.
- Prisma-Modellierung und Index.
- Tests: Pflicht sind API-Tests für die Label-Regel (alle Zeilen aus 6.1), das
  Speichern/Auffrischen (fehlend, veraltet, `not_found`, `429`, kein Key) und
  die 60er-Grenze; Kit-Tests für Formatierung und Marke.

## 10. Auslieferung

API zuerst (Migration legt Tabelle und Konfigkategorie an), dann den Key in
der Admin-Seite eintragen, dann Web, dann tvOS. Ohne Key zeigen alle Kacheln
einfach keine Labels und Werte.

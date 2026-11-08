# Vorschau-Kachel in openmedia-web — Design

> Stand: 2026-11-08 · Status: Entwurf, wartet auf Freigabe

## 1. Absicht

Auf den Startseiten von openmedia-web sieht man heute nur Poster mit Titel,
Bewertung und Jahr. Nur anhand eines Posters fällt die Entscheidung schwer:
„Will ich das sehen oder nicht?“. Der Betreiber möchte eine kurze Vorschau wie
bei Netflix. Bleibt man kurz auf einer Kachel, dann wird sie größer. Ein
Trailer läuft, und man sieht eine kurze Beschreibung und die Bewertung. Direkt
in der Kachel kann man dann entscheiden: herunterladen oder nicht.

Geschaut wird hauptsächlich am Beamer über Apple TV und Jellyfin. Ausgesucht
wird auf der Webseite, mit Laptop und Maus. Eine eigene Apple-TV-App ist ein
späteres, eigenes Projekt. Diese Kachel ist der günstige Test für das Konzept.

## 2. Ziele

1. Bleibt die Maus etwa 1 Sekunde auf einer Film- oder Serien-Kachel, dann
   wird diese Kachel zu einer großen Vorschau-Kachel.
2. In der Vorschau-Kachel läuft der Trailer, und es stehen dort die Infos für
   die Entscheidung.
3. Aus der Vorschau-Kachel heraus kann man direkt herunterladen: bei Filmen
   den Film, bei Serien die nächste Folge. Dabei ist klar sichtbar, was genau
   geladen wird.

## 3. Nicht-Ziele

- **Kein neuer Seitenaufbau.** Das Raster bleibt so wie heute (`MovieGrid`,
  `ShowGrid`). Es gibt keine Netflix-Reihen, keine Anbieter-Reihen und keine
  breiten Bilder. Das kommt später mit der Apple-TV-App.
- **Kein Film-Indexer.** Die Filmsuche über Treasure Maps ist das nächste,
  eigene Projekt. Bis dahin lädt der Film-Knopf nur Filme, für die es schon
  eine NZB gibt.
- **Keine Apple-TV-App.**
- **Keine Watchlist für Serien.** Die Watchlist kennt heute nur Filme
  (`WatchlistItem.movieId`). Die Serien-Kachel bekommt deshalb keinen
  „Merken“-Knopf. *(Entscheidung des Designers, vom Betreiber noch nicht bestätigt.)*
- **Keine Vorschau auf dem Handy.** Auf Touch-Geräten öffnet ein Tippen die
  Detailseite wie heute.

## 4. Anti-Ziele (das wäre ein Fehlschlag, auch wenn es technisch läuft)

- Der Nutzer klickt auf „Herunterladen“ und weiß vorher nicht, was geladen
  wird (welcher Film, welche Folge).
- Beim Laden der Startseite wird für jede Kachel TMDB abgefragt. Das darf nur
  für die Kachel passieren, auf der die Maus bleibt.
- Die Vorschau springt schon auf, wenn die Maus nur über das Raster fährt.
- Eine Kachel am Rand wird abgeschnitten.
- Ein Trailer startet mit Ton.
- Download- oder Anfrage-Logik wird für die Kachel neu gebaut, statt die von
  der Detailseite wiederzuverwenden.

## 5. Ausgangslage (verifiziert im Code)

Alle Pfade beziehen sich auf `/Users/jschattka/git_private/cinescope/`.

- **Kacheln:** `openmedia-web/src/components/movie/movie-card.tsx` und
  `openmedia-web/src/components/show/show-card.tsx`. Beide zeigen ein
  2:3-Poster mit Titel, Bewertung und Jahr. Beim Hover wird die Karte
  `scale-[1.03]` mit Gold-Rand.
- **Wo die Kacheln vorkommen:** Film-Startseite (`src/app/page.tsx` →
  `MovieGrid`), Serien-Startseite (`src/app/serien/page.tsx` → `ShowGrid`),
  Genres (`src/app/genres/[id]/page.tsx`), Suche (`src/app/search/page.tsx`
  über `SearchMovieCard` und `ShowCard`), Watchlist (`src/app/watchlist/page.tsx`),
  letzte Suchen (`src/components/search/recent-searches.tsx`) und „Ähnliche“
  auf den Detailseiten (`src/app/movie/[id]/page.tsx`, `src/app/serie/[id]/page.tsx`).
- **Trailer:** `src/components/movie/trailer-section.tsx` nimmt aus TMDB
  `videos` den ersten YouTube-Trailer (`official` bevorzugt) und bettet ihn
  über `youtube-nocookie.com/embed/{key}` ein.
- **TMDB:** `src/lib/tmdb.ts` → `tmdbFetch` setzt immer `language=de-DE`.
  `getMovieDetails` und `getShowDetails` hängen `credits,videos,similar` an.
  **Achtung:** Wegen `language=de-DE` liefert TMDB bei `videos` nur deutsche
  Videos. Viele Titel haben dann keinen Trailer. Mit dem TMDB-Parameter
  `include_video_language=de,en` kommen deutsche und englische Trailer.
  Der TMDB-Key ist nur auf dem Server (`process.env.TMDB_API_KEY`). Das
  Muster für Abfragen aus dem Browser ist eine Server Action, siehe
  `src/app/actions/tmdb-search.ts`.
- **Film-Download:** `src/components/movie/download-button.tsx`
  (`DownloadButton`) prüft über `checkAvailability(movie.id)`, ob es NZB-Versionen
  gibt, und startet den Download mit Versionsauswahl. Den gleichen Ablauf
  braucht die Kachel.
- **Watchlist:** `src/components/movie/watchlist-button.tsx`, nur für Filme.
- **Serien-Anfragen:** `src/lib/backend.ts` → `requestEpisodes({ tmdbId,
  season, episode? })` ruft `POST /series/requests` auf.
  `getNzbShowByTmdb(tmdbId)` liefert die vorhandenen Folgen-Versionen und
  `myRequests` (Status `searching | waiting | downloading | done | expired`).
  Ist eine Anfrage fertig, legt das Backend die Folge in die `UserLibrary` des
  Nutzers (`openmedia-api/src/lib/episode-requests.ts`, `markDone` →
  `addToLibrary`). Die Detailseite nutzt das alles in
  `src/components/show/season-list.tsx`.

## 6. Entscheidungen des Betreibers

### 6.1 Verhalten

- **Variante „Kachel springt auf“** (wie Netflix). Die Kachel wächst an ihrer
  Stelle und liegt dann über den Nachbarn. Die Variante „großer fester Bereich
  oben“ wurde verworfen.
- Die Vorschau öffnet sich nach **etwa 1 Sekunde** auf der Kachel.
- Das **Raster bleibt** wie heute.

### 6.2 Inhalt der Vorschau-Kachel

- Oben läuft der Trailer.
- Darunter stehen Titel, Bewertung, Jahr, Genre und eine kurze Beschreibung.
  Bei Serien steht dort auch die Anzahl der Staffeln.
- Knöpfe: **Herunterladen**, **Merken** (nur Filme) und **Mehr** (öffnet die
  Detailseite).

### 6.3 Serien: „Nächste Folge“

- Über dem Knopf steht, welche Folge geladen wird, zum Beispiel „Nächste:
  Staffel 2, Folge 4“. So weiß der Nutzer, was passiert. Passt das nicht,
  geht er über „Mehr“ auf die Detailseite und wählt dort selbst aus.

### 6.4 Filme

- Bis zum Film-Indexer lädt der Knopf nur Filme, für die es schon eine NZB
  gibt. Gibt es keine, dann bleibt nur „Mehr“.

## 7. Vom Designer entschieden (vom Betreiber bestätigt)

- **Ton:** Der Trailer startet stumm. Ein Lautsprecher-Knopf schaltet den Ton ein.
- **Kein Trailer:** Dann zeigt die Kachel das Backdrop-Bild statt des Videos.
- **Trailer-Sprache:** Ein deutscher Trailer kommt zuerst, sonst ein
  englischer. Innerhalb der Sprache kommt `official` zuerst.
- **Daten erst bei Bedarf:** Trailer, Genre, Staffel-Anzahl und „nächste Folge“
  werden erst geladen, wenn die Vorschau aufgeht. Ist eine Kachel einmal
  geladen, dann wird sie nicht neu geladen, solange die Seite offen ist.
- **Rand:** Die Vorschau wird nie abgeschnitten, auch nicht in Containern mit
  `overflow-hidden` oder am unteren Bildschirmrand. Dafür darf sie per Portal
  über der Seite liegen und ihre Position am Viewport messen. „Erste/letzte
  Spalte wächst nach innen“ ist nur die sichtbare Folge davon.
- **Überall:** Die Vorschau gibt es an jeder Stelle, wo heute eine Film- oder
  Serien-Kachel steht (Liste in Abschnitt 5).
- **Handy/Touch:** Keine Vorschau. Ein Tippen öffnet die Detailseite wie heute.
- **Maus weg:** Die Kachel wird wieder klein, und der Trailer stoppt.
- **Bedeutung „nächste Folge“ (Entscheidung des Betreibers):** Es geht weiter
  nach der letzten Folge, die der Nutzer **je hatte**. Beispiel: Er hatte
  S01E01–E03, hat sie geschaut und aus der Bibliothek gelöscht → nächste Folge
  ist S01E04, nicht S01E01. Lücken werden bewusst übersprungen: Hat er nur
  S01E05 angefragt, ist die nächste Folge S01E06.
  - „Je hatte“ zählt: jeder `UserLibrary`-Eintrag zu einer Folge dieser Serie,
    **auch mit gesetztem `removedAt`**, und jede eigene Anfrage mit Status
    `searching`, `waiting`, `downloading` oder `done`.
  - Nicht mit: Anfragen mit `expired`.
  - Maßgeblich ist die höchste Folge nach (Staffel, Folge). Die nächste Folge
    ist die direkt danach; ist eine Staffel zu Ende, dann Folge 1 der nächsten
    Staffel. Hatte er noch nichts, dann ist es S01E01.
  - Specials (Staffel 0) zählen nie, weder als „hatte“ noch als Vorschlag.
  - **Achtung:** `GET /library` liefert nur Einträge mit `removedAt: null`
    (`openmedia-api/src/routes/library.ts`, Zeile 14). Gelöschte Folgen sind
    dort also nicht sichtbar.
- **Quelle der ausgestrahlten Folgen ist TMDB**, nicht die Datenbank. Die
  Datenbank kennt nur Folgen, die schon jemand angefragt hat. Eine einzige
  Abfrage reicht: `tv/{id}` liefert `seasons[].episode_count` und
  `last_episode_to_air` (Staffel und Folge). Ausgestrahlt sind alle Folgen der
  Staffeln vor `last_episode_to_air.season_number` und in dieser Staffel die
  Folgen 1 bis `last_episode_to_air.episode_number`. `last_episode_to_air` fehlt
  heute noch im Typ `Show`/`ShowDetail` in `src/lib/tmdb.ts`.
- **Anfrage geht auch ohne Datenbank-Eintrag:** `POST /series/requests` mit
  `episode` holt die Folge über `getTmdbEpisodeById` von TMDB und legt sie an
  (`openmedia-api/src/lib/episode-requests.ts`, `resolveTargets`).
- **Serien-Knopf lädt genau eine Folge**, nicht die ganze Staffel. Er nutzt
  `requestEpisodes({ tmdbId, season, episode })`.
- **Alles da:** Ist die nächste Folge noch nicht ausgestrahlt (oder gibt es sie
  nicht), dann steht dort „Alles da“, und es gibt keinen Herunterladen-Knopf.
- **Nicht angemeldet:** Die Vorschau mit Trailer und Infos geht auch ohne
  Anmeldung. Die Knöpfe führen dann wie heute auf `/login`.

## 8. Dem Builder überlassen

- Wie die API die „je gehabten“ Folgen liefert. Weil `GET /library` gelöschte
  Einträge ausblendet, braucht es dafür wahrscheinlich einen kleinen neuen
  Endpunkt (oder eine Erweiterung von `GET /nzb/shows/by-tmdb/:id`). Ob die
  nächste Folge dann im Browser oder in der API berechnet wird, entscheidet der
  Builder. Bedeutung und Quelle aus Abschnitt 7 müssen gleich bleiben.
- Ob die Vorschau eine eigene Komponente ist, die `MovieCard` und `ShowCard`
  gemeinsam nutzen, und wie sie positioniert wird (z. B. absolut über dem
  Raster oder als Popover).
- Wie Hover/Touch erkannt wird (z. B. `(hover: hover)`-Media-Query).
- Ob der Film-Knopf `DownloadButton` direkt einbettet oder dessen Logik
  herauszieht. Wichtig ist nur: keine zweite Download-Logik.
- Wie der Trailer stumm startet und der Ton umgeschaltet wird (YouTube-Embed
  mit `autoplay=1&mute=1` oder die IFrame-API).
- Ob `include_video_language` nur für die Vorschau oder auch auf den
  Detailseiten gesetzt wird. Auch die Detailseiten würden davon profitieren.
- Die genaue Optik. Sie soll zur bestehenden dunklen Kino-Palette passen
  (`src/app/globals.css`, `cinema-gold`). Referenz für den Aufbau ist der
  Entwurf „A · Kachel springt auf“ in
  `openmedia-docs/.superpowers/brainstorm/21663-1794130341/content/vorschau-art.html`.

# Herkunft und Bewertungen Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Jede Film- und Serien-Kachel im Web und auf tvOS zeigt ein Herkunfts-Label (NETFLIX, HBO, KINO …) und 🍅 Kritiker · 🍿 Zuschauer von Rotten Tomatoes; Hover und Detailseiten zeigen alle Bewertungen. Daten kommen aus MDBList über die API und liegen in der Tabelle `title_meta`.

**Architecture:** Reine Regeln (Label, Marke, Bewertungs-Mapping, Auffrisch-Regel) liegen in `openmedia-api/src/lib/title-meta.ts`. `src/lib/mdblist.ts` holt einen Titel bei MDBList, `src/lib/meta-service.ts` liest/füllt/frischt `title_meta` auf, `src/routes/meta.ts` bietet `POST /meta`. Das Web sammelt alle Kachel-Anfragen eines Moments in einer Anfrage (`src/lib/title-meta-client.ts` + Hook) und zeigt sie mit drei kleinen Komponenten. tvOS bekommt Modell, API-Aufruf und Formatierung im Kit; `PosterGrid` holt pro Grid einmal.

**Tech Stack:** Express 5, Prisma 7, PostgreSQL, Vitest + supertest (echte Test-DB auf Port 5433); Next.js 16, React 19, Tailwind, Vitest + Testing Library; SwiftUI tvOS 18, Swift Testing in `OpenMediaKit`.

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-15-herkunft-bewertungen-design.md`

**Repos/Branches:** `openmedia-api`, `openmedia-web`, `openmedia-tvos` unter `/Users/jschattka/git_private/cinescope/`, jeweils Branch `feature/herkunft-bewertungen` von frischem `main`.

## Global Constraints

- Alle Texte für Nutzer auf Deutsch, mit echten Umlauten. Keine neuen Abhängigkeiten.
- MDBList-Key nur auf dem Server: Konfigkategorie `metadata`, Eintrag `mdblistApiKey`, gelesen mit `getEntry("metadata", "mdblistApiKey", true)`. Nie in Web-Bundle oder tvOS-App.
- Label-Regel und Normalisierung exakt wie Spec 6.1; die Tabelle steht nur in `title-meta.ts`. Clients bekommen `label` + `brand` fertig.
- Marken: `netflix`, `prime`, `disney`, `apple`, `hbo`, `paramount`, `sky`, `cinema`, `other`.
- Auffrischen: 7 Tage; 1 Tag bei `year >= aktuelles Jahr − 1` oder `status = not_found`. Konstanten an einer Stelle.
- `POST /meta`: höchstens 60 Einträge, höchstens 6 gleichzeitige MDBList-Abfragen, Zeitlimit 8 s je Abfrage; Clients teilen in 60er-Blöcke.
- Kachel zeigt `🍅 36% · 🍿 92%`; fehlender Wert fällt weg; Screenreader-Text „Kritiker 36 Prozent, Zuschauer 92 Prozent“.
- Der TMDB-Stern (`vote_average`) verschwindet aus `movie-card`, `show-card`, `hero-banner`, `movie-detail-hero`, `preview-tile`.
- MDBList-Fehler brechen nie eine Seite: Ergebnis ist dann `label: null`, `ratings: {}`.
- Migration nur additiv; bestehende Tests bleiben grün.

## Review Focus

1. Grid mit 20 neuen Titeln nach einem Deploy → höchstens 6 gleichzeitige MDBList-Abfragen, Antwort trotzdem in einem Rutsch (Test in Task 3).
2. Zwei Anfragen fast gleichzeitig für denselben fehlenden Titel → nur eine MDBList-Abfrage (Test in Task 3).
3. MDBList antwortet `429` oder läuft in das Zeitlimit → Kachel bleibt leer, nichts wird gespeichert, nächster Aufruf versucht es neu (Test in Task 3).
4. Titel, den MDBList nicht kennt → `not_found` gespeichert, nicht bei jedem Seitenaufruf neu gefragt (Test in Task 3).
5. Seite mit 130 Kacheln (Suche, Bibliothek) → drei Anfragen zu 60/60/10, keine 400 (Tests in Task 4 und Task 6).

---

### Task 1: Reine Regeln (API)

**Files:**
- Create: `openmedia-api/src/lib/title-meta.ts`
- Test: `openmedia-api/src/lib/title-meta.test.ts`

**Interfaces:**
- Produces:
  - `export type Kind = "movie" | "show"`
  - `export type Brand = "netflix" | "prime" | "disney" | "apple" | "hbo" | "paramount" | "sky" | "cinema" | "other"`
  - `export type Ratings = { critics?: number; audience?: number; imdb?: number; metacritic?: number; letterboxd?: number; trakt?: number; tmdb?: number }`
  - `export type StoredRatings = Record<string, { value: number; votes: number | null }>` (MDBList-Quelle → Wert)
  - `export const META_RULES = { STALE_DAYS: 7, FRESH_TITLE_STALE_DAYS: 1, NOT_FOUND_STALE_DAYS: 1, MAX_ITEMS: 60, CONCURRENCY: 6, TIMEOUT_MS: 8000 } as const`
  - `export function originLabel(kind: Kind, raw: { network?: string | null; digitalDetail?: string | null }): { label: string; brand: Brand } | null`
  - `export function storedRatings(raw: { source: string; value: number | null; votes?: number | null }[]): StoredRatings` — nur Einträge mit `value !== null`
  - `export function toRatings(stored: StoredRatings): Ratings` — `critics` ← `tomatoes`, `audience` ← `popcorn`, `imdb` ← `imdb`, `metacritic` ← `metacritic`, `letterboxd` ← `letterboxd`, `trakt` ← `trakt`, `tmdb` ← `tmdb`
  - `export function isStale(row: { status: string; year: number | null; fetchedAt: Date }, now: Date): boolean`

- [ ] **Step 1: Write the failing tests:**
  - `originLabel("show", { network: "HBO" })` → `{ label: "HBO", brand: "hbo" }`; `"Apple TV"` → `{ "APPLE TV+", "apple" }`; `"Netflix"` → `{ "NETFLIX", "netflix" }`; `"Disney+"` → `{ "DISNEY+", "disney" }`; `"Max"` → `{ "HBO", "hbo" }`; `"Sky Atlantic"` → `{ "SKY", "sky" }`; `"BBC One"` → `{ "BBC ONE", "other" }`; `network: null` → `null`.
  - `originLabel("movie", { digitalDetail: "Netflix" })` → NETFLIX; `"Amazon Prime Video"` → `{ "PRIME VIDEO", "prime" }`; `"Disney+, Disney +"` → DISNEY+; `"Paramount Plus"` → `{ "PARAMOUNT+", "paramount" }`; `null` → `{ "KINO", "cinema" }`; `""` → KINO; `"TVOD"` → KINO; `"tvod"` → KINO; `"Hulu"` → `{ "HULU", "other" }`.
  - `storedRatings([{source:"tomatoes",value:36,votes:174},{source:"rogerebert",value:null}])` → `{ tomatoes: { value: 36, votes: 174 } }`.
  - `toRatings({ tomatoes:{value:36,votes:1}, popcorn:{value:92,votes:1}, imdb:{value:6.3,votes:1}, metacriticuser:{value:5.9,votes:1} })` → `{ critics: 36, audience: 92, imdb: 6.3 }`.
  - `isStale`: `found`, `year: 2010`, 6 Tage alt → `false`, 8 Tage → `true`; `year` = aktuelles Jahr − 1, 2 Tage alt → `true`; `not_found`, 2 Tage → `true`; `year: null`, 6 Tage → `false`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/title-meta.test.ts` — FAIL (Modul fehlt).
- [ ] **Step 3: Implement** `title-meta.ts` (Normalisierung: Kleinbuchstaben, Leerzeichen entfernt; bei Komma der erste Name).
- [ ] **Step 4: Run** — PASS; `npm test` grün.
- [ ] **Step 5: Commit** `feat(meta): origin label and rating rules`

### Task 2: Tabelle, Konfigkategorie und MDBList-Client (API)

**Files:**
- Modify: `openmedia-api/prisma/schema.prisma` (Modell `TitleMeta`)
- Create: `openmedia-api/prisma/migrations/<timestamp>_add_title_meta/migration.sql` (Tabelle + Seed der Kategorie `metadata` wie in `20261107180000_add_episode_requests`)
- Modify: `openmedia-api/src/test/setup.ts` (`title_meta` in die TRUNCATE-Liste)
- Create: `openmedia-api/src/lib/mdblist.ts`
- Test: `openmedia-api/src/lib/mdblist.test.ts` (Muster `treasure-maps.test.ts`: `getEntry` gemockt, `fetch` per `vi.stubGlobal`)

**Interfaces:**
- Consumes: `Kind`, `META_RULES`, `originLabel`, `storedRatings` (Task 1).
- Produces:
  - Prisma `TitleMeta` (`@@map("title_meta")`): `id` uuid, `kind String`, `tmdbId Int`, `label String?`, `brand String?`, `ratings Json` (default `{}`), `score Int?`, `year Int?`, `status String`, `fetchedAt DateTime`; `@@unique([kind, tmdbId])`; snake_case `@map`.
  - Seed: `('metadata', 'Metadaten', 'Zugänge zu Film- und Seriendatenbanken (MDBList)')`, `ON CONFLICT ("name") DO NOTHING`.
  - `export type MdblistResult = { status: "found"; label: string | null; brand: Brand | null; ratings: StoredRatings; score: number | null; year: number | null } | { status: "not_found" } | { status: "error"; reason: "not_configured" | "rate_limited" | "timeout" | "http" }`
  - `export async function fetchMdblistTitle(kind: Kind, tmdbId: number): Promise<MdblistResult>` — `GET https://api.mdblist.com/tmdb/{movie|show}/{tmdbId}/?apikey=…` (Filme mit `&append_to_response=releases`), Zeitlimit `META_RULES.TIMEOUT_MS`; Label aus `network` bzw. `releases.digital.detail`; HTTP 404 oder JSON ohne `title` mit `error`, das „not found“ enthält (ohne Groß/Klein) → `not_found`; 429 → `rate_limited`; kein Key → `not_configured`. Wirft nie.

- [ ] **Step 1: Write the failing tests:** Film-Antwort mit `releases.digital.detail: "Netflix"` und `ratings` → `found`, `label: "NETFLIX"`, `ratings.tomatoes.value` gesetzt, `year` aus `year`; Serien-Antwort mit `network: "HBO"` → `label: "HBO"` und die URL ohne `append_to_response`; Film mit `detail: null` → KINO; 404 → `not_found`; `{ error: "Movie not found" }` mit 200 → `not_found`; 429 → `rate_limited`; `fetch` wirft `AbortError` → `timeout`; `getEntry` → `null` → `not_configured` und kein `fetch`-Aufruf; Key steht im Query-Parameter `apikey`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/mdblist.test.ts` — FAIL.
- [ ] **Step 3: Implement** Modell, Migration (keine lokale Dev-DB nötig: SQL per `npx prisma migrate diff` erzeugen, wenn `migrate dev` nicht geht), `setup.ts`, `mdblist.ts`.
- [ ] **Step 4: Run** — PASS; `npm test` und `npm run build` grün.
- [ ] **Step 5: Commit** `feat(meta): title_meta table, metadata config category and MDBList client`

### Task 3: Service und Route `POST /meta` (API)

**Files:**
- Create: `openmedia-api/src/lib/meta-service.ts`
- Create: `openmedia-api/src/routes/meta.ts`
- Modify: `openmedia-api/src/app.ts` (`app.use("/meta", metaRoutes)`)
- Test: `openmedia-api/src/routes/meta.test.ts` (`./lib/mdblist.js` per `vi.mock` gemockt; echte DB)

**Interfaces:**
- Consumes: Task 1 + 2.
- Produces:
  - `export type TitleMetaView = { kind: Kind; tmdbId: number; label: string | null; brand: Brand | null; ratings: Ratings; score: number | null }`
  - `export async function getMeta(items: { kind: Kind; tmdbId: number }[], now?: Date): Promise<TitleMetaView[]>` — Reihenfolge wie Anfrage, Duplikate erlaubt. Ablauf Spec 6.3: vorhandene frische Zeilen direkt; fehlende über `fetchMdblistTitle` (höchstens `CONCURRENCY` gleichzeitig) holen und upserten; veraltete sofort alt ausliefern und im Hintergrund auffrischen; laufende Abrufe pro `kind:tmdbId` teilen (Modul-Map); `found` und `not_found` speichern, `error` nicht.
  - Route `POST /meta` (requireAuth), Body `{ items: [...] }`: Validierung `kind ∈ {movie, show}`, `tmdbId` positive ganze Zahl, 1–60 Einträge, sonst 400 `{ error }` (deutsch). Antwort 200 `{ items: TitleMetaView[] }`.

- [ ] **Step 1: Write the failing tests:**
  - Fehlender Titel → ein `fetchMdblistTitle`-Aufruf, Antwort mit `label`, `ratings.critics`, Zeile in `title_meta`.
  - Zweiter Aufruf (frisch) → kein weiterer MDBList-Aufruf.
  - Zeile mit `fetchedAt` vor 8 Tagen, `year: 2010` → Antwort sofort mit alten Werten; nach `await vi.waitFor` wurde einmal neu geholt und die Zeile aktualisiert.
  - 20 fehlende Titel, gemockter Abruf mit Verzögerung → nie mehr als 6 gleichzeitig (Zähler im Mock), alle 20 in der Antwort.
  - Zwei parallele `POST /meta` für denselben fehlenden Titel → genau ein Abruf.
  - Abruf liefert `rate_limited` → Antwort `label: null, ratings: {}`, keine Zeile; nächster Aufruf fragt wieder.
  - Abruf liefert `not_found` → Zeile mit `status: not_found`; Aufruf am nächsten Tag (`now` + 2 Tage) fragt wieder, am selben Tag nicht.
  - `not_configured` → alle `null`, Status 200.
  - 61 Einträge → 400; `kind: "tv"` → 400; ohne Token → 401.
- [ ] **Step 2: Run** `npx vitest --run src/routes/meta.test.ts` — FAIL.
- [ ] **Step 3: Implement** Service, Route, Einbindung.
- [ ] **Step 4: Run** — PASS; `npm test`, `npm run build` grün.
- [ ] **Step 5: Commit** `feat(meta): POST /meta with cached MDBList lookups`

### Task 4: Web — Datenabruf und Anzeige-Bausteine

**Files:**
- Modify: `openmedia-web/src/lib/backend.ts` (Typen + `getTitleMeta`)
- Create: `openmedia-web/src/lib/title-meta-client.ts` (Sammler + Cache)
- Create: `openmedia-web/src/hooks/use-title-meta.ts`
- Create: `openmedia-web/src/components/meta/origin-label.tsx`, `tile-ratings.tsx`, `all-ratings.tsx`
- Test: `src/lib/title-meta-client.test.ts`, `src/components/meta/meta.test.tsx`

**Interfaces:**
- Consumes: `POST /meta` (Task 3), Antwortform `TitleMetaView`.
- Produces:
  - In `backend.ts`: `export type Brand = …` (wie API), `export interface Ratings { critics?; audience?; imdb?; metacritic?; letterboxd?; trakt?; tmdb? }` (alle `number`), `export interface TitleMeta { kind: "movie" | "show"; tmdbId: number; label: string | null; brand: Brand | null; ratings: Ratings; score: number | null }`, `export function getTitleMeta(items: { kind: "movie" | "show"; tmdbId: number }[], token: string)` → `{ items: TitleMeta[] }`.
  - `title-meta-client.ts`: `export function loadTitleMeta(kind: "movie" | "show", tmdbId: number): Promise<TitleMeta | null>` — sammelt alle Aufrufe innerhalb von 30 ms, teilt in Blöcke zu 60, eine Anfrage je Block, merkt Ergebnisse für die Sitzung (gleiche ID → keine neue Anfrage), Fehler → `null`, Token über `getToken()`, ohne Token → `null`. Dazu `export function __resetTitleMetaCache()` für Tests.
  - `use-title-meta.ts`: `export function useTitleMeta(kind: "movie" | "show", tmdbId: number): TitleMeta | null`.
  - `OriginLabel` Props `{ meta: TitleMeta | null; className?: string }` — kleine farbige Fläche je `brand`, nichts bei `label: null`.
  - `TileRatings` Props `{ meta: TitleMeta | null }` — `🍅 36% · 🍿 92%`, `aria-label` „Kritiker 36 Prozent, Zuschauer 92 Prozent“ (nur vorhandene Teile), nichts wenn beide fehlen.
  - `AllRatings` Props `{ meta: TitleMeta | null }` — Liste: 🍅 Kritiker %, 🍿 Zuschauer %, IMDb x.x, Metacritic n, Letterboxd x.x, Trakt n%, TMDB n%, MDBList n (`score`); nur vorhandene.

- [ ] **Step 1: Write the failing tests:**
  - Sammler (fake timers, `getTitleMeta` gemockt): 3 Aufrufe im selben Moment → 1 Anfrage mit 3 Einträgen; erneuter Aufruf derselben ID → keine Anfrage; 130 IDs → 3 Anfragen mit 60/60/10; Anfrage wirft → alle Promises `null`; kein Token → `null` ohne Anfrage.
  - `TileRatings` mit `{critics:36,audience:92}` → Text `🍅 36% · 🍿 92%`, `aria-label` wie oben; nur `audience` → `🍿 92%`; keine Werte → leer.
  - `OriginLabel` `{label:"NETFLIX",brand:"netflix"}` → Text NETFLIX; `label: null` → nichts.
  - `AllRatings` mit `imdb: 6.3, score: 56` → enthält „IMDb 6.3“ und „MDBList 56“.
- [ ] **Step 2: Run** `npx vitest --run src/lib/title-meta-client.test.ts src/components/meta` — FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `npm test`, `npm run lint`, `npm run build` — grün.
- [ ] **Step 5: Commit** `feat(meta): web title meta loader and display components`

### Task 5: Web — Kacheln, Hover, Hero und Detailseite

**Files:**
- Modify: `openmedia-web/src/components/movie/movie-card.tsx`, `components/show/show-card.tsx`, `components/library/library-card.tsx`, `components/home/continue-row.tsx`, `components/preview/preview-tile.tsx`, `components/movie/hero-banner.tsx`, `components/movie/movie-detail-hero.tsx`
- Create (falls nötig): `openmedia-web/src/components/meta/title-meta-badges.tsx` (Client-Baustein für die Server-Komponenten Hero und Detail)
- Test: bestehende Tests der Komponenten anpassen; neu `src/components/movie/movie-card.test.tsx` (falls nicht vorhanden)

**Interfaces:**
- Consumes: `useTitleMeta`, `OriginLabel`, `TileRatings`, `AllRatings` (Task 4).
- Produces:
  - Kacheln: Label oben links auf dem Poster, `TileRatings` statt Stern in der Infozeile (feste Höhe, damit nichts springt). Serien (`show-card`, Folgen in `continue-row`, Folgen in `library-card`) mit `kind: "show"` und der Serien-TMDB-ID.
  - `PreviewTile`: Prop `voteAverage` entfällt; zeigt `AllRatings` (Daten über `useTitleMeta(kind === "tv" ? "show" : "movie", id)`).
  - `hero-banner`, `movie-detail-hero` (Server-Komponenten): Stern raus, `TitleMetaBadges` (Client) mit Label + `AllRatings` rein; `movie-detail-hero` bekommt dafür `kind`.

- [ ] **Step 1: Write the failing tests:** `MovieCard` mit gemocktem `useTitleMeta` → zeigt NETFLIX und `🍅 36% · 🍿 92%`, kein `★`; `ShowCard` fragt mit `kind: "show"`; `ContinueRow` fragt für eine Folge mit der Serien-ID und `kind: "show"`; `PreviewTile` zeigt „IMDb 6.3“.
- [ ] **Step 2: Run** — FAIL.
- [ ] **Step 3: Implement**; `grep -rn "vote_average.toFixed\|voteAverage.toFixed" src/components` muss danach leer sein.
- [ ] **Step 4: Run** `npm test`, `npm run lint`, `npm run build` — grün. Lokal im Browser ansehen (API mit Test-Key oder gemockt): Kachel, Hover, Hero, Detail; Farben der Marken prüfen.
- [ ] **Step 5: Commit** `feat(meta): origin label and Rotten Tomatoes ratings on web tiles`

### Task 6: tvOS — OpenMediaKit

**Files:**
- Modify: `openmedia-tvos/OpenMediaKit/Sources/OpenMediaKit/Models.swift`, `APIClient.swift`
- Create: `openmedia-tvos/OpenMediaKit/Sources/OpenMediaKit/TitleMeta.swift`
- Test: `openmedia-tvos/OpenMediaKit/Tests/OpenMediaKitTests/TitleMetaTests.swift`, Ergänzung `APIClientTests.swift`

**Interfaces:**
- Consumes: `POST /meta` (Task 3).
- Produces:
  - `public enum MetaKind: String, Codable, Sendable { case movie, show }`
  - `public struct MetaRatings: Codable, Sendable, Equatable { critics, audience, imdb, metacritic, letterboxd, trakt, tmdb: Double? }`
  - `public struct TitleMeta: Codable, Sendable, Equatable, Identifiable { kind: MetaKind; tmdbId: Int; label: String?; brand: String?; ratings: MetaRatings; score: Double?; id: "\(kind)-\(tmdbId)" }`
  - `APIClient.meta(_ items: [(kind: MetaKind, tmdbId: Int)]) async throws -> [TitleMeta]` — teilt in Blöcke zu 60, `POST /meta`, Ergebnisse in Reihenfolge zusammengefügt.
  - `public func tileRatingsText(_ r: MetaRatings) -> String?` (`"🍅 36% · 🍿 92%"`, `nil` wenn beide fehlen), `public func tileRatingsAccessibility(_ r: MetaRatings) -> String?`, `public func allRatingsText(_ meta: TitleMeta) -> [String]` (gleiche Reihenfolge und Texte wie Web `AllRatings`).

- [ ] **Step 1: Write the failing tests:** Texte wie im Web (Task 4); `MetaRatings` dekodiert fehlende Felder als `nil`; `meta` mit 130 Einträgen → 3 `POST /meta` mit 60/60/10 (StubURLProtocol), Body-Felder `kind`/`tmdbId`.
- [ ] **Step 2: Run** `cd OpenMediaKit && swift test` — FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `swift test` — PASS.
- [ ] **Step 5: Commit** `feat(kit): title meta model, API call and rating texts`

### Task 7: tvOS — Kacheln und Detailseiten

**Files:**
- Modify: `openmedia-tvos/OpenMedia/Components/PosterGrid.swift`, `PosterCard.swift`, `ContinueRow.swift`, `OpenMedia/Screens/MovieDetailView.swift`, `ShowDetailView.swift`
- Create (falls nötig): `openmedia-tvos/OpenMedia/Components/OriginBadge.swift` (Marke → Farbe)

**Interfaces:**
- Consumes: Task 6.
- Produces:
  - `PosterGrid` lädt beim Erscheinen `AppServices.api.meta(...)` für alle `items` (aus `MediaLink.movie/.show`), hält `[String: TitleMeta]` und gibt jedem `PosterCard` sein `meta` (`TitleMeta?`).
  - `PosterImage` bekommt optional `meta`: Label oben links auf dem Poster, `tileRatingsText` unter dem Titel (feste Höhe), `accessibilityLabel` aus `tileRatingsAccessibility`.
  - `ContinueRow`: dasselbe (Folgen → `.show` mit Serien-ID).
  - Detailseiten: Label + `allRatingsText` als Zeile unter `metaLine` bzw. unter der Jahreszeile der Serie.

- [ ] **Step 1: Implement.**
- [ ] **Step 2: Build** `xcodebuild -scheme OpenMedia -destination 'generic/platform=tvOS Simulator' build` (vorher `xcodegen generate`) — `BUILD SUCCEEDED`; `swift test` — PASS.
- [ ] **Step 3: Manuelle Checkliste** in den Report (Gerät `Wohnzimmer`): Label und 🍅/🍿 auf Kacheln in Start, Filme, Serien, Suche, Watchlist, Weiterschauen; Detailseiten mit allen Werten; Fokus-Rahmen überdeckt das Label nicht.
- [ ] **Step 4: Commit** `feat(tvos): origin label and Rotten Tomatoes ratings on tiles and details`

### Task 8: Auslieferung (nur mit Freigabe des Betreibers)

- [ ] **Step 1:** Fast-Forward `main` + Push für API und Web, tvOS lokal `main`.
- [ ] **Step 2:** Prod: Backup `pg_dump -Fc` nach `/opt/backup/…-before-title-meta.dump`, Rollback-Images `pre-title-meta` taggen, Migration mit dem Builder-Image (`--network openmedia_default`, `npx prisma migrate deploy`), dann `docker compose build api web` und `up -d --no-deps api web` (Wiki `openmedia/openmedia-deployment.md`).
- [ ] **Step 3:** Key aus 1Password (`MDBList API Key`, Feld `Anmeldedaten`) als verschlüsselten Eintrag `metadata/mdblistApiKey` setzen — ohne den Key auszugeben: entweder der Betreiber trägt ihn in der Admin-Seite ein, oder per `docker exec -i openmedia-api node` mit der Schreibfunktion aus `dist/lib/config-service.js` (Key über stdin, Route `PUT /admin/config/entries` braucht sonst ein Admin-Token).
- [ ] **Step 4:** Prüfen: `/health` 200; `POST /meta` ohne Token 401; mit Token für `{movie 512195, show 1399}` → NETFLIX und HBO mit Werten; Tabelle `title_meta` hat 2 Zeilen.
- [ ] **Step 5:** tvOS bauen und auf `Wohnzimmer` installieren.
- [ ] **Step 6:** Wiki-Log + Deploy-Historie; Spec-Status auf „umgesetzt“.

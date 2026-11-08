# Vorschau-Kachel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bleibt die Maus etwa 1 Sekunde auf einer Film- oder Serien-Kachel, dann öffnet sich eine große Vorschau mit stummem Trailer, Infos und Knöpfen. Bei Filmen kann man direkt laden, bei Serien die „nächste Folge“.

**Architecture:** Die API liefert zusätzlich, welche Folgen der Nutzer je in der Bibliothek hatte (`myLibrary`). In openmedia-web kommen drei reine Bausteine dazu: Trailer-Auswahl (`lib/trailer.ts`), „nächste Folge“ (`lib/next-episode.ts`) und eine Server Action für die Vorschau-Daten (`app/actions/preview.ts`). Eine Komponente `PreviewTile` umschließt die bestehenden Kacheln. Sie öffnet nach 1 s ein Portal über der Seite. Film- und Serien-Knöpfe sind eigene kleine Komponenten, die die bestehende Download- und Anfrage-Logik nutzen.

**Tech Stack:** openmedia-api (Express 5, Prisma 7, PostgreSQL 16, Vitest + Supertest), openmedia-web (Next.js 16.4, React 19, Vitest + Testing Library, jsdom).

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-08-vorschau-kachel-design.md`

## Global Constraints

- Repos unter `/Users/jschattka/git_private/cinescope/`. Pro Repo (api, web) ein Branch `feature/vorschau-kachel` von frischem `origin/main`.
- Alle bestehenden Tests bleiben grün. Kein neues npm-Paket.
- API-Tests nur gegen die Test-DB `localhost:5433` (`docker compose up -d db-test`). Web-Tests rufen nie echtes TMDB oder die API auf (`fetch` bzw. Module mocken).
- Der TMDB-Key bleibt auf dem Server. Der Browser holt TMDB-Daten nur über die Server Action.
- Raster bleibt wie heute. Keine Watchlist für Serien, kein Film-Indexer, keine Vorschau auf Touch-Geräten.
- UI-Texte auf Deutsch, exakt: `Nächste: Staffel {s}, Folge {e}`, `Alles da`, `Herunterladen`, `Mehr`, `Ton an`, `Ton aus` (aria-labels des Lautsprecher-Knopfs).
- Optik in der bestehenden Palette (`src/app/globals.css`, `cinema-gold`). Aufbau wie Entwurf `openmedia-docs/.superpowers/brainstorm/21663-1794130341/content/vorschau-art.html`, Variante A.

## Review Focus

1. **Maus fährt schnell über das Raster:** Keine Vorschau öffnet sich, und `getPreview` wird nicht aufgerufen. → Test in Task 5.
2. **Kachel in der letzten Spalte oder ganz unten:** Die Vorschau bleibt ganz im Viewport. → Test in Task 5 (`placePreview`).
3. **Serie, die noch niemand angefragt hat (API antwortet 404):** Die Kachel bietet `Staffel 1, Folge 1` an, ohne Fehler-Toast. → Test in Task 7.
4. **Titel ohne Trailer und ohne Backdrop:** Die Vorschau zeigt das Backdrop bzw. einen neutralen Platzhalter, kein leeres iframe. → Test in Task 5.
5. **Nicht angemeldet:** Trailer und Infos erscheinen, es gibt keinen API-Aufruf, und der Knopf führt auf `/login`. → Test in Task 7.

---

### Task 1: `myLibrary` in `GET /nzb/shows/by-tmdb/:tmdbId` (openmedia-api)

**Files:**
- Modify: `src/routes/nzb.ts:204-250`
- Test: `src/routes/nzb.test.ts` (Block `describe("GET /nzb/shows/by-tmdb/:tmdbId")`)

**Interfaces:**
- Produces: Antwort bekommt zusätzlich `myLibrary: { season: number; episode: number }[]`. Das sind alle `UserLibrary`-Einträge des Nutzers zu Folgen dieser Serie, **auch mit `removedAt`**. Sortiert nach Staffel, dann Folge, ohne Duplikate (zwei Versionen derselben Folge → ein Eintrag).

- [ ] **Step 1: Failing test** `it("liefert myLibrary inklusive gelöschter Einträge, nur vom eigenen Nutzer")`. Aufbau: Serie `tmdbId: 95398` mit Folgen S01E01, S01E02, S01E03. Je Folge ein `NzbFile` (Pflichtfelder laut `prisma/schema.prisma`, `episodeId` gesetzt). Eigener Nutzer: S01E02 aktiv, S01E01 mit `removedAt: new Date()`, außerdem eine zweite Version von S01E02 in der Bibliothek. Fremder Nutzer: S01E03.
  ```ts
  expect(res.body.myLibrary).toEqual([{ season: 1, episode: 1 }, { season: 1, episode: 2 }]);
  ```
- [ ] **Step 2:** `npx vitest --run src/routes/nzb.test.ts` → FAIL (`myLibrary` ist `undefined`).
- [ ] **Step 3:** In der Route eine Abfrage `prisma.userLibrary.findMany({ where: { userId, nzbFile: { episode: { showId: show.id } } }, select: { nzbFile: { select: { episode: { select: { season: true, episode: true } } } } } })` ergänzen, deduplizieren, sortieren, als `myLibrary` mitsenden.
- [ ] **Step 4:** Test erneut → PASS. Ganze Suite `npm test` → grün.
- [ ] **Step 5: Commit** `feat(nzb): return user's episode library history for a show`

---

### Task 2: Trailer-Auswahl und TMDB-Felder (openmedia-web)

**Files:**
- Create: `src/lib/trailer.ts`, `src/lib/trailer.test.ts`
- Modify: `src/lib/tmdb.ts` (Typen `Video`, `ShowDetail`; `getMovieDetails`, `getShowDetails`)
- Modify: `src/lib/tmdb.test.ts`
- Modify: `src/components/movie/trailer-section.tsx` (nutzt `pickTrailer`)

**Interfaces:**
- Produces: `Video` bekommt `iso_639_1: string | null`. `ShowDetail` bekommt `last_episode_to_air: { season_number: number; episode_number: number } | null`.
- Produces: `pickTrailer(videos: Video[]): Video | null` in `src/lib/trailer.ts`.

- [ ] **Step 1: Failing tests** in `trailer.test.ts`:
  - nur `site === "YouTube"` und `type === "Trailer"` zählen (Teaser, Vimeo → `null`)
  - Reihenfolge: `de` + official > `de` > `en` + official > `en` > Rest
  - leere Liste → `null`

  In `tmdb.test.ts`: `getMovieDetails(1)` und `getShowDetails(1)` rufen eine URL mit `include_video_language=de%2Cen` auf (bzw. `de,en`, je nach `URLSearchParams`-Kodierung prüfen) **und** `append_to_response=credits,videos,similar` wie bisher.
- [ ] **Step 2:** `npx vitest --run src/lib/trailer.test.ts src/lib/tmdb.test.ts` → FAIL.
- [ ] **Step 3:** `pickTrailer` implementieren. In beiden Detail-Abfragen `include_video_language: "de,en"` ergänzen. Typen erweitern. `TrailerSection` ruft `pickTrailer(videos)` statt eigener Filterlogik.
- [ ] **Step 4:** Tests → PASS. `npm test` → grün.
- [ ] **Step 5: Commit** `feat(tmdb): prefer German trailers and expose last aired episode`

---

### Task 3: „Nächste Folge“ (openmedia-web)

**Files:**
- Create: `src/lib/next-episode.ts`, `src/lib/next-episode.test.ts`
- Modify: `src/lib/backend.ts` (Rückgabetyp von `getNzbShowByTmdb` um `myLibrary` erweitern)

**Interfaces:**
- Consumes: `myLibrary` aus Task 1.
- Produces:
  ```ts
  export interface EpisodeCode { season: number; episode: number }
  export function nextEpisode(
    had: EpisodeCode[],
    seasons: { season_number: number; episode_count: number }[],
    lastAired: EpisodeCode | null,
  ): EpisodeCode | null
  ```
  `null` bedeutet „Alles da“.

Regel (Spec Abschnitt 7): Staffel 0 zählt nie. Maßgeblich ist die höchste Folge in `had` nach (Staffel, Folge). Kandidat ist die Folge danach; liegt sie hinter `episode_count` der Staffel, dann Folge 1 der nächsten Staffel mit `episode_count > 0`. Ist `had` leer, dann Folge 1 der ersten Staffel ≥ 1. Kandidat nach `lastAired`, ohne passende Staffel oder `lastAired === null` → `null`.

- [ ] **Step 1: Failing tests** (Staffeln: S1 = 10 Folgen, S2 = 8 Folgen, `lastAired` = S2E3, wenn nicht anders genannt):
  - `had` S1E1–E3 → `{1,4}` (Spec-Beispiel: geschaut und gelöscht)
  - `had` nur S1E5 → `{1,6}` (Lücken werden übersprungen)
  - `had` leer → `{1,1}`
  - `had` S1E10 → `{2,1}`
  - `had` S2E3 → `null`
  - `had` nur S0E1 → `{1,1}`
  - `lastAired` = `null` → `null`
  - Reihenfolge von `had` egal (S1E3, S1E1 → `{1,4}`)
- [ ] **Step 2:** `npx vitest --run src/lib/next-episode.test.ts` → FAIL.
- [ ] **Step 3:** Implementieren, Typ in `backend.ts` ergänzen: `{ show: NzbShowInfo; myRequests: EpisodeRequestView[]; myLibrary: EpisodeCode[] }`.
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5: Commit** `feat(series): compute the next episode for the preview tile`

---

### Task 4: Server Action für Vorschau-Daten (openmedia-web)

**Files:**
- Create: `src/app/actions/preview.ts`, `src/app/actions/preview.test.ts`

**Interfaces:**
- Consumes: `getMovieDetails`, `getShowDetails`, `pickTrailer` (Task 2).
- Produces:
  ```ts
  "use server";
  export interface PreviewData {
    trailerKey: string | null;
    backdropPath: string | null;
    overview: string;
    genres: string[];            // höchstens 2 Namen
    numberOfSeasons: number | null; // nur Serien
    seasons: { season_number: number; episode_count: number }[]; // nur Serien, sonst []
    lastAired: { season: number; episode: number } | null;         // nur Serien
  }
  export async function getPreview(kind: "movie" | "tv", id: number): Promise<PreviewData | null>
  ```
  Fehler von TMDB → `null` (geloggt mit `console.error`, nicht geworfen). Ungültige `id` (keine positive Ganzzahl) → `null` ohne TMDB-Aufruf.

- [ ] **Step 1: Failing tests** mit gemocktem `@/lib/tmdb` (Muster: `src/app/actions/tmdb-search.test.ts`): Film-Mapping inkl. `trailerKey` aus `pickTrailer`; Serien-Mapping inkl. `lastAired` aus `last_episode_to_air`; TMDB wirft → `null`; `id = 0` / `1.5` → `null` und kein Aufruf.
- [ ] **Step 2:** `npx vitest --run src/app/actions/preview.test.ts` → FAIL.
- [ ] **Step 3:** Implementieren.
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5: Commit** `feat(preview): add server action for preview tile data`

---

### Task 5: `PreviewTile` – Hover, Portal, Trailer (openmedia-web)

**Files:**
- Create: `src/components/preview/preview-tile.tsx`, `src/components/preview/place-preview.ts`
- Test: `src/components/preview/preview-tile.test.tsx`, `src/components/preview/place-preview.test.ts`

**Interfaces:**
- Consumes: `getPreview`, `PreviewData` (Task 4), `getBackdropUrl` aus `src/lib/tmdb.ts:325` (bestehend).
- Produces:
  ```ts
  export function placePreview(
    tile: { left: number; top: number; width: number; height: number },
    viewport: { width: number; height: number },
    size: { width: number; height: number },
  ): { left: number; top: number }

  export function PreviewTile(props: {
    kind: "movie" | "tv";
    id: number;
    title: string;
    voteAverage: number;
    year: string;
    children: React.ReactNode;                       // die bestehende Kachel
    actions: (data: PreviewData | null) => React.ReactNode; // Knöpfe, Tasks 6/7
  }): JSX.Element
  ```

Festlegungen:
- Öffnen nach `1000` ms ununterbrochenem Verweilen. Nur wenn `window.matchMedia("(hover: hover)").matches`, sonst rendert `PreviewTile` nur `children`.
- `placePreview`: Breite der Vorschau `360` px, Abstand zum Rand `8` px. Horizontal mittig über der Kachel, dann in `[8, viewport.width - 8 - size.width]` geklemmt. Vertikal: `tile.top - 0.1 * tile.height`, dann in `[8, viewport.height - 8 - size.height]` geklemmt.
- Portal (`createPortal` nach `document.body`), `position: fixed`. Schließt bei `mouseleave` aus Kachel **und** Vorschau zusammen, bei Scroll und bei `Escape`.
- Daten: `getPreview` erst beim Öffnen, pro `kind:id` in einer Modul-`Map<string, Promise<PreviewData | null>>` gemerkt (einmal pro Seitenleben).
- Trailer: `https://www.youtube-nocookie.com/embed/{key}?autoplay=1&mute=1&controls=0&playsinline=1&loop=1&playlist={key}&enablejsapi=1`. Lautsprecher-Knopf schickt per `postMessage` `{"event":"command","func":"unMute"|"mute","args":[]}` an das iframe; aria-label `Ton an` / `Ton aus`. Kein Trailer → Backdrop-Bild; kein Backdrop → neutraler Platzhalter (`bg-muted`).
- Inhalt unter dem Trailer: Titel, `★ {voteAverage.toFixed(1)}`, Jahr, Genres, bei Serien `{n} Staffeln`, Beschreibung auf 3 Zeilen gekürzt (`line-clamp-3`), dann `actions(data)`.

- [ ] **Step 1: Failing tests** für `placePreview`: Kachel in der Mitte → mittig; Kachel ganz rechts (`left = viewport.width - 100`) → `left = viewport.width - 8 - 360`; Kachel ganz links → `left = 8`; Kachel ganz unten → `top + size.height <= viewport.height - 8`.
- [ ] **Step 2: Failing tests** für `PreviewTile` (Fake-Timer, `getPreview` gemockt, `matchMedia` gestubbt):
  - `mouseEnter`, nach 999 ms keine Vorschau, nach 1000 ms Vorschau (`role="dialog"`, Name = Titel)
  - `mouseEnter`, nach 500 ms `mouseLeave`, weitere 1000 ms → keine Vorschau, `getPreview` nie aufgerufen
  - zweites Öffnen derselben Kachel → `getPreview` nur einmal aufgerufen
  - `trailerKey: null`, `backdropPath: "/b.jpg"` → `img` statt `iframe`; beide `null` → weder `img` noch `iframe`
  - Klick auf `Ton an` → `postMessage` mit `"unMute"`, Label wechselt zu `Ton aus`
  - `matchMedia` → `false`: nach 1000 ms keine Vorschau
  - Wechsel der Maus von Kachel in die offene Vorschau → bleibt offen; `mouseLeave` aus der Vorschau → schließt
- [ ] **Step 3:** `npx vitest --run src/components/preview` → FAIL.
- [ ] **Step 4:** Implementieren.
- [ ] **Step 5:** Tests → PASS.
- [ ] **Step 6: Commit** `feat(preview): add hover preview tile with muted trailer`

---

### Task 6: Film-Knöpfe und Einbau in `MovieCard` (openmedia-web)

**Files:**
- Modify: `src/components/movie/download-button.tsx` (neues Prop)
- Create: `src/components/preview/movie-preview-actions.tsx`, `src/components/preview/movie-preview-actions.test.tsx`
- Modify: `src/components/movie/movie-card.tsx`

**Interfaces:**
- Consumes: `PreviewTile` (Task 5), `DownloadButton`, `WatchlistButton` (bestehend).
- Produces: `DownloadButton` bekommt `hideWhenUnavailable?: boolean` (Standard `false`). Bei `true` und keiner sichtbaren Version rendert er `null` statt „Nicht verfügbar“. Alles andere bleibt gleich.
- Produces: `MoviePreviewActions({ movie }: { movie: Movie })` → `DownloadButton hideWhenUnavailable`, `WatchlistButton variant="detail"`, Link `Mehr` auf `/movie/{id}`.

- [ ] **Step 1: Failing tests:** `DownloadButton` mit `hideWhenUnavailable` und leerer `checkAvailability` rendert keinen Knopf; ohne das Prop weiter „Nicht verfügbar“ (bestehendes Verhalten). `MoviePreviewActions` zeigt `Mehr` mit `href="/movie/603"`.
- [ ] **Step 2:** Tests → FAIL.
- [ ] **Step 3:** Implementieren. `MovieCard` umschließt seinen bisherigen Inhalt mit `PreviewTile kind="movie"` und `actions={() => <MoviePreviewActions movie={movie} />}`. Der bestehende Link und das Herz auf der kleinen Kachel bleiben.
- [ ] **Step 4:** `npm test` → grün (inkl. bestehender Karten- und Download-Tests).
- [ ] **Step 5: Commit** `feat(preview): add movie actions to preview tile`

---

### Task 7: Serien-Knöpfe und Einbau in `ShowCard` (openmedia-web)

**Files:**
- Create: `src/components/preview/show-preview-actions.tsx`, `src/components/preview/show-preview-actions.test.tsx`
- Modify: `src/components/show/show-card.tsx`, `src/components/show/show-card.test.tsx`

**Interfaces:**
- Consumes: `nextEpisode`, `EpisodeCode` (Task 3), `getNzbShowByTmdb`, `requestEpisodes` aus `src/lib/backend.ts`, `getToken` aus `src/lib/auth.ts`, `PreviewData` (Task 4).
- Produces: `ShowPreviewActions({ showId, data }: { showId: number; data: PreviewData | null })`.

Verhalten:
- Ohne Token: kein API-Aufruf. Knopf `Herunterladen` führt auf `/login`. Link `Mehr` auf `/serie/{id}`.
- Mit Token: `getNzbShowByTmdb(showId)`. Bei 404 gilt `had = []`. Sonst `had = [...myLibrary, ...myRequests.filter(r => r.status !== "expired")]`. Dann `nextEpisode(had, data.seasons, data.lastAired)`.
- Ergebnis `{s, e}` → Text `Nächste: Staffel {s}, Folge {e}` und Knopf `Herunterladen` → `requestEpisodes({ tmdbId: showId, season: s, episode: e })`. Danach `toast.success`; Fehler → `toast.error` mit der Meldung aus der Antwort.
- Ergebnis `null` → Text `Alles da`, kein `Herunterladen`.
- `data === null` (TMDB-Fehler) → nur `Mehr`.

- [ ] **Step 1: Failing tests** (Muster für Mocks: `src/components/show/season-list.test.tsx`):
  - 404 → Text `Nächste: Staffel 1, Folge 1`, kein `toast.error`
  - `myLibrary` S1E1–E3 → `Nächste: Staffel 1, Folge 4`; Klick → `requestEpisodes({ tmdbId, season: 1, episode: 4 }, "tok")`
  - `myRequests` mit `expired` für S1E4 und `myLibrary` S1E1–E3 → weiterhin `Folge 4`
  - Alles bis `lastAired` vorhanden → `Alles da`, kein Knopf `Herunterladen`
  - `getToken` → `null`: kein `getNzbShowByTmdb`-Aufruf, Knopf führt auf `/login`
- [ ] **Step 2:** Tests → FAIL.
- [ ] **Step 3:** Implementieren. `ShowCard` umschließt seinen Inhalt mit `PreviewTile kind="tv"` und `actions={(data) => <ShowPreviewActions showId={show.id} data={data} />}`. `show-card.test.tsx` bleibt grün (ggf. `matchMedia` stubben).
- [ ] **Step 4:** `npm test` → grün.
- [ ] **Step 5: Commit** `feat(preview): add next-episode download to series preview tile`

---

### Task 8: Prüfung im Browser

- [ ] **Step 1:** API und Web lokal starten (`npm run dev` in beiden Repos).
- [ ] **Step 2:** Auf `/` und `/serien` je eine Kachel in der Mitte, ganz rechts und in der untersten Reihe 1 s lang anfahren. Screenshots machen. Erwartet: Vorschau öffnet sich, ist nicht abgeschnitten, der Trailer läuft stumm, `Ton an` schaltet den Ton ein.
- [ ] **Step 3:** Schnell mit der Maus über das Raster fahren → keine Vorschau. In den Netzwerk-Requests gibt es keine Server-Action-Aufrufe.
- [ ] **Step 4:** Eine Serie mit vorhandenen Folgen in der Bibliothek prüfen → `Nächste: …` passt. Zum Schluss `/watchlist`, `/search?q=dark` und „Ähnliche“ auf einer Detailseite kurz ansehen.
- [ ] **Step 5:** `npm run lint` und `npm run build` in openmedia-web, `npm test` in beiden Repos → grün.

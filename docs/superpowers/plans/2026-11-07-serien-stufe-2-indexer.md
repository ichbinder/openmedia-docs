# Serien Stufe 2 (Indexer) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ein Nutzer fordert auf der Serien-Seite eine Folge oder eine ganze Staffel an. openmedia sucht selbst bei Treasure Maps, wählt nach festen Regeln die beste deutsche Version, lädt die NZB und gibt sie in die bestehende Download-Pipeline. Auf Wunsch sieht der Nutzer die Versionsliste einer Folge und wählt selbst.

**Architecture:** Neu sind vier kleine Bausteine in openmedia-api: ein Treasure-Maps-Client (`lib/treasure-maps.ts`), reine Auswahl-Regeln (`lib/release-ranking.ts`), eine Tabelle `EpisodeRequest` mit dem Abarbeiten der Anfragen (`lib/episode-requests.ts`) und die Routen `/series/*`. Damit Filme und Folgen denselben Weg in die Pipeline nehmen, wird der Kern von `POST /downloads/request` vorher als `lib/nzb-ingest.ts` herausgezogen. Nachzügler (fertige Downloads, fehlgeschlagene Jobs, "noch keine deutsche Fassung") räumt der bestehende Reconciler alle 5 Minuten auf. Die Webseite bekommt Anfordern-Knöpfe, einen Status pro Folge und eine aufklappbare Versionsliste.

**Tech Stack:** openmedia-api (Express 5, Prisma 7, PostgreSQL 16, Vitest + Supertest), openmedia-web (Next.js 16.4, Vitest).

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-07-serien-design.md`, Abschnitt 9, Stufe 2. Indexer-Fakten: `llm-wiki/web-sources/treasure-maps-indexer-api.md`. Staffel-Pakete (Stufe 3) und Abo/Versionswechsel (Stufe 4) gehören nicht hierher.

## Global Constraints

- Repos unter `/Users/jschattka/git_private/cinescope/`. Pro Repo (api, web) ein Branch `feature/serien-indexer` von frischem `main`.
- Film-Verhalten ändert sich nicht. Alle bestehenden Tests bleiben grün, besonders `src/routes/downloads-request*.test.ts`.
- API-Tests nur gegen die Test-DB `localhost:5433` (`docker compose up -d db-test`). Tests rufen nie den echten Indexer oder TMDB auf: `fetch` bzw. die Module werden gemockt.
- Der Indexer-Key steht nur in der Admin-Konfiguration: Kategorie `indexer`, Key `treasureMapsApiKey`, verschlüsselt. Lesen mit `getEntry("indexer", "treasureMapsApiKey", true)`. Der Key erscheint nie in Logs, Fehlermeldungen oder Antworten.
- Indexer: Basis `https://treasure-maps.com/api/v1`, Header `X-API-Key`, höchstens 5 Anfragen pro Sekunde (Abstand ≥ 250 ms), Suche immer mit `cat=5100&extended=true`.
- Auswahl (Spec 6.4 und 7.1): Stufen 1080p im Zielbereich > andere 1080p > 2160p > 720p > SD. Zielbereich 1,5 bis 6 GB pro Stunde, lesbar aus Kategorie `series`, Keys `targetGbPerHourMin` / `targetGbPerHourMax`, sonst diese Startwerte.
- Pro Folge darf eine Anforderung höchstens 3 NZBs laden. Liegt der Tagesverbrauch bei `download_max - 50` oder höher, dann wird nichts geladen.
- Cache: Indexer-Suche je Folge 1 h, TMDB-Lookups im API 24 h. Fehler werden nie gecacht.
- UI-Texte auf Deutsch, im Stil der bestehenden Serien-Seite. Kein neues npm-Paket.

## Review Focus

1. **Release ohne MediaInfo und ohne `German`/`DL` im Namen:** wird nie angezeigt und nie geladen (Anti-Ziel 1). → Test in Task 4.
2. **Zwilling ohne Daten ist größer als der Zwilling mit Daten:** Der Zwilling ohne Daten fliegt trotzdem raus. → Test in Task 4.
3. **Tageslimit fast erreicht:** Es wird keine NZB geladen, die Anforderung wartet mit Grund `budget`. → Test in Task 6.
4. **Zwei Nutzer fordern dieselbe Folge an:** Es gibt nur einen `DownloadJob`, und am Ende haben beide die Folge in der Bibliothek. → Test in Task 6.
5. **Kein Key oder HTTP 429:** Die Route antwortet mit klarer Meldung (503 bzw. Status `waiting`), nichts stürzt ab, der Key steht nicht in der Antwort. → Tests in Task 3 und Task 7.

---

### Task 1: Datenmodell EpisodeRequest, indexerGuid und Config-Kategorien (openmedia-api)

**Files:**
- Modify: `prisma/schema.prisma` (NzbFile, User, NzbEpisode, neues Modell nach `UserLibrary`)
- Create: `prisma/migrations/<timestamp>_add_episode_requests/migration.sql`
- Modify: `src/test/setup.ts` (TRUNCATE-Liste)
- Test: `src/lib/series-schema.test.ts`

**Interfaces:**
- Produces: `EpisodeRequest { id, userId → User (Cascade), episodeId → NzbEpisode (Cascade), status String @default("searching"), waitReason String?, manualGuid String?, nzbFileId String? → NzbFile (SetNull), nextSearchAt DateTime?, createdAt, updatedAt }` mit `@@unique([userId, episodeId])`, `@@index([status, nextSearchAt])`, `@@map("episode_requests")`, Spalten snake_case. Status-Werte: `searching | waiting | downloading | done | expired`. `waitReason`: `no_german | budget | indexer_error | null`.
- Produces: `NzbFile.indexerGuid String? @map("indexer_guid")` mit `@@index([indexerGuid])` (nicht unique: dieselbe NZB kann auf verschiedenen Wegen ankommen).

- [ ] **Step 1: Write the failing test** in `src/lib/series-schema.test.ts`: Eine Anforderung für (user, episode) lässt sich anlegen, eine zweite für dasselbe Paar wirft. Wird die NzbFile gelöscht, bleibt die Anforderung mit `nzbFileId = null`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/series-schema.test.ts` → FAIL
- [ ] **Step 3: Implement.** `npx prisma migrate dev --create-only --name add_episode_requests`. An die SQL anhängen, im Muster von `20260422171500_add_vpn_config_category`:
  `INSERT INTO "config_categories" (...) VALUES (gen_random_uuid(), 'indexer', 'Indexer', 'Zugang zu Usenet-Indexern (Treasure Maps)', ...), (gen_random_uuid(), 'series', 'Serien', 'Regeln für die Auswahl von Folgen-Versionen', ...) ON CONFLICT ("name") DO NOTHING;`
  `episode_requests` in die TRUNCATE-Liste vor `nzb_episodes`. `npm run db:generate`.
- [ ] **Step 4: Run** den Test → PASS, dann `npm test` → grün.
- [ ] **Step 5: Commit** `feat(series): add EpisodeRequest and NzbFile.indexerGuid`

---

### Task 2: TTL-Cache und TMDB-TV nach ID (openmedia-api)

**Files:**
- Create: `src/lib/ttl-cache.ts`, Test `src/lib/ttl-cache.test.ts`
- Modify: `src/lib/tmdb.ts` (nach `fetchTmdbEpisode`), Test `src/lib/tmdb-tv.test.ts`

**Interfaces:**
- Produces: `cached<T>(key: string, ttlMs: number, load: () => Promise<T>, keep?: (value: T) => boolean): Promise<T>` und `clearCache(): void` (für Tests). Ein `Map` im Prozess. Eine abgelehnte Promise und ein Wert mit `keep(value) === false` werden nicht gespeichert. Gleichzeitige Aufrufe mit gleichem Key teilen sich eine laufende Promise.
- Produces: `TmdbEpisodeResult` bekommt `runtime: number | null` (TMDB-Feld `runtime`, Minuten).
- Produces: `getTmdbEpisodeById(tmdbId: number, season: number, episode: number): Promise<{ status: "found"; show: TmdbShowResult; episode: TmdbEpisodeResult } | { status: "not_found" } | { status: "error"; reason: string } | { status: "disabled"; reason: string }>`. Eine fehlende Folge ergibt `not_found`.
- Produces: `getTmdbSeasonById(tmdbId: number, season: number)` mit demselben Status-Muster und `{ status: "found"; show: TmdbShowResult; episodes: (TmdbEpisodeResult & { episode: number })[] }` (aus `/tv/{id}/season/{n}`, `de-DE`).
- Beide nutzen `fetchTmdbShowDetails` und `fetchTmdbJson` und sind mit `cached(..., 24h, ..., r => r.status === "found" || r.status === "not_found")` umhüllt.

- [ ] **Step 1: Write the failing tests.** ttl-cache: zweiter Aufruf innerhalb der TTL ruft `load` nicht erneut; nach Ablauf (`vi.useFakeTimers`) doch; nach einem Fehler ruft der nächste Aufruf `load` erneut. tmdb-tv: `getTmdbEpisodeById(95396, 2, 5)` mit gemocktem `fetch` liefert `runtime: 52` und Titel; zweiter Aufruf macht keinen weiteren `fetch`; ein 404 der Folge ergibt `not_found`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/ttl-cache.test.ts src/lib/tmdb-tv.test.ts` → FAIL
- [ ] **Step 3: Implement.** `searchTmdbShowEpisode` liefert danach ebenfalls `runtime`. Bestehende Aufrufer bleiben unverändert.
- [ ] **Step 4: Run** → PASS, `npm test` → grün.
- [ ] **Step 5: Commit** `feat(series): cached TMDB TV lookups by id`

---

### Task 3: Treasure-Maps-Client (openmedia-api)

**Files:**
- Create: `src/lib/treasure-maps.ts`, Test `src/lib/treasure-maps.test.ts`
- Create: `src/lib/__fixtures__/treasure-maps-severance-s02e05.json`

**Interfaces:**
- Produces:
  ```ts
  export interface IndexerRelease {
    guid: string; title: string; size: number; postedAt: string;
    password: number | null; season: number | null; episode: number | null;
    audioLanguages: string[] | null; subtitles: string[] | null;
    videoWidth: number | null;           // aus video.resolution "1918x802"
  }
  export class IndexerError extends Error { constructor(public reason: "not_configured" | "rate_limited" | "http", public status?: number) }
  export function searchEpisode(tmdbId: number, season: number, episode: number): Promise<IndexerRelease[]>;   // GET /tv?tmdbid&season&ep&cat=5100&extended=true&limit=100, 1 h gecacht
  export function downloadNzb(guid: string): Promise<string>;                                               // GET /releases/{guid}/download, folgt dem 302
  export function getDownloadBudget(): Promise<{ used: number; max: number }>;                             // GET /user → limits.download_current / download_max
  ```
- `guid` wird vor jeder URL gegen `/^[a-f0-9]{32}$/` geprüft, sonst `IndexerError("http", 400)`.

- [ ] **Step 1: Fixture anlegen.** Quelle: `/private/tmp/claude-501/-Users-jschattka-git-private-mediatoken/c1fbad2c-f896-4b83-b03f-b1e207d93180/scratchpad/tm/tv-sample.json` (echte Antwort für Severance S02E05, 20 Treffer). Nur `items` und `pagination` übernehmen, `user` und `meta` weglassen.
- [ ] **Step 2: Write the failing tests** mit gemocktem `fetch`:
  - `searchEpisode(95396, 2, 5)` schickt Header `X-API-Key` und die Query oben, liefert 20 Releases. Der erste hat `videoWidth: 1918`, `audioLanguages: ["English","German"]`. Ein Release mit `video: null` hat `videoWidth: null` und `audioLanguages: null`.
  - Zweiter gleicher Aufruf macht keinen `fetch`.
  - Kein Config-Eintrag → `IndexerError` mit `reason "not_configured"`, ohne `fetch`.
  - HTTP 429 → `reason "rate_limited"`. Die Fehlermeldung enthält den Key nicht.
  - Zwei direkt folgende Aufrufe von `getDownloadBudget` liegen ≥ 250 ms auseinander (`vi.useFakeTimers`).
  - `downloadNzb("../x")` wirft ohne `fetch`.
- [ ] **Step 3: Run** `npx vitest --run src/lib/treasure-maps.test.ts` → FAIL
- [ ] **Step 4: Implement.** Abstand über eine Promise-Kette im Modul. Für die Suche `cached` aus Task 2 mit Key `tm:tv:${tmdbId}:${season}:${episode}`.
- [ ] **Step 5: Run** → PASS. **Commit** `feat(series): Treasure Maps indexer client`

---

### Task 4: Auswahl-Regeln (openmedia-api)

**Files:**
- Create: `src/lib/release-ranking.ts`, Test `src/lib/release-ranking.test.ts`

**Interfaces:**
- Consumes: `IndexerRelease` (Task 3), `parseNzbName` (`src/lib/nzb-parser.ts`).
- Produces:
  ```ts
  export type QualityTier = "2160p" | "1080p" | "720p" | "480p";
  export interface RankedRelease extends IndexerRelease {
    qualityTier: QualityTier; inTargetBand: boolean; gbPerHour: number;
    germanSubtitles: boolean; audioDescription: boolean;
    languageSource: "mediainfo" | "name"; releaseType: string | null;
  }
  export function rankReleases(releases: IndexerRelease[], opts: { season: number; episode: number; runtimeMinutes: number; band: { min: number; max: number } }): RankedRelease[];
  export function getTargetBand(): Promise<{ min: number; max: number }>;   // Config "series", Startwerte 1.5 / 6
  ```
- Regeln, in dieser Reihenfolge:
  1. Raus: `season`/`episode` passen nicht (Staffel-Pakete haben `episode: null` und fliegen in Stufe 2 raus), `password === 2`.
  2. Zwillinge: Hat ein Release `audioLanguages === null` und gibt es ein Release mit gleichem `title` (Groß/Klein egal) und Daten, dann fliegt das ohne Daten raus.
  3. Deutsch: Mit Daten muss `audioLanguages` "German" enthalten (`languageSource: "mediainfo"`). Ohne Daten muss der Titel `German`, `Deutsch` oder `DL` als eigenes Wort (Trenner `.` `_` `-` Leerzeichen) enthalten (`"name"`). Sonst raus.
  4. Stufe nach Bildbreite: ≥ 3000 → 2160p, ≥ 1700 → 1080p, ≥ 1200 → 720p, sonst 480p. Ohne Breite: `resolveQualityTier(parseNzbName(title).resolution)`, sonst 480p.
  5. `gbPerHour = size / 1e9 / (runtimeMinutes / 60)`. `inTargetBand` = 1080p und im Bereich.
  6. Sortierung: Rang (1080p im Bereich 5, 1080p 4, 2160p 3, 720p 2, 480p 1) absteigend. Dann Bonus `(germanSubtitles ? 2 : 0) + (audioDescription ? 1 : 0)` absteigend. Dann im Bereich die größere Datei zuerst, außerhalb der kleinere Abstand zum Bereich zuerst.
  7. `audioDescription`: Titel enthält `Hörfilm`, `Hoerfilm`, `Audiodeskription` oder `AD` als eigenes Wort. `germanSubtitles`: `subtitles` enthält "German".

- [ ] **Step 1: Write the failing table tests** mit dem Fixture aus Task 3 und `runtimeMinutes: 50`, Band 1,5/6:
  - Ergebnis hat 12 Einträge. Die ersten drei GUIDs in dieser Reihenfolge: `17c315c9…` (1080p DV, 3,01 GB, Untertitel), `73af81be…` (1080p, 2,6 GB, Untertitel), `eaa01ef2…` (1080p H264, 4,41 GB, nur Name). Platz 4 ist `41167b5c…` (1080p, 1,02 GB, unter dem Bereich).
  - Die Zwillinge ohne Daten `25f4bc84`, `2da553c3`, `38974f14`, `3e1f89d0`, `8562c9c2`, `6f950338`, `fc25c9b5`, `cc6e62d6` fehlen.
  - Die letzten zwei haben `qualityTier: "480p"`.
  - Synthetisch: `audioLanguages: ["English"]` → raus; `audioLanguages: null` mit Titel `Show.S02E05.1080p.WEB.x264-GRP` ohne Zwilling → raus; `password: 2` → raus; `episode: 6` → raus; nur ein SD-Release vorhanden → genau dieses kommt zurück.
  - Bonus: zwei 1080p im Bereich, das kleinere mit `subtitles: ["German"]` und `…Hoerfilm…` im Titel, gewinnt.
- [ ] **Step 2: Run** `npx vitest --run src/lib/release-ranking.test.ts` → FAIL
- [ ] **Step 3: Implement** (reine Funktion, nur `getTargetBand` liest die Config).
- [ ] **Step 4: Run** → PASS. **Commit** `feat(series): release ranking rules`

---

### Task 5: NZB-Übernahme als Bibliotheksfunktion herausziehen (openmedia-api)

Reiner Umbau. Das Verhalten von `POST /downloads/request` bleibt gleich, die bestehenden Tests ändern sich nicht.

**Files:**
- Create: `src/lib/nzb-ingest.ts`
- Modify: `src/routes/downloads.ts:373-794` (Route ruft nur noch `ingestNzb` und übersetzt das Ergebnis in HTTP)
- Modify: `src/lib/episode-resolver.ts` (Ergebnis trägt `season`/`episode`)
- Test: bestehende `src/routes/downloads-request*.test.ts`, neu `src/lib/nzb-ingest.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type IngestResolution =
    | { source: "tmdb-found"; tmdb: TmdbMovieResult }
    | { source: "tmdb-found"; show: TmdbShowResult; season: number; episode: number; tmdbEpisode: TmdbEpisodeResult | null }
    | { source: "needs-review"; reason: "not-found" | "error"; errorDetail?: string }
    | { source: "tmdb-disabled"; reason: string };
  export interface IngestInput {
    nzbContent: string; title: string; filename?: string; userId: string | null;
    resolve: (parsed: ParsedNzbName) => Promise<IngestResolution>;
    fileData?: { indexerGuid: string; qualityTier: string; resolution: string; fileSize: bigint; audioLanguages: string[]; subtitleLanguages: string[]; releaseType: string | null };
  }
  export type IngestResult =
    | { kind: "already-available"; file: NzbFile & { movie: NzbMovie | null } }
    | { kind: "broken"; file: NzbFile & { movie: NzbMovie | null } }
    | { kind: "conflict"; activeJob: DownloadJob }
    | { kind: "missing" } | { kind: "concurrent" }
    | { kind: "service-unavailable" } | { kind: "tmdb-disabled" }
    | { kind: "created"; job: DownloadJob & { nzbFile: /* include JOB_NZB_FILE_INCLUDE, wie heute */ }; reused: boolean; needsReview: boolean };
  export function ingestNzb(input: IngestInput): Promise<IngestResult>;
  ```
- `ingestNzb` startet für nicht-`needsReview`-Jobs selbst den Provisioner (wie heute, `AUTO_PROVISION !== "false"`).
- Mit `fileData` gelten dessen Werte statt der geparsten. Bei einer schon bekannten NzbFile ohne `indexerGuid` wird `fileData.indexerGuid` nachgetragen.
- `resolveEpisodeFromTmdb` liefert im Fund-Fall `{ source: "tmdb-found", show, season, episode, tmdbEpisode }`. `upsertEpisode` bekommt die Werte daraus statt aus `parsed`.

- [ ] **Step 1: Run** `npx vitest --run src/routes/downloads-request.test.ts src/routes/downloads-request-episode.test.ts` → grün (Ausgangslage notieren).
- [ ] **Step 2: Write the failing test** `src/lib/nzb-ingest.test.ts`: `ingestNzb` mit `resolve` → Folge und `fileData` legt NzbFile mit `episodeId`, `indexerGuid` und `qualityTier` aus `fileData` an. Ein zweiter Aufruf mit derselben NZB liefert `conflict`.
- [ ] **Step 3: Implement** den Umbau. Die Route behält Validierung, Statuscodes und Texte 1:1.
- [ ] **Step 4: Run** `npm test` → alles grün, ohne Änderungen an bestehenden Tests.
- [ ] **Step 5: Commit** `refactor(downloads): extract ingestNzb from /downloads/request`

---

### Task 6: Anforderungen abarbeiten (openmedia-api)

**Files:**
- Create: `src/lib/episode-requests.ts`, Test `src/lib/episode-requests.test.ts`
- Modify: `src/lib/job-reconciler.ts` (`reconcileStaleJobs`, nach `cleanupExpiredReviews`)

**Interfaces:**
- Consumes: `getTmdbEpisodeById`, `getTmdbSeasonById` (Task 2), `searchEpisode`, `downloadNzb`, `getDownloadBudget`, `IndexerError` (Task 3), `rankReleases`, `getTargetBand` (Task 4), `ingestNzb` (Task 5), `upsertEpisode` (`episode-resolver.ts`).
- Produces:
  ```ts
  export type RequestView = { season: number; episode: number; status: string; waitReason: string | null };
  export function requestEpisodes(userId: string, tmdbId: number, season: number, episode?: number, manualGuid?: string): Promise<RequestView[]>;
  export function fulfillEpisodeRequest(requestId: string): Promise<void>;
  export function syncEpisodeRequests(): Promise<void>;
  ```
- `requestEpisodes`: Mit `episode` eine Folge, ohne `episode` alle Folgen der Staffel mit `airDate <= heute` (laut `getTmdbSeasonById`). Legt Serie und Folgen mit `upsertEpisode` an und macht ein Upsert von `EpisodeRequest` pro Folge. Eine vorhandene Anforderung mit Status `expired` oder `waiting` startet neu (`status: "searching"`, `createdAt: now`). `manualGuid` wird gesetzt, wenn er übergeben wird. Bei einer einzelnen Folge wird `fulfillEpisodeRequest` abgewartet. Bei einer Staffel laufen die Folgen nacheinander im Hintergrund. Rückgabe ist der Stand danach. TMDB `not_found` ergibt eine leere Liste.
- `fulfillEpisodeRequest`, in dieser Reihenfolge:
  1. Gibt es für die Folge eine Version mit `s3Key` und Status ≠ `broken` (bei `manualGuid` nur mit diesem `indexerGuid`): beste nach `qualityTier` nehmen, in `UserLibrary` eintragen (Upsert wie in `downloads.ts:1007`), Status `done`.
  2. Hat eine Version der Folge einen aktiven Job (`queued | provisioning | downloading | uploading`): Status `downloading`, `nzbFileId` setzen.
  3. Suchen und ranken (Laufzeit aus TMDB, sonst 45 min). Kandidaten entfernen, deren `indexerGuid` zu einer NzbFile mit Status `broken` gehört. Mit `manualGuid` nur diesen Kandidaten nehmen. Ist keiner übrig, dann `waiting`, Grund `no_german`, `nextSearchAt` = +1 h.
  4. Budget prüfen. Bei `used >= max - 50` folgt `waiting`, Grund `budget`, +1 h.
  5. Bis zu 3 Kandidaten der Reihe nach: `downloadNzb`, dann `ingestNzb` mit fester Folge und `fileData` aus dem Kandidaten. `created` oder `conflict` ergibt `downloading` mit `nzbFileId`. `already-available` macht weiter wie in 1. `broken` nimmt den nächsten Kandidaten.
  6. `IndexerError` oder `service-unavailable` ergibt `waiting`, Grund `indexer_error`, +1 h.
- `syncEpisodeRequests` (jeder Reconciler-Lauf):
  - Alle `downloading`: Hat die NzbFile einen `s3Key`, dann folgen ein Eintrag in `UserLibrary` und `done`. Hat sie keinen aktiven oder `needs_review`-Job mehr, dann `waiting` mit `nextSearchAt = now`.
  - Danach höchstens 20 `waiting` mit `nextSearchAt <= now`, älteste zuerst: Ist `createdAt` älter als 30 Tage, dann `expired`. Sonst `fulfillEpisodeRequest`.

- [ ] **Step 1: Write the failing tests** (Module aus Task 2/3 gemockt, `ingestNzb` echt, `storeNzbInService` gemockt, `AUTO_PROVISION=false`):
  - Einzelne Folge, Fixture-Suche → ein DownloadJob, Anforderung `downloading`, die NzbFile hat `indexerGuid` `17c315c9…`.
  - Zwei Nutzer fordern nacheinander dieselbe Folge an → ein DownloadJob, beide `downloading`. NzbFile bekommt `s3Key`, `syncEpisodeRequests()` → beide `done`, beide in `UserLibrary`.
  - Budget `{ used: 350, max: 400 }` → kein `downloadNzb`-Aufruf, `waiting`/`budget`.
  - Suche ohne deutsche Treffer → `waiting`/`no_german`, `nextSearchAt` etwa +1 h.
  - Staffel mit drei Folgen, die dritte mit `airDate` morgen → zwei Anforderungen.
  - Erste Kandidaten-NZB ist als `broken` bekannt (gleicher Hash) → zweiter Kandidat wird geladen.
  - `waiting` mit `createdAt` vor 31 Tagen → `syncEpisodeRequests` setzt `expired`, keine Suche.
  - Job einer `downloading`-Anforderung ist `failed` → nach `syncEpisodeRequests` ist sie `waiting` mit `nextSearchAt <= now`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/episode-requests.test.ts` → FAIL
- [ ] **Step 3: Implement.** Im Reconciler `syncEpisodeRequests` in eigenem `try/catch` aufrufen, wie die Nachbarn.
- [ ] **Step 4: Run** → PASS, `npm test` → grün.
- [ ] **Step 5: Commit** `feat(series): fulfil episode requests via indexer`

---

### Task 7: Routen `/series` und Anforderungen in der Serien-Antwort (openmedia-api)

**Files:**
- Create: `src/routes/series.ts`, Test `src/routes/series.test.ts`
- Modify: `src/app.ts` (`app.use("/series", seriesRoutes)`), `src/routes/nzb.ts:205-247`, Test `src/routes/nzb.test.ts`

**Interfaces:**
- Consumes: `requestEpisodes` (Task 6), `searchEpisode`, `IndexerError` (Task 3), `rankReleases`, `getTargetBand` (Task 4), `getTmdbEpisodeById` (Task 2).
- Produces:
  - `POST /series/requests` (requireAuth), Body `{ tmdbId, season, episode?, guid? }` → `200 { requests: RequestView[] }`. Prüfen: `tmdbId` und `season` ganze Zahlen (`tmdbId ≥ 1`, `season ≥ 0`), `episode ≥ 1` wenn gesetzt, `guid` nur zusammen mit `episode` und nur `/^[a-f0-9]{32}$/`. Sonst 400.
  - `GET /series/candidates?tmdbId=&season=&episode=` (requireAuth) → `200 { candidates: { guid, title, qualityTier, sizeBytes, gbPerHour, inTargetBand, germanSubtitles, audioDescription, releaseType, languageSource }[] }`.
  - Fehler beider Routen: `IndexerError not_configured` → `503 { error: "Indexer ist nicht eingerichtet.", reason: "indexer_not_configured" }`; andere `IndexerError` → `502 { error: "Indexer gerade nicht erreichbar.", reason: "indexer_error" }`; TMDB `disabled` → 503 wie in `downloads.ts`.
  - `GET /nzb/shows/by-tmdb/:tmdbId` bekommt `myRequests: RequestView[]` für den angemeldeten Nutzer.

- [ ] **Step 1: Write the failing tests:** Validierung (Tabelle mit ungültigen Bodies → 400, u. a. `guid: "../x"`). Ohne Token → 401. Ohne Config → 503 mit `reason`, die Antwort enthält keinen Key. `candidates` mit Fixture → 12 Einträge, der erste ist `17c315c9…`. `by-tmdb` enthält `myRequests` nur vom eigenen Nutzer.
- [ ] **Step 2: Run** `npx vitest --run src/routes/series.test.ts src/routes/nzb.test.ts` → FAIL
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS, `npm test` → grün, `npm run build` → ok.
- [ ] **Step 5: Commit** `feat(series): request and candidate routes`

---

### Task 8: Anfordern und Status auf der Serien-Seite (openmedia-web)

**Files:**
- Modify: `src/lib/backend.ts` (nach `getNzbShowByTmdb`), `src/components/show/season-list.tsx`, Test `src/components/show/season-list.test.tsx`

**Interfaces:**
- Consumes: API aus Task 7.
- Produces in `backend.ts`: `EpisodeRequestView`, `NzbShowInfo.myRequests?: EpisodeRequestView[]`, `requestEpisodes(body: { tmdbId: number; season: number; episode?: number; guid?: string }, token: string)`, `ReleaseCandidate`, `getReleaseCandidates(tmdbId: number, season: number, episode: number, token: string)`.
- Verhalten pro Folge, nur mit Token:
  - Version vorhanden: wie heute.
  - Sonst mit Anforderung ein Hinweis statt Knopf: `searching` "Wird gesucht…", `downloading` "Wird geladen…", `waiting` + `no_german` "Noch keine deutsche Fassung", `waiting` + `budget` "Tageslimit erreicht, später erneut", `waiting` + `indexer_error` "Indexer nicht erreichbar, später erneut", `expired` "Keine Version gefunden" plus Knopf "Erneut anfordern".
  - Sonst, wenn `air_date <= heute`: Knopf "Anfordern". Nach der Antwort zeigt die Folge den neuen Status. 503 `indexer_not_configured` → Toast "Indexer ist nicht eingerichtet".
  - Pro Staffel neben der Überschrift ein Knopf "Staffel anfordern", wenn mindestens eine erschienene Folge ohne Version und ohne Anforderung existiert. Danach Toast "N Folgen angefordert".
  - Folgen ohne `air_date` gelten als nicht erschienen.

- [ ] **Step 1: Write the failing tests** (`backend` gemockt): Knopf "Anfordern" nur bei erschienenen Folgen ohne Version. Klick ruft `requestEpisodes({ tmdbId, season, episode })` und zeigt danach "Noch keine deutsche Fassung" bei `waiting/no_german`. `myRequests` mit `downloading` zeigt "Wird geladen…" ohne Knopf. "Staffel anfordern" ruft `requestEpisodes({ tmdbId, season })`.
- [ ] **Step 2: Run** `npx vitest --run src/components/show/season-list.test.tsx` → FAIL
- [ ] **Step 3: Implement.** Knopf-Stil wie "Laden" (`bg-cinema-gold/20 …`).
- [ ] **Step 4: Run** → PASS, `npm run lint`, `npm test`, `npm run build` → ok.
- [ ] **Step 5: Commit** `feat(series): request episodes and seasons from the show page`

---

### Task 9: Versionsliste pro Folge (openmedia-web)

**Files:**
- Create: `src/components/show/episode-candidates.tsx`, Test `src/components/show/episode-candidates.test.tsx`
- Modify: `src/components/show/season-list.tsx`

**Interfaces:**
- Consumes: `getReleaseCandidates`, `requestEpisodes` (Task 8).
- Produces: `<EpisodeCandidates tmdbId season episode onRequested={(view: EpisodeRequestView) => void} />`. Lädt erst beim Aufklappen. Zeile: `{qualityTier} · {releaseType} · {Größe}` mit Größe als `x,y GB` (`toLocaleString("de-DE", { maximumFractionDigits: 1 })`). Dazu kleine Marken "DE-Untertitel" und "Hörfilm", wenn gesetzt, und "Sprache nur laut Name" bei `languageSource: "name"`. Der erste Eintrag trägt die Marke "Empfohlen". Knopf "Diese laden" ruft `requestEpisodes({ tmdbId, season, episode, guid })`. Leere Liste → "Keine deutsche Version gefunden". 503/502 → die Fehlermeldung aus der Antwort.
- In `season-list.tsx`: Link "Versionen" pro erschienener Folge (auch wenn schon eine Version da ist), klappt `EpisodeCandidates` darunter auf.

- [ ] **Step 1: Write the failing tests:** Zeile für `{ qualityTier: "1080p", releaseType: "WEB-DL", sizeBytes: 3010000000 }` zeigt "1080p · WEB-DL · 3 GB". Der erste Eintrag zeigt "Empfohlen". "Diese laden" schickt `guid`. Leere Liste zeigt den Hinweis.
- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS, `npm run lint`, `npm test`, `npm run build` → ok.
- [ ] **Step 5: Commit** `feat(series): per-episode version list`

---

### Task 10: Durchstich (Betreiber, nach Merge und Deploy)

- [ ] Migration auf Prod wie in `llm-wiki/homelab-updates/node-project-dependency-update-traps.md` ("openmedia-api schema changes do not reach prod by themselves"): Backup, Probe auf Kopie, `prisma migrate deploy`. Danach API und Web deployen.
- [ ] Admin → Konfiguration → Kategorie "Indexer": Eintrag `treasureMapsApiKey`, verschlüsselt, Wert aus `https://treasure-maps.com/apihelp`.
- [ ] Auf `/serie/95396` bei S02E05 "Versionen" öffnen: Die Liste zeigt die 1080p-Versionen oben. Danach "Anfordern". Auf `/downloads` erscheint der Job. Nach dem Download steht die Folge in der Bibliothek und in Jellyfin "openmedia Serien".

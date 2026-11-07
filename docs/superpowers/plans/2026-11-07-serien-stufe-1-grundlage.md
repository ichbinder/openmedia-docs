# Serien Stufe 1 (Grundlage) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Folgen einer Serie gelangen über den bestehenden manuellen NZB-Weg in openmedia. Sie erscheinen als Serie mit Staffeln und Folgen auf der Webseite und in einer eigenen Jellyfin-Serien-Bibliothek. Alle TMDB-Aufrufe der Webseite werden gecacht.

**Architecture:** Das Film-Modell bekommt eine Schwester: `NzbShow → NzbEpisode → NzbFile`. Dabei bleibt `NzbFile` eine einzige Tabelle für Filme und Folgen. Eine NZB mit `SxxEyy` im Namen wird beim Request über TMDB-TV einer Folge zugeordnet. Danach laufen Download, Remux, S3, `UserLibrary` und Re-Upload unverändert. Das Jellyfin-Plugin holt Folgen nur auf ausdrückliche Anfrage (`?include=episodes`) und schreibt sie in eine zweite Bibliothek vom Typ `tvshows`. Die Webseite cacht TMDB über `fetch(..., { next: { revalidate } })`.

**Tech Stack:** openmedia-api (Express 5, Prisma 7, PostgreSQL 16, Vitest + Supertest), openmedia-web (Next.js 16.4 ohne Cache Components, Vitest, Playwright), openmedia-jellyfin-plugin (C# net9.0, Jellyfin 10.11, xUnit 2.9).

**Spec:** `openmedia-docs/docs/superpowers/specs/2026-11-07-serien-design.md`. Dieser Plan setzt Abschnitt 9, Stufe 1 um. Die Stufen 2 bis 4 (Indexer, Staffel-Pakete, Abo und Versionswechsel) bekommen eigene Pläne.

## Global Constraints

- Repos liegen unter `/Users/jschattka/git_private/cinescope/`. Jeder Task nennt sein Repo. Pro Repo gibt es einen Branch `feature/serien-grundlage`, abgezweigt von frischem `main`.
- Bestehende Film-Daten und Film-Verhalten dürfen sich nicht ändern. Alle bestehenden Tests bleiben grün.
- API-Tests laufen gegen die Test-DB auf `localhost:5433` (`docker compose up -d db-test` in openmedia-api). `src/test/setup.ts` macht `prisma db push --force-reset`. Tests deshalb nie gegen eine andere `DATABASE_URL` laufen lassen.
- TMDB-Sprache bleibt `de-DE`. Deutsche Titel haben Vorrang, Fallback ist Englisch (`titleDe || titleEn`).
- Jellyfin-Ordner für Folgen: `{Serie} ({Jahr}) [tmdbid-N]/Season NN/{Serie} SNNENN.strm`. Der Dateiname ist stabil und enthält nie den Hash.
- Cache-Dauer (Spec 7): Listen/Trending 6 h = `21600`, Detail-Infos 24 h = `86400`, Suche 1 h = `3600` Sekunden.
- UI-Texte auf Deutsch, im Stil der bestehenden Seiten.
- Kein neues npm- oder NuGet-Paket.

## Review Focus

1. **Altes Plugin, neue API:** Ein Jellyfin mit Plugin-Version 1.2.0 ruft `/jellyfin/library` ohne Parameter auf. Es darf keine Folgen bekommen, sonst landen sie als falsche Filme in der Film-Bibliothek. → Test in Task 6.
2. **Titel mit Sonderzeichen:** Zum Beispiel "Star Trek: Discovery" oder "Love, Death & Robots". Der Ordnername muss gültig sein und darf beim nächsten Sync nicht wechseln. → Test in Task 7.
3. **Mehrfach-Folgen im Namen** ("S01E01E02", "S01E01-E02"): Diese werden als die erste Folge erkannt und nie als Film. → Test in Task 2.
4. **Aufräumen löscht keine Folgen:** `cleanupExpiredReviews` löscht nur verwaiste NzbFiles ohne Film. Eine NzbFile mit `episodeId` gilt nicht als verwaist. → Test in Task 4.
5. **Re-Upload pro Folge:** Eine eigene Archiv-Kopie von Folge 1 darf den Re-Upload von Folge 2 derselben Serie nicht verhindern. → Test in Task 5.

---

### Task 1: Datenmodell Serie/Folge (openmedia-api)

**Files:**
- Modify: `prisma/schema.prisma` (NzbFile 79-136, UploadJob 197-226, neue Modelle nach NzbMovie)
- Create: `prisma/migrations/<timestamp>_add_series/migration.sql`
- Modify: `src/test/setup.ts` (TRUNCATE-Liste)
- Test: `src/lib/series-schema.test.ts`

**Interfaces:**
- Produces: Prisma-Modelle `NzbShow { id, tmdbId Int @unique, titleDe, titleEn, description, year Int?, posterPath String?, episodes }` (`@@map("nzb_shows")`) und `NzbEpisode { id, showId, season Int, episode Int, titleDe String @default(""), airDate DateTime?, nzbFiles }` mit `@@unique([showId, season, episode])` (`@@map("nzb_episodes")`). Dazu `NzbFile.episodeId String? @map("episode_id")` mit Relation `episode NzbEpisode? onDelete: Cascade` und `@@index([episodeId])`, außerdem `UploadJob.episodeId String? @map("episode_id")` als reiner String wie `movieId`. Die Spalten folgen dem Muster von NzbMovie: snake_case `@map`, `createdAt`/`updatedAt`, uuid-IDs, `NzbEpisode.show` mit `onDelete: Cascade`.

- [ ] **Step 1: Write the failing test** `src/lib/series-schema.test.ts`

```ts
it("legt Serie → Folge → Version an und erzwingt eindeutige Folgen", async () => {
  const show = await prisma.nzbShow.create({ data: { tmdbId: 95396, titleDe: "Severance", titleEn: "Severance", year: 2022 } });
  const ep = await prisma.nzbEpisode.create({ data: { showId: show.id, season: 2, episode: 5, titleDe: "Trojan's Horse" } });
  const file = await prisma.nzbFile.create({ data: { hash: "a".repeat(64), originalFilename: "x.nzb", episodeId: ep.id } });
  expect(file.movieId).toBeNull();
  await expect(prisma.nzbEpisode.create({ data: { showId: show.id, season: 2, episode: 5 } })).rejects.toThrow();
});
it("löscht Folgen und Versionen mit der Serie (Cascade)", async () => { /* delete show → nzbEpisode.count() === 0, nzbFile.count() === 0 */ });
```

- [ ] **Step 2: Run** `npx vitest --run src/lib/series-schema.test.ts` → FAIL (`prisma.nzbShow` undefined)
- [ ] **Step 3: Implement** the schema changes. Run `npx prisma migrate dev --create-only --name add_series` against the dev DB. Append to the generated SQL: `ALTER TABLE "nzb_files" ADD CONSTRAINT "nzb_files_movie_xor_episode" CHECK ("movie_id" IS NULL OR "episode_id" IS NULL);` Add `nzb_episodes, nzb_shows` to the TRUNCATE list in `src/test/setup.ts`, directly after `nzb_movies`. Then run `npm run db:generate`.
- [ ] **Step 4: Run** `npx vitest --run src/lib/series-schema.test.ts` → PASS. Then run `npm test` → all green.
- [ ] **Step 5: Commit** `git commit -m "feat(series): add NzbShow and NzbEpisode models"`

Hinweis: Tests nutzen `db push`, nicht die Migration. Die CHECK-Bedingung wirkt deshalb nur in echten Datenbanken. Der Code setzt nie beide Spalten gleichzeitig.

---

### Task 2: SxxEyy im NZB-Namen erkennen (openmedia-api)

**Files:**
- Modify: `src/lib/nzb-parser.ts` (`ParsedNzbName`, `parseNzbName` :86, `extractTitle` :153-185)
- Test: `src/lib/nzb-parser.test.ts`

**Interfaces:**
- Produces: `ParsedNzbName` bekommt die neuen Felder `season: number | null` und `episode: number | null`. Bei Folgen ist `title` nur der Serienname.

- [ ] **Step 1: Write the failing tests** (table test)

| Eingabe | title | year | season | episode | resolution |
|---|---|---|---|---|---|
| `Severance.S02E05.German.AC3.DL.1080p.Web.x265-FuN.nzb` | `Severance` | null | 2 | 5 | `1080p` |
| `Severance.2022.S02E05.German.DL.Atmos.2160p.DV.HDR.ATVP.WEB.H265-ZeroTwo.nzb` | `Severance` | 2022 | 2 | 5 | `2160p` |
| `Show.Name.S01E01E02.German.720p.WEB.x264-GRP.nzb` | `Show Name` | null | 1 | 1 | `720p` |
| `Show.Name.S01E01-E02.German.720p.WEB.x264-GRP.nzb` | `Show Name` | null | 1 | 1 | `720p` |
| `Dune.2021.German.DL.1080p.BluRay.x264-GRP.nzb` | `Dune` | 2021 | null | null | `1080p` |

- [ ] **Step 2: Run** `npx vitest --run src/lib/nzb-parser.test.ts` → FAIL
- [ ] **Step 3: Implement** Match `/(?:^|[ ._-])S(\d{1,2})[ ._-]?E(\d{1,3})/i` before the title is extracted. The title is everything before the match, with the existing cut rules (year and keywords). The first match counts, so later `E02` parts are ignored. Without a match, `season` and `episode` are `null` and the film behaviour stays unchanged.
- [ ] **Step 4: Run** the parser tests and then `npm test` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): parse season/episode from NZB names"`

---

### Task 3: TMDB-TV-Lookup im API (openmedia-api)

**Files:**
- Modify: `src/lib/tmdb.ts`
- Test: `src/lib/tmdb-tv.test.ts`

**Interfaces:**
- Produces:
  - `interface TmdbShowResult { tmdbId: number; titleDe: string; titleEn: string; description: string; year: number | null; posterPath: string | null }`
  - `interface TmdbEpisodeResult { titleDe: string; airDate: string | null }`
  - `type TmdbShowLookupResult = { status: "found"; show: TmdbShowResult; episode: TmdbEpisodeResult | null } | { status: "not_found" } | { status: "error"; reason: string } | { status: "disabled"; reason: string }`
  - `searchTmdbShowEpisode(title: string, year: number | null, season: number, episode: number): Promise<TmdbShowLookupResult>`

- [ ] **Step 1: Write the failing tests.** Stub global `fetch` with `vi.stubGlobal`, following the pattern of the existing tmdb tests in this repo:
  - Found: `/search/tv` returns `[{ id: 95396 }]`, `/tv/95396` returns de and en names, and `/tv/95396/season/2/episode/5` returns `{ name: "Trojan's Horse", air_date: "2025-02-13" }`. Expect `status: "found"`, `show.tmdbId === 95396`, `show.year === 2022` (from `first_air_date: "2022-02-18"`), `episode.titleDe === "Trojan's Horse"`.
  - Episode call returns 404 → `status: "found"` with `episode: null`.
  - `/search/tv` returns no results → `not_found`. 429 → `error`. Empty `TMDB_API_KEY` → `disabled`.
  - `year` is passed as `first_air_date_year`.
- [ ] **Step 2: Run** `npx vitest --run src/lib/tmdb-tv.test.ts` → FAIL
- [ ] **Step 3: Implement** with the same timeout, key handling and de/en fallback logic as `searchTmdbMovie`/`fetchTmdbMovieDetails` (lines 35-88). Take `results[0]`.
- [ ] **Step 4: Run** → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): TMDB TV episode lookup"`

---

### Task 4: Folge beim NZB-Request und im Reconciler zuordnen (openmedia-api)

**Files:**
- Create: `src/lib/episode-resolver.ts`
- Modify: `src/routes/downloads.ts` (POST `/request` 592-715; POST `/jobs` 271-278; `fullJob`-include)
- Modify: `src/lib/job-reconciler.ts` (`retryTmdbForPendingReviews` 118, `cleanupExpiredReviews` 370)
- Test: `src/routes/downloads-request-episode.test.ts`, `src/lib/job-reconciler-review.test.ts`

**Interfaces:**
- Consumes: `searchTmdbShowEpisode` (Task 3), `ParsedNzbName.season/episode` (Task 2)
- Produces:
  - `resolveEpisodeFromTmdb(parsed: ParsedNzbName, fallbackTitle: string): Promise<{ source: "tmdb-found"; show: TmdbShowResult; episode: TmdbEpisodeResult | null } | { source: "needs-review"; reason: "not-found" | "error"; errorDetail?: string } | { source: "tmdb-disabled"; reason: string }>`. Same contract as `resolveMovieFromTmdb` (downloads.ts:43-89).
  - `upsertEpisode(tx: Prisma.TransactionClient, show: TmdbShowResult, season: number, episode: number, tmdbEpisode: TmdbEpisodeResult | null): Promise<string>` returns the `episodeId`. It reuses or creates by `tmdbId` and `(showId, season, episode)` and catches P2002 the same way as the movie path.
  - Response of `/downloads/request`: `job.nzbFile.episode` with `{ id, season, episode, titleDe, show: { tmdbId, titleDe, titleEn, year, posterPath } }`. This field is set next to `movie`.

- [ ] **Step 1: Write the failing tests** in `downloads-request-episode.test.ts`. Mock `../lib/tmdb.js`, `../lib/s3.js` and `../lib/nzb-service.js` like `downloads-request.test.ts:28-41`.
  - `filename: "Severance.S02E05.German.DL.1080p.WEB.x264-FuN.nzb"` → 201, `needsReview: false`. `nzbFile.episodeId` is set, `movieId` is null, and exactly 1 `NzbShow` (tmdbId 95396) and 1 `NzbEpisode` (2/5) exist. `searchTmdbMovie` was never called.
  - Same show, other episode (S02E06) → no second `NzbShow`.
  - `searchTmdbShowEpisode` returns `not_found` → `needsReview: true`, job `needs_review`, `episodeId` null.
  - A film name still uses the movie path, as before.
  - Then `POST /downloads/jobs` with the episode NzbFile → not 409.
- [ ] **Step 2: Write the failing tests** in `job-reconciler-review.test.ts`:
  - A needs_review job with an episode filename whose `tmdbRetryAfter` has passed. TMDB now finds the show, so after `reconcileStaleJobs()` the file has an `episodeId` and the job is `queued`.
  - `cleanupExpiredReviews` does not delete a NzbFile with `episodeId` set, even when `movieId` is null and there is no `s3Key` (Review Focus 4).
- [ ] **Step 3: Run** both files → FAIL
- [ ] **Step 4: Implement**
  - In `/request`: if `parsed.season !== null && parsed.episode !== null`, take the episode path (`resolveEpisodeFromTmdb` → in the same tx `upsertEpisode` → create `nzbFile` with `episodeId`). Otherwise keep the existing movie path unchanged.
  - Change the 409 check in `/jobs` to `!nzbFile.movieId && !nzbFile.episodeId`.
  - In the reconciler, add `assignEpisodeToNzbFile` next to `assignMovieToNzbFile`, with the same sibling-flip logic.
  - Extend the orphan condition in `cleanupExpiredReviews` with `episodeId === null`.
- [ ] **Step 5: Run** both test files, then `npm test` → PASS
- [ ] **Step 6: Commit** `git commit -m "feat(series): resolve episodes on NZB request and TMDB retry"`

Bekannte Grenze von Stufe 1: `POST /jobs/:id/assign-movie` kann nur Filme zuweisen. Eine Folge im Zustand needs_review wartet auf den automatischen TMDB-Retry oder läuft ab. Die manuelle Zuordnung kommt mit Stufe 2.

---

### Task 5: Re-Upload ins Archiv für Folgen (openmedia-api)

**Files:**
- Modify: `src/routes/downloads.ts` (Completed-Callback 1014-1024 und `uploadJob.create`)
- Modify: `src/routes/uploads.ts:341-356` (eigene NzbFile bekommt `episodeId`)
- Test: `src/routes/downloads.test.ts` (neuer `describe`), `src/routes/uploads.test.ts`

**Interfaces:**
- Consumes: `NzbFile.episodeId`, `UploadJob.episodeId` (Task 1)

- [ ] **Step 1: Write the failing tests**
  - Completed callback for an episode file → exactly 1 `UploadJob` with `episodeId` set to the episode and `movieId: null`.
  - An `own` copy of episode 1 already exists → completing episode 2 of the same show still creates an `UploadJob` (Review Focus 5).
  - An `own` copy of the same episode exists → no new `UploadJob`, the same as for films.
  - In `uploads.test.ts`: when an episode UploadJob completes, the `own` NzbFile has `episodeId` set to the original episode.
- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement** The skip check filters by `movieId` for films and by `episodeId` for episodes. `uploadJob.create` carries `episodeId`. `uploads.ts` copies `episodeId` from the UploadJob into the own NzbFile.
- [ ] **Step 4: Run**, then `npm test` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): re-upload episodes to own archive"`

---

### Task 6: Lese-Endpunkte für Serien (openmedia-api)

**Files:**
- Modify: `src/routes/nzb.ts` (neuer Handler nach `/movies/by-tmdb/:tmdbId` 179-202)
- Modify: `src/routes/library.ts` (GET `/` 9-32, Select)
- Modify: `src/routes/jellyfin.ts` (GET `/library` 704-751)
- Test: `src/routes/nzb.test.ts`, `src/routes/library.test.ts`, `src/routes/jellyfin.test.ts`

**Interfaces:**
- Produces:
  - `GET /nzb/shows/by-tmdb/:tmdbId` → `{ show: { id, tmdbId, titleDe, titleEn, year, posterPath, episodes: [{ id, season, episode, titleDe, airDate, nzbFiles: [<nzbFileSelect, fileSize als string>] }] } }`. Episodes are sorted by season and episode. 404 `"Serie nicht gefunden."`, 400 for NaN.
  - `GET /library` items: `nzbFile.episode: { id, season, episode, titleDe, show: { tmdbId, titleDe, titleEn, year, posterPath } } | null` next to `movie`.
  - `GET /jellyfin/library`: every film item gets `kind: "movie"`. Only with `?include=episodes` are episode items also included: `{ kind: "episode", hash, tmdbId: <Serien-TMDB-ID>, title: show.titleDe || show.titleEn, year: show.year, season, episode, fileSize, duration, resolution }`.

- [ ] **Step 1: Write the failing tests**
  - by-tmdb returns the episodes sorted, with versions and with `fileSize` as a string. Unknown ID → 404.
  - `/library` contains `nzbFile.episode.show.tmdbId === 95396` for an episode entry.
  - Jellyfin: without parameter, a user with 1 film + 1 episode gets exactly 1 item with `kind: "movie"` (Review Focus 1). With `?include=episodes` there are 2 items, and the episode item has `season: 2, episode: 5, tmdbId: 95396`.
- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement** Reuse the existing `nzbFileSelect` and file serialization from `nzb.ts`. `/library/version` stays unchanged, because its hash fingerprint already contains the episodes.
- [ ] **Step 4: Run**, then `npm test` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): expose shows and episodes via nzb, library and jellyfin APIs"`

---

### Task 7: Folgen-Sync in der Serien-Ordnerstruktur (openmedia-jellyfin-plugin)

**Files:**
- Modify: `Jellyfin.Plugin.OpenMedia/Api/OpenMediaApiClient.cs` (`LibraryItem` :16, `GetLibraryAsync` :58)
- Create: `Jellyfin.Plugin.OpenMedia/Tasks/EpisodeStrmSyncEngine.cs`
- Test: `Jellyfin.Plugin.OpenMedia.Tests/EpisodeStrmSyncEngineTests.cs`

**Interfaces:**
- Consumes: `/jellyfin/library?include=episodes` (Task 6)
- Produces:
  - `LibraryItem` gets new trailing parameters with defaults: `string? Kind = null, int? Season = null, int? Episode = null`. `GetLibraryAsync` calls `/jellyfin/library?include=episodes`.
  - `public static class EpisodeStrmSyncEngine` with `public static Task<StrmSyncResult> SyncAsync(string tvDirectory, IEnumerable<LibraryItem> episodes, string apiBaseUrl, string apiToken, PrecacheStateStore? precacheState, ILogger? logger, CancellationToken ct)`
  - `internal static string BuildEpisodeRelativePath(string? showTitle, int? year, long tmdbId, int season, int episode)` → `Path.Combine(StrmSyncEngine.BuildFolderName(showTitle, year, tmdbId), $"Season {season:00}", $"{name} S{season:00}E{episode:00}.strm")`. Here `name` is `StrmSyncEngine.SanitizeTitle(showTitle)`, and empty means `"Untitled"`.

- [ ] **Step 1: Write the failing tests.** Use the fixture style of `StrmSyncEngineTests.cs:14-54`:
  - `WritesEpisodeStrmInSeasonFolder`: Severance (2022, 95396) S02E05 → `Severance (2022) [tmdbid-95396]/Season 02/Severance S02E05.strm` with content `{BaseUrl}/jellyfin/stream/{hash}?token={Token}`.
  - `VersionChangeKeepsFileNameAndRewritesContent`: same episode, new hash → same path, `Updated == 1`, the old file no longer exists elsewhere.
  - `RemovesEpisodeAndEmptySeasonAndShowFolders`: episode leaves the library → `.strm`, `Season 02` and the show folder are gone.
  - `LeavesForeignFilesUntouched`: a foreign `.strm` with different content and a `notes.txt` in our season folder stay. Foreign folders without `[tmdbid-N]` stay too.
  - `SanitizesSpecialCharactersStably`: "Star Trek: Discovery" gives the same path in two syncs, and the second sync reports `Unchanged == 1` (Review Focus 2).
  - `SkipsItemsWithoutSeasonOrEpisodeOrTmdbId`, `RejectsInvalidHashes`.
  - `SkipsCachedHash`: same pattern as the existing cached tests (:468).
- [ ] **Step 2: Run** `dotnet test Jellyfin.Plugin.OpenMedia.Tests/Jellyfin.Plugin.OpenMedia.Tests.csproj --filter EpisodeStrmSyncEngineTests` → FAIL (compile error)
- [ ] **Step 3: Implement** with the same path guards (`rootWithSep`) as `StrmSyncEngine`. Ownership rule for cleanup: below a root folder matching ` [tmdbid-N]`, the engine only deletes `.strm` files whose content starts with `{baseUrl}/jellyfin/stream/` and that are no longer expected. Then it deletes empty `Season NN` folders and empty show folders.
- [ ] **Step 4: Run** the filter, then the whole test project → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): episode strm sync in tvshows layout"`

---

### Task 8: Zweite Bibliothek, Sync-Verdrahtung, Pre-Cache und Media-Source (openmedia-jellyfin-plugin)

**Files:**
- Modify: `Configuration/PluginConfiguration.cs`, `Configuration/configPage.html` (:55-85, :117-119, :129-131)
- Modify: `LibraryAutoCreateService.cs` (:52-143)
- Modify: `Tasks/LibrarySyncTask.cs` (:59-67), `Tasks/LibrarySyncRunner.cs` (:53-122), `Tasks/StrmSyncEngine.cs` (`StrmSyncResult` :17)
- Modify: `Tasks/PrecacheWorker.cs` (:162-173, :482-490)
- Modify: `MediaSources/OpenMediaMediaSourceProvider.cs` (`ExtractHash` :258-278)
- Test: `OpenMediaMediaSourceProviderTests.cs`, `PrecacheWorkerTests.cs`, `StrmSyncEngineTests.cs`

**Interfaces:**
- Consumes: `EpisodeStrmSyncEngine` (Task 7)
- Produces:
  - Config: `TvStrmDirectory` (default `""`, effective `DataPath/openmedia-strm-tv`) and `TvLibraryName` (default `"openmedia Serien"`).
  - `LibrarySyncTask.GetEffectiveTvStrmDirectory(string? configured, IApplicationPaths paths)`.
  - `StrmSyncResult.Combine(StrmSyncResult a, StrmSyncResult b)` (sums all fields).
  - `internal static (string Folder, string FileStem) StrmSyncEngine.MediaLocationFor(LibraryItem item, string movieDir, string tvDir)`. Films: `(movieDir/BuildFolderName(..), BuildMediaFileStem(..))`. Episodes: `(tvDir/<Show-Ordner>/Season NN, "{Show} SNNENN")`.

- [ ] **Step 1: Write the failing tests**
  - `ExtractHash` for an existing `Severance S02E05.strm` whose content is `https://api.example.com/jellyfin/stream/{64hex}?token=x` → returns the hash. The existing tests for files without stream content stay null.
  - `MediaLocationFor` for a film and for an episode → exact paths as defined above.
  - `PrecacheWorker`: an episode item is downloaded into the season folder as `Severance S02E05.mp4` (same fixture as the existing PrecacheWorker tests).
  - `StrmSyncResult.Combine` sums all seven fields.
- [ ] **Step 2: Run** `dotnet test …` → FAIL
- [ ] **Step 3: Implement**
  - `ExtractHash`: if the file name is not a hash, read the `.strm` file (at most 4 KB) and take `/jellyfin/stream/([0-9a-f]{64})`.
  - `LibraryAutoCreateService`: extract the creation into `EnsureVirtualFolderAsync(string name, string dir, CollectionTypeOptions type, CancellationToken ct)` with the existing name and path checks. Call it for `movies` (unchanged) and `tvshows` (`TvLibraryName`, TV dir). Both use the same `AutoCreateLibrary` flag.
  - Runner: split the items by `Kind` (`null`/`"movie"` → `StrmSyncEngine`, `"episode"` → `EpisodeStrmSyncEngine`) and return `Combine`. The scan logic stays the same.
  - PrecacheWorker: use `MediaLocationFor` and remove the private copy of `GetEffectiveStrmDirectory`.
  - configPage: add both fields the same way as `StrmDirectory`/`LibraryName`.
  - Raise the version in the csproj to 1.3.0.0.
- [ ] **Step 4: Run** `dotnet build Jellyfin.Plugin.OpenMedia.sln` and `dotnet test Jellyfin.Plugin.OpenMedia.Tests/Jellyfin.Plugin.OpenMedia.Tests.csproj` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): tvshows library, sync wiring, episode precache and media source"`

---

### Task 9: TMDB-Cache (openmedia-web)

**Files:**
- Modify: `src/lib/tmdb.ts` (`tmdbFetch` :91-121 und alle Aufrufer darin)
- Test: `src/lib/tmdb.test.ts`

**Interfaces:**
- Produces: `export const TMDB_REVALIDATE = { lists: 21600, details: 86400, search: 3600 } as const;` and `tmdbFetch<T>(endpoint: string, params: Record<string, string> | undefined, revalidate: number): Promise<T>`, which calls `fetch(url, { next: { revalidate } })`.

- [ ] **Step 1: Write the failing test.** Use the fetch stub from `src/lib/auth.test.ts:4-14`, with `vi.stubEnv("TMDB_API_KEY", "k")`:
  - `getTrendingMovies()` → `fetch` called with `(expect.stringContaining("trending/movie/week"), { next: { revalidate: 21600 } })`.
  - `discoverMoviesByGenre(28)` → 21600. `getGenres()` → 86400. `getMovieDetails(1)` → 86400. `searchMovies("dune")` → 3600.
- [ ] **Step 2: Run** `npx vitest --run src/lib/tmdb.test.ts` → FAIL
- [ ] **Step 3: Implement.** This is the "previous model" from `node_modules/next/dist/docs/01-app/02-guides/caching-without-cache-components.md`. Do not enable `cacheComponents`.
- [ ] **Step 4: Run** `npm test`, `npx tsc --noEmit` and `npm run build` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(tmdb): cache TMDB responses via fetch revalidate"`

---

### Task 10: TMDB-TV-Client (openmedia-web)

**Files:**
- Modify: `src/lib/tmdb.ts`
- Test: `src/lib/tmdb.test.ts`

**Interfaces:**
- Produces:
  - `interface Show { id: number; name: string; original_name: string; overview: string; poster_path: string | null; backdrop_path: string | null; vote_average: number; first_air_date: string; genre_ids: number[] }`
  - `interface SeasonSummary { season_number: number; name: string; episode_count: number; air_date: string | null; poster_path: string | null }`
  - `interface ShowDetail extends Show { number_of_seasons: number; seasons: SeasonSummary[]; genres: Genre[]; tagline: string | null; credits: { cast: CastMember[] }; videos: { results: Video[] }; similar: { results: Show[] } }`
  - `interface EpisodeSummary { episode_number: number; name: string; overview: string; air_date: string | null; still_path: string | null; runtime: number | null }`
  - `interface SeasonDetail { season_number: number; name: string; episodes: EpisodeSummary[] }`
  - Functions:
    - `getTrendingShows(): Promise<Show[]>` (`trending/tv/week`, lists)
    - `searchShows(query: string, page?: number): Promise<{ page; results: Show[]; total_pages; total_results }>` (`search/tv`, search, blank → empty without fetch)
    - `getShowDetails(id: number): Promise<ShowDetail>` (`tv/{id}` with `append_to_response: "credits,videos,similar"`, details)
    - `getSeasonDetails(id: number, season: number): Promise<SeasonDetail>` (`tv/{id}/season/{season}`, details)

- [ ] **Step 1: Write the failing tests:** endpoint and revalidate for every function. `searchShows("  ")` does not call fetch.
- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement** with `tmdbFetch` from Task 9.
- [ ] **Step 4: Run** → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): TMDB TV client"`

---

### Task 11: Folgen in Bibliothek und Downloads (openmedia-web)

**Files:**
- Modify: `src/lib/backend.ts` (`LibraryItem` :182-198, neue Funktion nach `getNzbMovieByTmdb` :160)
- Modify: `src/app/bibliothek/page.tsx` (`LibraryCard` :26-184)
- Modify: `src/app/downloads/page.tsx:96`
- Test: `src/app/bibliothek/library-card.test.tsx`

**Interfaces:**
- Consumes: `GET /nzb/shows/by-tmdb/:tmdbId`, `GET /library` with `nzbFile.episode` (Task 6)
- Produces:
  - `getNzbShowByTmdb(tmdbId: number, token: string)` → `{ show: NzbShowInfo }`.
  - `NzbShowInfo = { id; tmdbId; titleDe; titleEn; year; posterPath; episodes: { id; season; episode; titleDe; airDate; nzbFiles: NzbFileInfo[] }[] }`.
  - `LibraryItem.nzbFile.movie` becomes nullable, and `nzbFile.episode: { id; season; episode; titleDe; show: { tmdbId; titleDe; titleEn; year; posterPath } } | null` is added.
  - Helper `libraryItemLink(item: LibraryItem): string` returns `/movie/{tmdbId}` for films and `/serie/{show.tmdbId}` for episodes.

- [ ] **Step 1: Write the failing test.** Export `LibraryCard` for the test:
  - An episode item shows "Severance", the line "S02E05 · Trojan's Horse" and the show poster, and links to `/serie/95396`.
  - A film item shows the same as before.
- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement.** On the downloads page, the link uses the same rule (`/serie/…` when `nzbFile.episode` is set). The counter text "N Film/Filme" becomes "N Titel".
- [ ] **Step 4: Run** `npm test`, `npx tsc --noEmit`, `npm run build` → PASS
- [ ] **Step 5: Commit** `git commit -m "feat(series): show episodes in library and downloads"`

---

### Task 12: Serien-Seiten, Navigation und Suche (openmedia-web)

**Files:**
- Create: `src/app/serien/page.tsx`, `src/app/serien/loading.tsx`
- Create: `src/app/serie/[id]/page.tsx`, `src/app/serie/[id]/loading.tsx`
- Create: `src/components/show/show-card.tsx`, `src/components/show/show-grid.tsx`, `src/components/show/season-list.tsx`
- Modify: `src/components/movie/hero-banner.tsx` (neue Prop `href`)
- Modify: `src/components/layout/header.tsx:17-24`
- Modify: `src/app/search/page.tsx`
- Modify: `e2e/tmdb-mock-server.ts` (Handler ab :147)
- Test: `src/components/show/season-list.test.tsx`, `src/components/show/show-card.test.tsx`

**Interfaces:**
- Consumes: Task 10, plus `getNzbShowByTmdb` from Task 11 (client-side in `SeasonList`)
- Produces:
  - `ShowCard({ show }: { show: Show })` → link `/serie/${show.id}`. Title is `show.name`, year is `first_air_date.slice(0,4)`. No watchlist button, because the watchlist only supports films.
  - `ShowGrid({ shows, heading = "Trending Serien" }: { shows: Show[]; heading?: string })`
  - `HeroBanner({ movie, href }: { movie: Movie; href?: string })`, default `/movie/${movie.id}`.
  - `SeasonList({ showId, seasons }: { showId: number; seasons: { summary: SeasonSummary; episodes: EpisodeSummary[] }[] })` is a client component. Per episode it shows "SNNENN · Titel · Ausstrahlung". When versions exist, it shows a badge per version `qualityTier || resolution` ("1080p") and a "Abspielen"/"Laden" action like the library page.

- [ ] **Step 1: Write the failing tests**
  - `show-card.test.tsx`: link `href="/serie/95396"`, text "Severance", "2022".
  - `season-list.test.tsx` (mock `@/lib/backend`): an episode without versions has no badge. An episode with a 1080p version shows the badge "1080p". Seasons and episodes appear in ascending order. Season 0 is labelled "Specials".
- [ ] **Step 2: Run** `npx vitest --run src/components/show` → FAIL
- [ ] **Step 3: Implement**
  - `/serien`: `getTrendingShows()`. The first show with a backdrop goes into the `HeroBanner`, mapped to the `Movie` fields it uses, with `href=/serie/{id}`. The rest goes into a `ShowGrid`. The error display follows `app/page.tsx`.
  - `/serie/[id]`: `getShowDetails` plus `getSeasonDetails` for all `seasons` in parallel (`Promise.all`). Reuse the hero, cast and trailer layout of the movie page, with `SeasonList` and `ShowGrid` "Ähnliche Serien" below. Metadata title `"{name} — CineScope"`. Wrap content in `<div>`, not a second `<main>`.
  - Nav: add `{ href: "/serien", label: "Serien" }` after "Home".
  - Search: add a second section "Serien" with `searchShows(query)`. A TMDB error only hides this section (try/catch). The film results stay unchanged.
  - E2E mock: `GET /search/tv` returns an empty result set.
- [ ] **Step 4: Run** `npm test`, `npx tsc --noEmit`, `npm run build` → PASS. Check visually with `npm run dev`: `/serien`, `/serie/95396` and `/search?q=severance` look like the film pages.
- [ ] **Step 5: Commit** `git commit -m "feat(series): series pages, navigation and search"`

---

### Task 13: Durchstich von Hand prüfen (alle Repos)

- [ ] **Step 1:** Start the API, web and the Jellyfin plugin locally (README dev loop in each repo).
- [ ] **Step 2:** Send a German 1080p episode NZB (for example `Severance.S02E05.German…`) through the browser extension or `POST /downloads/request`.
- [ ] **Step 3:** Once the job is `completed`, check:
  - `/bibliothek` shows "Severance · S02E05".
  - `/serie/95396` shows the "1080p" badge for S02E05.
  - Jellyfin has the library "openmedia Serien" with `Severance (2022) [tmdbid-95396]/Season 02/Severance S02E05.strm`, and the episode plays.
  - An `UploadJob` with `episodeId` exists.
  - The film library is unchanged.
- [ ] **Step 4:** Reload the page several times. The TMDB request count must not grow with each reload (Next server log, or `fetch` log in dev).
- [ ] **Step 5:** Write the result as a note in the PR descriptions of the four repos.
